# CLAUDE.md

LLM-facing assets for this tree. Four layers:

- **`acme/SKILL.md`** — Claude Code skill describing acme's 9P control
  surface (`/mnt/acme`), ctl/nctl verbs, the `event` protocol, rc-script
  conventions, and the absolute rules ("never write `dirty`/`clean` to
  `ctl`", "never mutate a dirty window"). Source of truth for any new
  agent driving a live acme or writing acme rc scripts.

- **`rc/SKILL.md`** — Claude Code skill for plan9port's `rc` shell:
  the language itself (lists, `~` match, free carets, `` `{} ``
  substitution, `>[2]` redirections, heredoc rules), the p9p tool
  flavours (`test`, `awk`, `sed`, `grep`, `tr`, `wc`, `regexp(7)`),
  and a cheat sheet of bash vs rc reflexes. Source of truth whenever
  a new script is written or an existing one is edited; cross-linked
  from `acme/SKILL.md` for scripts that drive `/mnt/acme`.

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
~/.claude/skills/rc/SKILL.md
~/.claude/commands/update-acme.md
~/.claude/settings.json                         (PostToolUse entry)
```

Granular targets: `install-acme-skill`, `install-rc-skill`,
`install-skill` (both), `install-bin`, `install-command`,
`register-hook`. Symmetrical `uninstall-*` targets reverse each step.

`mk uninstall` removes everything and de-registers the hook. The
settings merger keeps a `.bak` of the live `settings.json` before each
modification and refuses to clobber it if the merge would produce
invalid JSON.

This `mkfile` is standalone: neither `./INSTALL` at the tree root nor
`mk install` from the root descend here.

## When to update acme/SKILL.md

- Whenever a new `ctl` or `nctl` verb is added or removed in
  `src/cmd/acme/xfid.c`, update the verb list.
- Whenever a new rc idiom proves useful (or a footgun is discovered),
  add it to the "Common idioms" or "Hard rules" section here — and
  cross-check whether the same idiom belongs in `rc/SKILL.md`.
- Whenever `acme(1)` or `acme(4)` changes meaningfully, re-check the
  catalogue and event protocol sections.
- Whenever the hook's behaviour changes (`bin/acme-update`), keep the
  "Auto-syncing acme with Claude Code edits" section in step.

Keep paths in `acme/SKILL.md` project-relative so the file stays portable
when this directory is split into its own repo (per the `scripts/`
README note).

## When to update rc/SKILL.md

- Whenever the `rc(1)` man page changes meaningfully (grammar,
  built-in list, redirection syntax) — re-check the language section
  and the bash/POSIX cheat-sheet.
- Whenever a new p9p text tool quirk bites a script (a flag missing
  vs GNU, a `regexp(7)` shortcoming, a different default), add it to
  the "Plan 9 toolchain quirks" section with a one-line workaround.
- Whenever the rc footguns list grows (an empirical discovery while
  driving acme or writing a tag script), capture it under "Pitfalls".
- Whenever a new acme-related script is added to `scripts/` that
  exercises a non-trivial rc construct, link it from the "Pointers"
  section as a worked example.

Keep `rc/SKILL.md` agnostic of acme specifics where possible — defer
to `acme/SKILL.md` for ctl/nctl, dirty rules, event protocol — so the
file stays useful for any rc work, not just acme tag scripts.

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
- **`for(line in `{cmd})` splits on every whitespace, not on
  newlines.** A multi-field record (e.g. one line of `acme/index`)
  becomes one iteration per word; `awk '{print $K}'` for K>1 then
  returns empty every time. Use ``` `$nl{cmd} ``` with `nl='\n'` to
  split strictly on newlines whenever the records contain spaces.
  `bin/acme-update`'s `find_window` was originally written without this
  and silently never matched a window; see `rc/SKILL.md` "Reading one
  field out of structured output".
- **`return` is not a builtin in plan9port rc.** Inside an `fn`, it is
  treated as an external command and fails with `return: No such file
  or directory`. To short-circuit a scan in a function, either let an
  `awk`/`sed` helper exit at the first hit (current `find_window`
  pattern) or set a sentinel variable and guard subsequent iterations.
  See `rc/SKILL.md` Pitfalls.
- `^` is concatenation: `'a'^$var^'b'`.
- `~ var pat1 pat2 …` is the multi-pattern match used as a condition;
  `! ~ var pat` negates.
- ``` `{cmd} ``` is command substitution.
- `>[2]/dev/null` discards stderr; `>[1=2]` makes stdout go to stderr.
- Use `if (cond) cmd` (single statement, no braces) when a heredoc is
  the only thing inside the branch.

## Acme quirks the hook depends on

- **`dot=addr` vs `addr=dot` are opposites.** The hook always writes
  `dot=addr` (move the user's selection onto the address just written
  to `<id>/addr`); `addr=dot` would do the inverse and is never used.
  Don't mix them up when describing or modifying `set_dot`.
- **`<id>/addr` must be written without a trailing newline.** The
  parser rejects `\n` as a non-address character and falls back to
  the previous address (often `0,0`), silently. The hook uses
  `awk 'BEGIN{printf a}'` for this reason; preserve that pattern.
- **The hook never reads `<id>/ctl`.** Dirtiness comes from field 5
  of `acme/index`, fetched once by `find_window` in a single `awk`
  pass. If you need a new per-window attribute, prefer extending
  that pass over adding a `9p read acme/<id>/ctl`.
- **The hook never touches `<id>/body`, `<id>/data`, `<id>/xdata`,
  `<id>/tag`, or `<id>/event`.** All buffer refreshes go through
  `echo get | 9p write <id>/ctl` (acme reloads from disk). Keep this
  property if you extend the hook — direct `data` writes break the
  "never mutate a dirty window" invariant the moment a race opens.

## See also

- `README.md` — file-by-file table, installation summary, and three
  rc implementation notes (newline preservation, space-safe iteration,
  awk-as-fixed-string-grep). Front door for someone arriving at the
  directory for the first time.
- `codebase_analysis.md` — whole-directory analysis: per-file
  breakdown, exact 9P surfaces consumed by the hook, architecture
  diagrams, security and performance assessment, and a punch list of
  potential improvements. Update when the directory layout changes
  meaningfully, when the hook's 9P surface changes, or when a new
  script joins `bin/`.
