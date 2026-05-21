# llm/

LLM-facing assets for this tree.

| File | Purpose |
|---|---|
| `SKILL.md` | Claude Code skill: how to drive a live acme via 9P and write rc scripts for it. Authoritative on the ctl/nctl/event surface and on the hard rules (no `dirty`/`clean` writes; no mutation of dirty windows). |
| `plan` | Prompt template: build a meticulously detailed implementation plan. Writes to `./PLAN.md`. |
| `implement` | Prompt template: implement the next stage of `./PLAN.md`, test it, record in `./DONE`. |
| `verify` | Prompt template: verify the latest stage marked done. |
| `TODO` | In-flight task notes (currently: converting the in-editor `Emph*` commands to standalone rc scripts). |

## Install the skill

```
mk install        # -> $HOME/.claude/skills/acme/SKILL.md
mk uninstall
```

This `mkfile` is standalone: the tree-root `./INSTALL` and `mk install`
do not descend into this directory.

## See also

- `scripts/` — the rc scripts SKILL.md uses as worked examples
  (`A`, `F`, `Bold`, `Color`, `EmphAs`, …).
- `src/cmd/acme/` — acme's C source, `CLAUDE.md`, and
  `codebase_analysis.md`.
- `CODE_CHANGES.md` (tree root) — local divergence from upstream
  plan9port (the emphasis feature).
