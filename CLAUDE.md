# claude-context-audit

Agent skill for Claude Code. Audits context window overhead and recommends cuts.

## Structure

```
SKILL.md        — 5-step audit workflow with embedded bash
README.md       — User docs: install, usage, limitations
LICENSE.txt     — MIT
docs/CODEBASE_MAP.md — Architecture and navigation
```

## Skill Workflow

1. **Measure**: `wc` CLAUDE.md + MEMORY.md, count skills, `jq` settings.json, `du` session logs
2. **Audit skills**: grep skill names in last 5 `.jsonl` session logs → ACTIVE/UNUSED
3. **Audit hooks**: find plugin SessionStart hooks, measure UserPromptSubmit hook output
4. **Optimize**: restructure MEMORY.md, archive skills, disable plugins, slim hooks, clean logs
5. **Report**: re-run measurements, show before/after

## Key Constraints

- All optimizations reversible (archive, not delete)
- User approval required before applying fixes
- `jq` required for JSON parsing (graceful fallback if missing)
- Only checks last 5 session logs for skill usage
- Measures bytes not tokens (approximate)
- Shell-safe: no `eval`, `2>/dev/null` on all globs, safe tilde expansion

## Editing Checklist

- Changing bash commands? Keep `2>/dev/null` on globs (zsh nomatch safety)
- Adding strategies? Update both `SKILL.md` Step 4 AND `README.md` optimization section
- New trigger? Add to `SKILL.md` frontmatter `triggers` array
- Bumping version? `SKILL.md` frontmatter `metadata.version`

See `docs/CODEBASE_MAP.md` for full architecture.
