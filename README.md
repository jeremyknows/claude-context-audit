# claude-context-audit

Audit and optimize Claude Code context window usage. Measures overhead from MEMORY.md, skills, plugins, hooks, and session logs — then recommends specific cuts with estimated token savings.

## What it does

- Measures byte/line/token counts for CLAUDE.md (global + project-level) and all MEMORY.md files
- Measures actual content size per skill directory — flags multi-MB outliers
- Audits plugin SessionStart hooks by measuring **output size**, not just script size
- Measures UserPromptSubmit hook output sizes (per-prompt tax)
- Totals reclaimable disk space from old session logs
- Reports a summary table with estimated tokens and load frequency
- Recommends three tiers of skill action: keep, suppress (`disable-model-invocation`), or archive

## Install

### Claude Code

```bash
git clone git@github.com:jeremyknows/claude-context-audit.git ~/.claude/skills/claude-context-audit
```

### Other agents

Copy `SKILL.md` into your agent's skills directory.

## Usage

### Natural language

Just say any of these:

- "context audit"
- "optimize context"
- "context filling up"
- "compacting too often"
- "token usage high"

### Direct invocation

```
/claude-context-audit
```

## What it audits

| Source | Why it matters |
|--------|---------------|
| **CLAUDE.md** | Global and project-level files loaded every session. |
| **MEMORY.md** | First 200 lines auto-loaded **twice** every session (known Claude Code behavior). |
| **Skills** | Each registered skill adds ~200 chars to the system prompt. Unused skills are dead weight. |
| **Plugins** | SessionStart hooks re-inject on every compaction — the more you compact, the more they cost. |
| **Hooks** | UserPromptSubmit hooks fire on every user message. Verbose hooks add up fast. |
| **Session logs** | Old `.jsonl` files consume disk. |

## Optimization strategies

The skill walks you through each of these, applying only what you approve:

1. **MEMORY.md** — Restructure as a lean index (<100 lines) with topic files loaded on-demand
2. **Skills (archive)** — Move unused skills to `~/.claude/skills-archive/` (reversible)
3. **Skills (suppress)** — Add `disable-model-invocation: true` to rarely-needed skills to remove from prompt without uninstalling
4. **Plugins** — Disable unused in `settings.json` (flip `false` to `true` to re-enable)
5. **Hooks** — Slim verbose hooks to 1-2 lines with the same semantic meaning
6. **Session logs** — Delete logs older than 7 days with `find -mtime +7 -delete`

## Prerequisites

- `jq` — used to parse `settings.json` (falls back gracefully if missing)
- `wc`, `du`, `find`, `grep`, `bash` — standard Unix tools

## Limitations

- Token estimates use bytes÷4 approximation — actual tokens vary by content
- Skill usage audit only checks the 5 most recent session logs — older usage isn't captured
- Cannot prevent MEMORY.md double-loading (Claude Code internals) — only mitigation is keeping it small
- Plugin cache layout may change between Claude Code versions

## File structure

```
claude-context-audit/
├── SKILL.md           # Skill instructions and audit steps
├── README.md          # This file
├── LICENSE.txt        # MIT license
└── docs/
    └── CODEBASE_MAP.md  # Architecture and navigation
```

## License

MIT — see [LICENSE.txt](LICENSE.txt).
