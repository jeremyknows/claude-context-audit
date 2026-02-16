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
  version: "1.0.0"
---

# Claude Code Context Audit

## Step 1: Measure Overhead

Run all checks, then report a summary table.

```bash
wc -c ~/.claude/CLAUDE.md 2>/dev/null || echo "No CLAUDE.md"

for f in ~/.claude/projects/*/memory/MEMORY.md; do
  [ -f "$f" ] && printf '%s bytes, %s lines: %s\n' "$(wc -c < "$f")" "$(wc -l < "$f")" "$f"
done 2>/dev/null

printf 'Active skills: %s\n' "$(ls -d ~/.claude/skills/*/ 2>/dev/null | wc -l)"

jq '.enabledPlugins // {}' ~/.claude/settings.json 2>/dev/null || echo "No plugins config"
jq '.hooks // {}' ~/.claude/settings.json 2>/dev/null || echo "No hooks config"

du -sh ~/.claude/projects/*/*.jsonl 2>/dev/null | sort -rh | head -10
```

## Step 2: Audit Skill Usage

Find the 5 most recent session logs, then check which skills appear:

```bash
LOGS=$(ls -t ~/.claude/projects/*/*.jsonl 2>/dev/null | head -5)
for d in ~/.claude/skills/*/; do
  [ -d "$d" ] || continue
  name=$(basename "$d")
  count=0
  for log in $LOGS; do
    grep -q "$name" "$log" 2>/dev/null && count=$((count + 1))
  done
  printf '%d %s\n' "$count" "$name"
done | sort -rn
```

Categorize as ACTIVE (found) or UNUSED (not found). **Check with user before archiving** — a skill with 0 recent hits may still be needed for rare scenarios.

## Step 3: Audit Plugins & Hooks

Check each enabled plugin for SessionStart hooks (these re-fire on every compaction):

```bash
# Find plugin SessionStart hooks (cache layout: vendor/plugin/version/hooks/)
for d in ~/.claude/plugins/cache/*/*/*/hooks/; do
  [ -d "$d" ] || continue
  [ -f "${d}session-start.sh" ] && printf '%s bytes: %s\n' "$(wc -c < "${d}session-start.sh")" "${d}session-start.sh"
done 2>/dev/null

# Check hook matchers (compact in matcher = re-fires on compaction)
for f in ~/.claude/plugins/cache/*/*/*/hooks/hooks.json; do
  [ -f "$f" ] || continue
  printf '%s: %s\n' "$f" "$(jq -r '.hooks[]?[]?.matcher // empty' "$f" 2>/dev/null)"
done 2>/dev/null

# Measure each UserPromptSubmit hook's actual output size
jq -r '.hooks.UserPromptSubmit[]?.hooks[]?.command // empty' ~/.claude/settings.json 2>/dev/null | while read -r cmd; do
  # Expand ~ to $HOME safely (no eval)
  expanded="${cmd/#\~/$HOME}"
  [ -f "$expanded" ] && printf 'Hook output: %s bytes — %s\n' "$(bash "$expanded" 2>/dev/null | wc -c)" "$expanded"
done
```

## Step 4: Optimize

Present findings, then apply only fixes the user approves:

**MEMORY.md** — Restructure as lean index (<100 lines) + topic files:
```
memory/MEMORY.md     ← Index only, auto-loaded (first 200 lines)
memory/topic-a.md    ← On-demand, NOT auto-loaded
memory/archive.md    ← Historical notes
```

**Skills** — Archive unused (reversible — backup, not delete):
```bash
mkdir -p ~/.claude/skills-archive
# Archive all UNUSED skills identified in Step 2:
for skill in <list-unused-skill-names-here>; do
  [ -d ~/.claude/skills/"$skill" ] && mv ~/.claude/skills/"$skill" ~/.claude/skills-archive/
done
# Restore: mv ~/.claude/skills-archive/<name> ~/.claude/skills/
```

**Plugins** — Disable unused in `~/.claude/settings.json` → `enabledPlugins` (set to `false`). Re-enable by setting back to `true`.

**Hooks** — Slim verbose hooks to 1-2 lines with same semantic meaning.

**Session logs** — Identify current session first, then remove old ones:
```bash
ls -lt ~/.claude/projects/*/*.jsonl 2>/dev/null | head -3  # current sessions — DO NOT delete these
# Then rm specific old session IDs after confirming they're not current
```

**Optional env vars** (check your Claude Code version supports these):
- `SLASH_COMMAND_TOOL_CHAR_BUDGET` — caps skill definitions in prompt
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` — override auto-compact threshold (default ~95%)

**Skill frontmatter** — For rarely-used skills that should stay registered:
```yaml
disable-model-invocation: true
```

## Step 5: Report

Re-run Step 1 measurements and present before/after savings.
