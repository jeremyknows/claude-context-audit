---
last_mapped: "2026-02-16T22:00:00Z"
total_files: 4
total_tokens: 3200
---

# Codebase Map — claude-context-audit

## Overview

A Claude Code agent skill that audits context window overhead and recommends optimizations. No executable code — instructional markdown with embedded bash snippets. v2 adds accurate skill invocation detection, token estimation, hook output measurement, and three-tier skill optimization.

```mermaid
flowchart TB
    SKILL[SKILL.md<br/>Audit instructions v2]
    README[README.md<br/>User documentation]
    LICENSE[LICENSE.txt<br/>MIT license]

    SKILL -->|mirrors structure| README
    SKILL -->|declares license| LICENSE
```

## Directory Tree

```
claude-context-audit/          # Root — agent skill
├── SKILL.md          (1,900t) # Audit workflow: 5 steps with bash commands
├── README.md           (750t) # Install, usage, strategies, limitations
├── LICENSE.txt         (220t) # MIT — Jeremy Knows, 2026
├── .gitignore            (4t) # .DS_Store
└── docs/
    └── CODEBASE_MAP.md        # This file
```

## Key Files

| File | Purpose | Key Content |
|------|---------|-------------|
| `SKILL.md` | Agent instructions | 5-step audit: measure (with token estimates), audit skills (invocation-pattern grep), audit hooks (output measurement), optimize (three-tier), report (before/after table) |
| `README.md` | User documentation | Install, natural language triggers, audit targets, three-tier optimization strategies, limitations |
| `LICENSE.txt` | Legal | MIT license |

## Audit Data Flow

```mermaid
sequenceDiagram
    participant U as User
    participant CC as Claude Code
    participant FS as Filesystem

    U->>CC: "context audit"
    Note over CC: Step 1: Measure
    CC->>FS: wc CLAUDE.md (global + project-level)
    CC->>FS: wc MEMORY.md files
    CC->>FS: find + cat skill content sizes
    CC->>FS: jq settings.json
    CC->>FS: du session logs + find -mtime +7 totals
    CC-->>U: Summary table (bytes, tokens, load frequency)

    Note over CC: Step 2: Skill Usage
    CC->>FS: find recent .jsonl logs (exclude subagents)
    CC->>FS: grep invocation patterns (not bare names)
    CC-->>U: ACTIVE / UNUSED categorization

    Note over CC: Step 3: Hooks
    CC->>FS: bash SessionStart hooks, measure output size
    CC->>FS: bash UserPromptSubmit hooks, measure output size
    CC->>FS: check hook matchers for compaction triggers
    CC-->>U: Flag hooks >500 bytes output

    Note over CC: Step 4: Optimize
    CC-->>U: Three-tier recommendations (keep/suppress/archive)
    U->>CC: Approve specific fixes

    Note over CC: Step 5: Report
    CC->>FS: Re-run measurements
    CC-->>U: Before/after savings table
```

## Claude Code Internals Referenced

| Path | What | Why It Matters |
|------|------|----------------|
| `~/.claude/CLAUDE.md` | Global instructions | Loaded every session |
| `./CLAUDE.md` (+ nested) | Project-level instructions | Loaded every session when in that project |
| `~/.claude/projects/*/memory/MEMORY.md` | Per-project memory | First 200 lines auto-loaded **twice** (known behavior) |
| `~/.claude/skills/*/` | Skill registry | Each adds ~200 chars to system prompt |
| `~/.claude/skills-archive/` | Archived skills | Not loaded — restore by moving back to `skills/` |
| `~/.claude/settings.json` | Plugins + hooks config | `enabledPlugins`, `hooks` sections |
| `~/.claude/plugins/cache/*/*/*/hooks/` | Plugin hook scripts | SessionStart hooks re-fire on every compaction |
| `~/.claude/projects/*/*.jsonl` | Session logs | Disk usage, skill usage audit source |

## v2 Changes (from v1)

| Area | v1 | v2 |
|------|----|----|
| Skill usage detection | `grep "$name"` (false positives from registry) | Regex matching invocation patterns only |
| Skill sizing | Count of directories | `find -exec cat` for actual content bytes; flags >1MB |
| Hook measurement | Script file size (`wc -c < script`) | Script output size (`bash script \| wc -c`) |
| CLAUDE.md scope | Global only | Global + project-level (`find . -maxdepth 3`) |
| Token estimates | Not included | bytes÷4 throughout, summary table with load frequency |
| Disk totals | Top 10 files only | Adds total reclaimable (count + MB for >7 days old) |
| Skill optimization | Keep or archive | Three tiers: keep / suppress (`disable-model-invocation`) / archive |
| Log cleanup | Manual identification | `find -mtime +7 -delete` with empty dir cleanup |
| Report | Re-run measurements | Structured before/after comparison table |

## Bash Dependencies

`wc`, `du`, `find`, `grep`, `jq`, `printf`, `sort`, `head`, `bash`, `mv`, `xargs`, `ls`, `awk`

Only `jq` is non-standard — skill falls back gracefully if missing.

## Conventions

- **Safety first**: All optimizations are reversible (archive, not delete)
- **User approval required**: Findings presented before any changes applied
- **Shell safety**: `2>/dev/null` on all globs (zsh compatibility), no `eval`, safe tilde expansion via `${cmd/#\~/$HOME}`
- **Bash arithmetic**: `count=$((count + 1))` instead of `((count++))` (avoids exit 1 with `set -e`)
- **Measure output, not source**: Hook context cost = script output size, not file size

## Navigation Guide

| To do this... | Touch these files |
|---------------|-------------------|
| Add a new audit check | `SKILL.md` — add a new bash block in the appropriate step |
| Update install instructions | `README.md` — Install section |
| Change the optimization strategies | `SKILL.md` Step 4, `README.md` Optimization strategies section |
| Add a new trigger phrase | `SKILL.md` frontmatter `triggers` array |
| Update version number | `SKILL.md` frontmatter `metadata.version` |
| Verify cross-file consistency | Compare `SKILL.md` Step 4 against `README.md` optimization strategies |
| Add a new Claude Code internal path | `CODEBASE_MAP.md` internals table + relevant `SKILL.md` step |
