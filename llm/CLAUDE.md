# CLAUDE.md

LLM-facing assets for this tree. Three layers:

- **`SKILL.md`** — Claude Code skill describing acme's 9P control
  surface (`/mnt/acme`), ctl/nctl verbs, the `event` protocol, rc-script
  conventions, and the absolute rules ("never write `dirty`/`clean` to
  `ctl`", "never mutate a dirty window"). Source of truth for any new
  agent driving a live acme or writing acme rc scripts.

- **Auto-sync wiring** — `bin/acme-update` is a Claude Code PostToolUse
  hook that keeps acme windows in step with `Edit` / `Write` /
  `MultiEdit` / `NotebookEdit` calls. `bin/acme-sync-recent` is the
  manual driver behind the `/update-acme` slash command
  (`commands/update-acme.md`). `install-hook.rc` performs a safe
  jq-based merge into `~/.claude/settings.json` (with rollback). All
  installed by `mk install`.

- **Prompt templates** consumed by hand (`plan`, `implement`, `verify`)
  and the working `TODO` for in-flight work. Nothing installs them;
  copy/paste them into a session.

## Install / uninstall

`mk install` ⇒ writes:
```
~/.claude/skills/acme/SKILL.md
~/.claude/skills/acme/bin/acme-update           (chmod +x)
~/.claude/skills/acme/bin/acme-sync-recent      (chmod +x)
~/.claude/commands/update-acme.md
~/.claude/settings.json                         (PostToolUse entry)
```

`mk uninstall` removes them and de-registers the hook. The settings
merger keeps a `.bak` of the live `settings.json` before each
modification and refuses to clobber it if the merge would produce
invalid JSON.

This `mkfile` is standalone: neither `./INSTALL` at the tree root nor
`mk install` from the root descend here.

## When to update SKILL.md

- Whenever a new `ctl` or `nctl` verb is added or removed in
  `src/cmd/acme/xfid.c`, update the verb list.
- Whenever a new rc idiom proves useful (or a footgun is discovered),
  add it to the "Common idioms" or "Hard rules" section.
- Whenever `acme(1)` or `acme(4)` changes meaningfully, re-check the
  catalogue and event protocol sections.
- Whenever the hook's behaviour changes (`bin/acme-update`), keep the
  "Auto-syncing acme with Claude Code edits" section in step.

Keep paths in `SKILL.md` project-relative so the file stays portable
when this directory is split into its own repo (per the `scripts/`
README note).

## When to update the hook

- New tool name to react to (e.g. a future `Refactor` tool): extend the
  `~ $tool …` guard in `bin/acme-update` AND the `matcher` regex
  emitted by `install-hook.rc`.
- New best-effort selection rule (e.g. anchor on `\b<symbol>\b` rather
  than a literal first line): edit `set_dot` in `bin/acme-update`.
- New "open new window" trigger (currently only `Write`): expand the
  `if (~ $tool Write)` test in the no-window branch.

After any change, `mk install` reinstalls the hook (idempotent for
add).

## Plan 9 rc quirks worth knowing

- **Heredocs do not work inside `{}` blocks.** A `cat <<EOF` between an
  `if (cond) { … }` brace pair will be parsed as commands, not as a
  heredoc body. Either move the heredoc to a top-level statement or
  wrap it in a `fn name { … }` *and* use a series of `echo` statements
  instead of a heredoc (functions have the same `{}` problem for
  heredocs). `install-hook.rc` uses the function + `echo` pattern.
- `^` is concatenation: `'a'^$var^'b'`.
- `~ var pat1 pat2 …` is the multi-pattern match used as a condition;
  `! ~ var pat` negates.
- ``` `{cmd} ``` is command substitution.
- `>[2]/dev/null` discards stderr; `>[1=2]` makes stdout go to stderr.
- Use `if (cond) cmd` (single statement, no braces) when a heredoc is
  the only thing inside the branch.
