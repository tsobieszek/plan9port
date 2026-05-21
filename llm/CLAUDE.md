# CLAUDE.md

LLM-facing assets for this tree. Two kinds of file live here:

- **`SKILL.md`** — the Claude Code skill describing acme's 9P control
  surface (`/mnt/acme`), ctl/nctl verbs, the `event` protocol, rc-script
  conventions, and the absolute rules ("never write `dirty`/`clean` to
  `ctl`", "never mutate a dirty window"). It is the source of truth for
  any new agent driving a live acme or writing acme rc scripts.
- **Prompt templates** consumed by hand (`plan`, `implement`, `verify`)
  and the working `TODO` for in-flight work. These are not installed
  anywhere; copy/paste them into a session.

## Install

`mk install` copies `SKILL.md` to `$HOME/.claude/skills/acme/SKILL.md`
so Claude Code picks it up as the `acme` skill. `mk uninstall` removes
it. This `mkfile` is standalone: neither `./INSTALL` at the tree root
nor `mk install` from the root descend here.

## When to update SKILL.md

- Whenever a new `ctl` or `nctl` verb is added or removed in
  `src/cmd/acme/xfid.c`, update the verb list.
- Whenever a new rc idiom proves useful (or a footgun is discovered),
  add it to the "Common idioms" or "Hard rules" section.
- Whenever `acme(1)` or `acme(4)` changes meaningfully, re-check the
  catalogue and event protocol sections.

Keep paths in `SKILL.md` project-relative so the file stays portable
when this directory is split into its own repo (per the `scripts/`
README note).
