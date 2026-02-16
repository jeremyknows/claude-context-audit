# claude-context-audit

Audit and optimize Claude Code context window usage. Measures overhead from MEMORY.md, skills, plugins, hooks, and session logs — then recommends specific cuts.

## What it does

- Measures byte/line counts for CLAUDE.md and all MEMORY.md files
- Counts active skills and reports which ones actually appear in recent sessions
- Audits plugin SessionStart hooks (which re-fire on every compaction)
- Measures UserPromptSubmit hook output sizes
- Finds large session log files consuming disk
- Recommends specific optimizations with reversible commands

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
| **MEMORY.md** | First 200 lines auto-loaded every session. Large files waste context on every turn. |
| **Skills** | Each registered skill adds ~200 chars to the system prompt. Unused skills are dead weight. |
| **Plugins** | SessionStart hooks re-inject on every compaction — the more you compact, the more they cost. |
| **Hooks** | UserPromptSubmit hooks fire on every user message. Verbose hooks add up fast. |
| **Session logs** | Old `.jsonl` files consume disk and slow down glob operations. |

## Optimization strategies

The skill walks you through each of these, applying only what you approve:

1. **MEMORY.md** — Restructure as a lean index (<100 lines) with topic files loaded on-demand
2. **Skills** — Archive unused skills to `~/.claude/skills-archive/` (reversible)
3. **Plugins** — Disable unused in `settings.json` (flip `false` to `true` to re-enable)
4. **Hooks** — Slim verbose hooks to 1-2 lines with the same semantic meaning
5. **Session logs** — Remove old logs after confirming they're not the current session
6. **Frontmatter** — Add `disable-model-invocation: true` to rarely-used skills

## Prerequisites

- `jq` — used to parse `settings.json` (falls back gracefully if missing)
- `wc`, `du`, `ls`, `grep` — standard Unix tools

## Limitations

- Measures file sizes, not actual token counts (tokens vary by content)
- Skill usage audit only checks the 5 most recent session logs — older usage isn't captured
- Cannot detect MEMORY.md double-loading at the source level (this is a Claude Code internals issue)
- Plugin cache layout may change between Claude Code versions

## File structure

```
claude-context-audit/
├── SKILL.md       # Skill instructions and audit steps
├── LICENSE.txt    # MIT license
└── README.md      # This file
```

## License

MIT — see [LICENSE.txt](LICENSE.txt).
