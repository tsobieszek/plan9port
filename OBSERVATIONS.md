# Observations — 2026-05-21

Notes collected while writing `llm/acme/SKILL.md` and cross-checking it against
the codebase. Each item is scoped narrowly enough to be actionable on its
own.

## 1. `man/man4/acme.4` diverges from the code

The man page is the project's stated source of truth, but `xfid.c` and
`wind.c` disagree with it on several points. Each merits a small patch to
the troff source.

### 1.1. `addr` read format

`acme(4)` says:

> When read, it returns the value of the address … in the format `#m,#n`
> where m and n are character offsets. If m and n are identical, the
> format is just `#m`.

The code (`xfid.c:391`) actually writes:

```c
sprint(buf, "%11d %11d ", w->addr.q0, w->addr.q1);
```

— **two fixed-width 11-character decimals separated by a single space**,
with no `#` and no comma. The `#m`-single-form does not exist.

Suggested fix: update the man page paragraph at `man/man4/acme.4:172-200`.

### 1.2. `nctl` font field quoting

`acme(4)` describes the `nctl` read as:

> the effective emphasis font name (quoted, per-window override if set, …)

The code (`xfid.c:410`) uses `%s`, not `%q`:

```c
b = smprint("%s %06lux ", fn, col >> 8);
```

So a font path with whitespace (typical of fontsrv layouts like
`/mnt/font/Droid Sans Mono/13a/font`) appears unquoted and breaks the
expected `$1 = font`, `$2 = color` split. Either fix the code to match the
docs (use `%q`) or fix the docs to match the code. The `ctl` read does use
`%q`, so the asymmetry between `ctl` and `nctl` is at least surprising —
aligning on `%q` would be the cleaner of the two.

### 1.3. Undocumented `ctl` verbs

`xfid.c:xfidctlwrite` implements four verbs missing from `acme(4)`:

| Verb | Lines | Effect |
|---|---|---|
| `lock` | 727–731 | take `w->ctllock`, record `w->ctlfid` — exclusive ctl owner |
| `unlock` | 732–735 | release the lock |
| `menu` | 895–898 | re-enable the auto command menu in the tag |
| `nomenu` | 890–894 | suppress it |

They appear to be intentional (have been there a while) and are used by
external controllers (`win`, `mail/`). Document them under the same list
as the other `ctl` verbs in `acme(4)`.

### 1.4. Hazard of external `clean` / `dirty` writes is undocumented

Writing `clean` from outside the owning controller calls `filereset`
(`file.c:282`), which **wipes the undo/redo deltas** in addition to
clearing the mod bit — silently losing the user's Ctrl-Z stack. The man
page lists the verbs without warning.

Suggested addition near the `clean` / `dirty` entries in `acme(4)`:

> Note: `clean` and `dirty` are intended for the program that owns the
> window via 9P (e.g. `win`). Writing them from an arbitrary script
> desynchronises the dirty flag from the buffer and, for `clean`,
> additionally discards the window's undo and redo history.

## 2. `src/cmd/acme/codebase_analysis.md` is incomplete

Reasonably up-to-date but missing the following (verified against the
code on 2026-05-21):

- The `emphcolor REGEX` / `emphcolor` reset verb on `nctl` (`xfid.c:997
  ff.`) — neither verb is listed in section 6 ("Polices et emphase") of
  the analysis. The accompanying `winsetemphcolor` / `winresetemphcolor`
  functions (mentioned only as `parsecolor` consumers) deserve a line.
- The `lock` / `unlock` / `menu` / `nomenu` verbs on `ctl` are absent
  from section 3 (the file-by-file walk) and the cycle-of-a-write
  diagram.
- The `nctl` read format line (`%s %06lux`) is not documented; only the
  write verbs are.

These are five-line edits, not a rewrite.

## 3. Scripts are fragile for fontsrv paths with whitespace

`scripts/Bold`, `BoldAll`, `Color`, `ColorAll` extract the font path via:

```rc
font=`{echo $ctl | awk '{print $1}'}      # on nctl
```

The `nctl` read writes the font name *unquoted* (`%s`). For a path like
`/mnt/font/Droid Sans Mono/13a/font`, `awk '{print $1}'` returns
`/mnt/font/Droid`. The scripts therefore work only for whitespace-free
paths. Same story for `F`, `F+`, `F-`, `A`, `A+`, `A-`, which use
`awk '{print $7}'` on `ctl` — `ctl` does quote with `%q`, so a quoted
path containing a space surfaces as `'/mnt/font/Droid` for `$7`.

Two ways to fix this:

1. Make `nctl` use `%q` (see §1.2). Then update each script to strip the
   surrounding `'…'` (rc has `${var:q}`-style tooling via `~~`-quoting; a
   small `unquote` helper would suffice).
2. Add a single helper script `acmectlfont` (or expose the field via a
   different file) that returns just the font path on stdout, handling
   the quoting once.

Option (1) is the more honest fix; the man page already promises quoting.

## 4. Debugging affordances are missing

The `9lab.org` walkthrough for debugging acme with acid recommends a
`mk syms` / `mk acme.acid` workflow. **Neither target exists in this
fork's `src/cmd/acme/mkfile`.** Two minor adds would help:

```mk
syms:V:
	9 acidtypes $PLAN9/bin/acme >acme.acid

acme.acid:	$PLAN9/bin/acme
	9 acidtypes $PLAN9/bin/acme >acme.acid
```

These are convenience targets only — `9 acidtypes` works without them —
but the documented path through them is what people search for.

## 5. Root `./CLAUDE.md` doesn't surface key sub-trees

The top-level guide mentions `codebase_analysis.md`, `CODE_CHANGES.md`,
and `src/cmd/acme/`, but not:

- `llm/` — the LLM-facing skill and prompts (added 2026-05-21).
- `scripts/` — the acme rc commands that drive font / emphasis from the
  tag, and which now serve as worked examples for the skill.

A one-line bullet under "What this is" pointing at each would close the
loop and avoid sending future agents on a discovery hunt.

The top-level CLAUDE.md is also a good place to lift the **two absolute
rules** from the skill (`never write dirty/clean to ctl`, `never mutate
the body of a dirty window`), because they apply project-wide whenever
acme is involved — they should not be visible only inside `llm/acme/SKILL.md`.

## 6. Top-level convention drift: `Toujours répondre en français`

The directive is present in the **last line** of `CLAUDE.md`, almost as
an afterthought. The user-level `~/.claude/CLAUDE.md` also requires
French. New agents should not have to scroll to find a directive that
shapes every response. Consider lifting it to the very top of `CLAUDE.md`
(directly under the title) or making it the first bullet under
"Conventions worth knowing".

## 7. `EmphAs` is the only Emph-family command that is a script

All other `Emph*` commands are built into the editor (`exec.c`,
`exectab[]`); `EmphAs <lang>` is an external rc script (`scripts/EmphAs`)
that writes `emph=…` to `nctl`. The `llm/TODO` file plans to convert all
of them to scripts, which is the right direction: it reduces the
editor's surface, makes the language→pattern mapping editable without a
rebuild, and unifies discovery (everything in `~/acme/`).

When that conversion happens:

- The built-ins in `exectab[]` (`Lemph`, `LemphFont`, `LemphMe`,
  `LemphAll`, `LemphNone`, `LautoEmph`) should be removed.
- `acme(1)` loses six entries.
- `lib/emph.regexp` likely moves to `$HOME/lib/`.
- `nctl` gains a 3rd field (`emphon` toggle, per the TODO).
- `llm/acme/SKILL.md` needs the same update.

This is large enough to warrant its own PLAN/DONE pass (the `llm/`
directory already has prompts for that workflow).

## 8. Documentation file proliferation in `src/cmd/acme/`

The directory has, in addition to source: `CLAUDE.md`,
`codebase_analysis.md`, `CODE_CHANGES.md`, `NOTES`, `TASK`,
`STAGE0_SUMMARY.md`. Some redundancy: `NOTES` is partly superseded by
`CLAUDE.md` (the latter notes that explicitly), and `STAGE0_SUMMARY.md`
is a frozen snapshot of work now reflected in `codebase_analysis.md` and
`CODE_CHANGES.md`. Worth archiving `STAGE0_SUMMARY.md` (e.g. into a
`docs/history/` subdirectory) and trimming `NOTES` to just the items not
yet absorbed elsewhere — so a new agent has a clearer entry path.

Lower priority than the man-page issues; flagged for whenever someone
cleans up.

## 9. Writing to `addr` requires no trailing newline

**Discovery (2026-05-21):** the `addr` file parser (`address()` in
`addr.c`) validates that every rune written is a legal address character,
via `isaddrc()`:

```c
if(r && utfrune("0123456789+-/$.#,;?", r)!=nil)
    return TRUE;
```

A newline (`\n`) is not in that set. `xfid.c:xfidwrite` calls
`address()` and then checks `if(nb < nr)` — if the parser stopped before
consuming all input, it returns `Ebadaddr` (`"bad address syntax"`). The
subsequent `data` write then operates on whatever `addr` was before the
failed write (typically `0,0`), silently **prepending** the new content
instead of replacing it.

`echo` always appends `\n`. All `echo addr | 9p write .../addr` calls in
`llm/acme/SKILL.md` and `llm/bin/acme-update` were therefore broken; in
`acme-update` the error was silenced by `>[2]/dev/null`, hiding the bug.

**Fix pattern:** use `awk 'BEGIN{printf "addr"}'` to write without newline:

```rc
awk 'BEGIN{printf "0,$"}' | 9p write acme/$winid/addr
awk -v n=$ln 'BEGIN{printf n}' | 9p write acme/$winid/addr
```

`llm/acme/SKILL.md` and `llm/bin/acme-update` have been corrected.
The `scripts/T` script (tree view) was the immediate trigger.

## 10. `put` ctl verb does not clear the dirty flag after `addr`/`data` writes

**Discovery (2026-05-22, `scripts/Meld`):** writing `echo put | 9p write
acme/$winid/ctl` after replacing a window's buffer via `addr` + `data`
does **not** mark the window clean, even though the same command works
fine as a standalone tag invocation. A `sleep` before the write makes no
difference, ruling out a timing race in 9P message delivery.

Root cause unknown. `put` likely writes the file to disk but fails to
clear the dirty bit in this context; investigation would require acid
tracing through `xfid.c:xfidctlwrite` → `winsave` → `winclean`.

**Workaround:** use `echo clean | 9p write acme/$id/ctl` after the
`data` write. This clears the dirty flag (via `filereset`) at the cost of
wiping the undo/redo history. The cost is acceptable when the script is
performing a deliberate full-content replacement (e.g. a post-meld sync)
where the pre-replacement undo stack is no longer meaningful. Document the
intent explicitly in the script, as `scripts/T` does.

**Impact on SKILL.md:** the statement "If a script genuinely needs to
make a window clean, the right action is `put`" is partially wrong. It
holds for standalone `put` invocations but not for the replace-then-clean
pattern. `llm/acme/SKILL.md` has been corrected (§ Hard rules).

## Recommended order of action

1. **High impact, low effort**: fix `man/man4/acme.4` per §1.1, §1.3,
   §1.4. Update `src/cmd/acme/codebase_analysis.md` per §2. (Half a day
   total.)
2. **Medium effort, observable user benefit**: align `nctl` font field
   quoting with `ctl` (§1.2) and patch the four scripts that depend on
   it (§3).
3. **Cleanup**: §5 (CLAUDE.md cross-references) and §6 (French
   directive placement). Tiny edits.
4. **Workflow improvement**: §4 (acid `syms` target).
5. **Planned migration**: §7 (Emph built-ins → scripts) — already
   tracked in `llm/TODO`.
6. **Hygiene**: §8 (doc proliferation) — whenever convenient.
