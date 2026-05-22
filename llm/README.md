# llm/

LLM-facing assets for this tree.

## Skills, hook, and slash command

| File | Purpose |
|---|---|
| `acme/SKILL.md` | Claude Code skill: how to drive a live acme via 9P and write rc scripts for it. Authoritative on the ctl/nctl/event surface and the hard rules (no `dirty`/`clean` writes; no mutation of dirty windows). |
| `rc/SKILL.md` | Claude Code skill for the plan9port `rc` shell and the text tools it composes (`test`, `awk`, `sed`, `grep`, `tr`, `regexp(7)`). Covers the language proper (lists, `~`, free carets, redirections, heredoc rules), the bash-vs-rc cheat sheet, and the pitfalls that bite scripters with Bourne reflexes. Companion to `acme/SKILL.md`. |
| `bin/acme-update` | PostToolUse hook + manual single-file sync helper. Reloads the matching acme window when Claude edits a file (clean windows only; dirty ones get a warning in `+Errors`). |
| `bin/acme-sync-recent` | Walks `git diff --name-only HEAD` + untracked files and calls `acme-update` on each. The body of the `/update-acme` slash command. |
| `commands/update-acme.md` | Slash command `/update-acme` for manual sync (fallback when the hook is disabled). |
| `install-hook.rc` | Settings merger. Adds/removes the PostToolUse entry in `~/.claude/settings.json` with a `.bak` rollback and a JSON-validity check. Falls back to printing a manual-merge snippet when `jq` is missing. Worked example of rc's heredoc-in-`{}` workaround (function full of `echo` lines). |

### Implementation notes

The helper scripts are written in Plan 9 `rc` and use a few specific constructs to work robustly with JSON and filenames:
- **Newline preservation**: We capture the `stdin` JSON payload using an empty separator (`` `''{cat} ``) so `rc` does not squash newlines and spaces, preventing corruption of JSON string values.
- **Space-safe iteration**: We use a literal newline variable (`nl='
'`) as the separator for command substitutions (e.g. `` `$nl{git diff...} ``) so filenames and JSON strings with internal spaces are preserved as single list elements.
- **Standard awk**: Plan 9 `grep` implements `regexp(7)` and lacks GNU extensions like `-F` and `-m1`. We instead use standard `awk -v s=$firstline 'index($0, s)'` for safe, fixed-string prefix matching against file contents.

## Prompt templates (consumed by hand)

| File | Purpose |
|---|---|
| `plan` | Build a meticulously detailed implementation plan from `./TODO`; output goes to `./PLAN.md`. |
| `implement` | Implement the next stage of `./PLAN.md`, test it, append to `./DONE`. |
| `verify` | Verify the last completed stage. |
| `TODO` | In-flight task notes (currently: converting the in-editor `Emph*` commands to standalone rc scripts). |

## Reference docs

| File | Purpose |
|---|---|
| `CLAUDE.md` | Maintainer guide: when to update each `SKILL.md` or the hook, plus an "Acme quirks the hook depends on" cheat sheet (notably `dot=addr` vs `addr=dot`, the no-trailing-newline rule for `<id>/addr`, and the buffers the hook deliberately never touches). |
| `codebase_analysis.md` | Whole-directory analysis: per-file breakdown, the exact 9P surfaces the hook reads/writes (and the ones it never touches), execution and dependency diagrams, security and performance notes, suggested improvements. |

## Install

```
mk install              # everything: both skills, bin, command, hook
mk install-skill        # both SKILL.md files
mk install-acme-skill   # just acme/SKILL.md → ~/.claude/skills/acme/
mk install-rc-skill     # just rc/SKILL.md → ~/.claude/skills/rc/
mk install-bin          # just bin/* → ~/.claude/skills/acme/bin/
mk install-command      # just the slash command → ~/.claude/commands/
mk register-hook        # just modify ~/.claude/settings.json
```

`mk uninstall` removes everything; granular `uninstall-*` targets
match each install target (`uninstall-rc-skill`,
`uninstall-acme-skill`, …).

`mk install` overall result:

```
~/.claude/
├── settings.json                     (PostToolUse entry added)
├── commands/update-acme.md
└── skills/
    ├── acme/
    │   ├── SKILL.md
    │   └── bin/
    │       ├── acme-update
    │       └── acme-sync-recent
    └── rc/
        └── SKILL.md
```

Note: each `SKILL.md` is copied from its source in `acme/` or `rc/` in
the tree.

Requirements: `jq` (for the settings merge) and `9p` (plan9port). The
hook itself exits 0 silently when either is unavailable, so it never
breaks Claude Code; the install merely complains and prints a snippet
when `jq` is missing.

This `mkfile` is standalone: the tree-root `./INSTALL` and `mk install`
do not descend into this directory.

## See also

- `scripts/` — the rc scripts that `acme/SKILL.md` and `rc/SKILL.md`
  use as worked examples (`A`, `F`, `Bold`, `Color`, `EmphAs`, …).
- `src/cmd/acme/` — acme's C source, `CLAUDE.md`, and
  `codebase_analysis.md`.
- `src/cmd/rc/` — the rc implementation itself; `rcmain` (tree root)
  is what rc evaluates at startup on plan9port.
- `OBSERVATIONS.md` (tree root) — gaps between `man/man4/acme.4` and
  the code, and follow-up work spotted while writing the skill.
- `CODE_CHANGES.md` (tree root) — local divergence from upstream
  plan9port (the emphasis feature).
