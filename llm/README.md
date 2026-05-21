# llm/

LLM-facing assets for this tree.

## Skill, hook, and slash command

| File | Purpose |
|---|---|
| `acme/SKILL.md` | Claude Code skill: how to drive a live acme via 9P and write rc scripts for it. Authoritative on the ctl/nctl/event surface and the hard rules (no `dirty`/`clean` writes; no mutation of dirty windows). |
| `bin/acme-update` | PostToolUse hook + manual single-file sync helper. Reloads the matching acme window when Claude edits a file (clean windows only; dirty ones get a warning in `+Errors`). |
| `bin/acme-sync-recent` | Walks `git diff --name-only HEAD` + untracked files and calls `acme-update` on each. The body of the `/update-acme` slash command. |
| `commands/update-acme.md` | Slash command `/update-acme` for manual sync (fallback when the hook is disabled). |
| `install-hook.rc` | Settings merger. Adds/removes the PostToolUse entry in `~/.claude/settings.json` with a `.bak` rollback and a JSON-validity check. Falls back to printing a manual-merge snippet when `jq` is missing. |

## Prompt templates (consumed by hand)

| File | Purpose |
|---|---|
| `plan` | Build a meticulously detailed implementation plan from `./TODO`; output goes to `./PLAN.md`. |
| `implement` | Implement the next stage of `./PLAN.md`, test it, append to `./DONE`. |
| `verify` | Verify the last completed stage. |
| `TODO` | In-flight task notes (currently: converting the in-editor `Emph*` commands to standalone rc scripts). |

## Install

```
mk install            # all four: skill, bin, slash command, hook config
mk install-skill      # just SKILL.md → ~/.claude/skills/acme/
mk install-bin        # just bin/* → ~/.claude/skills/acme/bin/
mk install-command    # just the slash command → ~/.claude/commands/
mk register-hook      # just modify ~/.claude/settings.json
```

`mk uninstall` removes everything; granular `uninstall-*` targets match
each install target.

`mk install` overall result:

```
~/.claude/
├── settings.json                     (PostToolUse entry added)
├── commands/update-acme.md
└── skills/acme/
    ├── SKILL.md
    └── bin/
        ├── acme-update
        └── acme-sync-recent
```

Note: `SKILL.md` is copied from `acme/SKILL.md` in the tree.

Requirements: `jq` (for the settings merge) and `9p` (plan9port). The
hook itself exits 0 silently when either is unavailable, so it never
breaks Claude Code; the install merely complains and prints a snippet
when `jq` is missing.

This `mkfile` is standalone: the tree-root `./INSTALL` and `mk install`
do not descend into this directory.

## See also

- `scripts/` — the rc scripts `acme/SKILL.md` uses as worked examples (`A`,
  `F`, `Bold`, `Color`, `EmphAs`, …).
- `src/cmd/acme/` — acme's C source, `CLAUDE.md`, and
  `codebase_analysis.md`.
- `OBSERVATIONS.md` (tree root) — gaps between `man/man4/acme.4` and
  the code, and follow-up work spotted while writing the skill.
- `CODE_CHANGES.md` (tree root) — local divergence from upstream
  plan9port (the emphasis feature).
