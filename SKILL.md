---
name: claude-context-audit
description: >
  Audit and optimize Claude Code context window usage — measures MEMORY.md,
  skills, plugins, hooks overhead and recommends cuts.
license: MIT
triggers:
  - context audit
  - optimize context
  - context filling up
  - compacting too often
  - token usage high
metadata:
  author: jeremyknows
  version: "2.0.0"
---

# Claude Code Context Audit

## Step 1: Measure Overhead

Run all checks in parallel, then report a summary table with **file, size (bytes), estimated tokens (bytes ÷ 4), and load frequency** (every session, every prompt, on-demand, or on compaction).

### Global and project-level instructions

```bash
# Global CLAUDE.md
wc -c ~/.claude/CLAUDE.md 2>/dev/null || echo "No global CLAUDE.md"

# Project-level CLAUDE.md files (check working directory tree)
find . -maxdepth 3 -name "CLAUDE.md" 2>/dev/null | while read -r f; do
  printf '%s bytes: %s\n' "$(wc -c < "$f")" "$f"
done

# All MEMORY.md files (loaded twice per session — once as auto-memory, once in claudeMd rules)
for f in ~/.claude/projects/*/memory/MEMORY.md; do
  [ -f "$f" ] && printf '%s bytes, %s lines: %s\n' "$(wc -c < "$f")" "$(wc -l < "$f")" "$f"
done 2>/dev/null
```

### Skills — count and content size

```bash
printf 'Registered skills: %s\n' "$(ls -d ~/.claude/skills/*/ 2>/dev/null | wc -l)"

# Measure actual content size per skill (not just inode size)
for d in ~/.claude/skills/*/; do
  [ -d "$d" ] || continue
  name=$(basename "$d")
  size=$(find "$d" -type f -exec cat {} + 2>/dev/null | wc -c)
  printf '%8s bytes  %s\n' "$size" "$name"
done | sort -rn
```

Flag any skill over 1 MB — it likely contains data fixtures or assets, not just instructions.

### Plugins and hooks config

```bash
jq '.enabledPlugins // {}' ~/.claude/settings.json 2>/dev/null || echo "No plugins config"
jq '.hooks // {}' ~/.claude/settings.json 2>/dev/null || echo "No hooks config"
```

### Session log disk usage

```bash
# Top 10 largest log files
du -sh ~/.claude/projects/*/*.jsonl 2>/dev/null | sort -rh | head -10

# Total reclaimable space (logs older than 7 days)
old_count=$(find ~/.claude/projects/ -name "*.jsonl" -mtime +7 2>/dev/null | wc -l)
old_size=$(find ~/.claude/projects/ -name "*.jsonl" -mtime +7 -exec du -sh {} + 2>/dev/null | awk '{s+=$1} END {printf "%.0f", s}')
printf '%s files older than 7 days, ~%sMB reclaimable\n' "$old_count" "$old_size"
```

### Summary table format

Present results as:

| Source | Size (bytes) | Est. Tokens | Load Frequency |
|--------|-------------|-------------|----------------|
| `~/.claude/CLAUDE.md` | X | X÷4 | Every session |
| `MEMORY.md` (auto-memory) | X | X÷4 | Every session |
| `MEMORY.md` (claudeMd duplicate) | X | X÷4 | Every session |
| Skill registry (N entries × ~200 chars) | N×200 | N×50 | Every session |
| SessionStart hook output | X | X÷4 | Startup + every compaction |
| UserPromptSubmit hook output | X | X÷4 | Every user prompt |

## Step 2: Audit Skill Usage

Find the 5 most recent session logs (excluding subagent logs), then check which skills were actually **invoked** (not just mentioned in the registry listing):

```bash
LOGS=$(find ~/.claude/projects/ -maxdepth 2 -name "*.jsonl" -not -path "*/subagents/*" 2>/dev/null | xargs ls -t 2>/dev/null | head -5)
for d in ~/.claude/skills/*/; do
  [ -d "$d" ] || continue
  name=$(basename "$d")
  count=0
  for log in $LOGS; do
    # Match actual skill invocations, not registry listings
    # Patterns: "skill":"name", Skill(name), skill: "name"
    grep -qE "\"skill\"[[:space:]]*:[[:space:]]*\"$name\"|Skill\($name\)|skill:[[:space:]]*\"$name\"" "$log" 2>/dev/null && count=$((count + 1))
  done
  printf '%d %s\n' "$count" "$name"
done | sort -rn
```

**IMPORTANT**: A bare `grep "$name"` will produce false positives because every skill name appears in the system prompt registry listing in every session log. The pattern above matches actual invocations only.

Categorize as:
- **ACTIVE**: Found in at least 1 recent log
- **UNUSED**: No invocations found

**Check with user before archiving** — a skill with 0 recent hits may still be needed for rare scenarios. Offer `disable-model-invocation: true` as a lighter alternative to archiving (see Step 4).

## Step 3: Audit Plugins & Hooks

### SessionStart hooks (re-fire on every compaction)

Measure the **actual output** injected into context, not just the script file size:

```bash
for d in ~/.claude/plugins/cache/*/*/*/hooks/; do
  [ -d "$d" ] || continue
  if [ -f "${d}session-start.sh" ]; then
    file_size=$(wc -c < "${d}session-start.sh")
    # Measure actual output — this is what gets injected into context
    output_size=$(bash "${d}session-start.sh" 2>/dev/null | wc -c)
    printf 'Script: %s bytes, Output: %s bytes — %s\n' "$file_size" "$output_size" "${d}session-start.sh"
  fi
done 2>/dev/null
```

The output size is what matters for context budget — scripts often read and embed other files, making output 2-3× larger than the script itself.

### Hook matchers (compact in matcher = re-fires on compaction)

```bash
for f in ~/.claude/plugins/cache/*/*/*/hooks/hooks.json; do
  [ -f "$f" ] || continue
  matcher=$(jq -r '.hooks | to_entries[] | "\(.key): \(.value[].matcher // "none")"' "$f" 2>/dev/null)
  printf '%s\n  %s\n' "$f" "$matcher"
done 2>/dev/null
```

### UserPromptSubmit hooks (fire on every user message)

```bash
jq -r '.hooks.UserPromptSubmit[]?.hooks[]?.command // empty' ~/.claude/settings.json 2>/dev/null | while read -r cmd; do
  expanded="${cmd/#\~/$HOME}"
  if [ -f "$expanded" ]; then
    output_size=$(bash "$expanded" 2>/dev/null | wc -c)
    printf 'Output: %s bytes (~%s tokens/prompt) — %s\n' "$output_size" "$((output_size / 4))" "$expanded"
  fi
done
```

Flag any hook output over 500 bytes as a candidate for slimming.

## Step 4: Optimize

Present findings, then apply only fixes the user approves:

### MEMORY.md — Restructure as lean index + topic files

Only the first 200 lines of `MEMORY.md` are auto-loaded, and they're loaded **twice** (known Claude Code behavior). Keep it under 100 lines. Move details to topic files that are read on-demand only:

```
memory/MEMORY.md     ← Index only, auto-loaded (first 200 lines, loaded twice)
memory/topic-a.md    ← On-demand, NOT auto-loaded
memory/archive.md    ← Historical notes
```

### Skills — three tiers of action

**Archive** unused skills (reversible):
```bash
mkdir -p ~/.claude/skills-archive
for skill in <list-unused-skill-names-here>; do
  [ -d ~/.claude/skills/"$skill" ] && mv ~/.claude/skills/"$skill" ~/.claude/skills-archive/
done
# Restore: mv ~/.claude/skills-archive/<name> ~/.claude/skills/
```

**Suppress** rarely-needed but valuable skills — add to frontmatter so they stay installed but don't appear in the model's system prompt unless explicitly invoked:
```yaml
disable-model-invocation: true
```
This removes ~200 chars/skill from the prompt without losing the skill.

**Keep** actively used skills as-is.

### Plugins — Disable unused

In `~/.claude/settings.json` → `enabledPlugins`, set unused plugins to `false`. Re-enable by setting back to `true`.

### Hooks — Slim verbose output

Rewrite hooks that output >500 bytes to use minimal text with the same semantic meaning. Example — a 1,380-byte claudeception hook can be reduced to ~160 bytes:

```bash
# Before (1,380 bytes output):
# Box art, headers, numbered protocol, repeated emphasis...

# After (160 bytes output):
cat << 'EOF'
After completing this request: if it required non-obvious debugging or produced reusable knowledge, use Skill(claudeception). Skip for routine tasks.
EOF
```

### Session logs — Safe cleanup

Delete logs older than 7 days. The current session is always <7 days old, so this is safe:
```bash
find ~/.claude/projects/ -name "*.jsonl" -mtime +7 -delete 2>/dev/null
# Clean up empty directories left behind
find ~/.claude/projects/ -type d -name "subagents" -empty -delete 2>/dev/null
find ~/.claude/projects/ -mindepth 2 -maxdepth 2 -type d -empty -delete 2>/dev/null
```

### Optional env vars

Check your Claude Code version supports these:
- `SLASH_COMMAND_TOOL_CHAR_BUDGET` — caps skill definitions in prompt
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` — override auto-compact threshold (default ~95%)

## Step 5: Report

Re-run Step 1 measurements and present a before/after comparison:

| Category | Before | After | Savings |
|----------|--------|-------|---------|
| Skills in registry | N entries (~X tokens) | M entries (~Y tokens) | ~Z tokens/session |
| Hook output per prompt | X bytes (~Y tokens) | A bytes (~B tokens) | ~C tokens/prompt |
| Session logs on disk | X GB | Y MB | ~Z GB freed |
| **Estimated total savings** | | | **~N tokens/session** |

For per-prompt savings, multiply by estimated prompts per session (~50) to show cumulative impact.
