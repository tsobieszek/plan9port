---
name: acme
description: Drive a live acme(1) editor and write acme rc scripts via its 9P filesystem at /mnt/acme. Use when scripting acme, listing or filtering windows, changing fonts, sizes, emphasis or colors, intercepting button-2/3 events, routing plumber messages, or reading/writing window content. Two absolute rules: never write `dirty` or `clean` to a ctl file, and never mutate the body of a dirty window.
---

# acme: scripting and driving a live instance

Acme exposes everything it knows about its world as a **9P filesystem** mounted
at `/mnt/acme`. Reading or writing a file there *is* the API; there is no
other. Acme is glue: it links programs and tools through that filesystem rather
than embedding their behaviors. This skill covers two related tasks:

1. **Driving a running acme** from a script (rc) or one-liner via `9p read` /
   `9p write` on `/mnt/acme`.
2. **Writing new rc scripts** that live in `$HOME/acme/` (or anywhere on
   `$PATH`) and are invoked from acme's tag with a middle-click (button 2).

The authoritative references are the man pages:

- `9 man 1 acme` — user-facing commands, mouse model, chords.
- `9 man 4 acme` — every virtual file, every ctl/nctl verb, and the event
  protocol.

Read them when in doubt. What follows summarises the parts a scripter touches
most. The example scripts that ship in the sibling `scripts/` directory
(`A`, `A+`, `A-`, `F`, `F+`, `F-`, `Bold`, `BoldAll`, `Color`, `ColorAll`,
`EmphAs`) demonstrate every idiom listed below; they are the best starting
templates.

## Hard rules

These come up often enough to be lifted out of the rest.

- **Never write `dirty` or `clean` to a ctl file.** Both verbs exist in the
  protocol; they are reserved for the program that *owns* a window via 9P
  (e.g. `win`, `mail/`) and uses them to project its own notion of
  dirtiness. From an outside script they desynchronise acme's flag from the
  buffer's real state. Concretely, writing `clean` calls `filereset` in
  `file.c` — it **wipes the window's undo and redo history** and clears the
  modified bit, so the user loses both their Ctrl-Z stack and the visual
  warning that the buffer is unsaved. Writing `dirty` does the opposite:
  it marks a clean buffer as modified, so `Del` refuses on the first try
  and the user has no real change to save. Acme manages this flag itself.
  If a script genuinely needs to make a window clean, the right action is
  `put` (which writes the file *and* clears the flag honestly).
- **Never mutate the body of a dirty window.** Writing to `body`, `data`, or
  running any `Edit` command against a window whose dirty flag is set risks
  discarding the user's unsaved work. Before any mutating write, check the
  flag and skip dirty windows (or warn — never proceed silently).
- **Treat windows as ephemeral.** A user can `Del` a window between two of
  your `9p` calls. Empty reads and write errors are expected; redirect
  stderr to `/dev/null` (the example scripts do, with `>[2]/dev/null`) for
  non-essential writes so they fail quietly.
- **Default to read-only inspection.** Inventory before mutation. Even a
  benign-looking `font` ctl write changes line heights and can scroll the
  user's view; ask first if the intent is unclear.

## Checking dirtiness

The dirty flag is the **5th whitespace-separated field** of either:

- `9p read acme/$winid/ctl` (single line, no trailing newline), or
- the matching line of `9p read acme/index` (one line per window, keyed by
  the 1st field, the id).

`0` is clean, `1` is dirty. Idiomatic guard in rc:

```rc
isdirty=`{9p read acme/$winid/ctl | awk '{print $5}'}
if (! ~ $isdirty 0) {
    echo 'window is dirty; refusing' >[1=2]
    exit dirty
}
```

For multi-window scripts, filter `acme/index`:

```rc
for(line in `{9p read acme/index}) {
    id=`{echo $line | awk '{print $1}'}
    isdirty=`{echo $line | awk '{print $5}'}
    if (~ $isdirty 0) {
        ... act on $id ...
    }
}
```

## Mental model

| Concept | Where it lives |
|---|---|
| Global config (fonts), all-windows index, log of events | `acme/cons`, `acme/ctl`, `acme/index`, `acme/log`, `acme/new/` |
| Per-window state | `acme/<id>/…` (where `<id>` is the integer window ID) |
| Random access into a window's body | write `acme/<id>/addr`, then read/write `acme/<id>/data` or `xdata` |
| Built-in commands (`Edit`, `Put`, `Look`, `Emph`, …) | a verb in `ctl`, or executed by a button-2 click in the tag |
| Inter-app messages ("click to open", B3 navigation) | the separate plumber daemon — `9p` via `plumb/send` |
| External event-driven control of a window | open `acme/<id>/event`; messages deferred to the reader |

Inside a command spawned by acme, useful environment variables are pre-set:

- `$winid` — integer id of the window that launched the command.
- `$samfile` and `$%` — file name shown in that window's tag.
- `$acmeaddr` — for 2-1 chords, a fully-qualified button-3 style address of
  the chorded argument.
- `$font`, `$emphfont` — current default body / emphasis fonts.
- `$acmeshell` — shell used to run B2 commands (default `rc`).
- `$tabstop`, `$PLAN9`, `$home`.

The local emphasis feature adds (per-window) the `nctl` file with the verbs
`emph=`, `noemph`, `emphfont`, `emphcolor` (see acme(4) and CODE_CHANGES.md).
**Emphasis-related verbs go to `nctl`, never to `ctl`.**

## File catalogue

### Per-window (`acme/<id>/…`)

| File | Read | Write |
|---|---|---|
| `tag` | tag text | append (file offset ignored) |
| `body` | body text | append (file offset ignored) |
| `addr` | the current addr as **two fixed-width 11-char rune offsets `q0 q1`** (not `#m,#n` — the man page is wrong; verify in `xfid.c:391`) | set address — line numbers, regex, sam-style compound addresses |
| `data` | text from addr onward, count-bounded; **advances addr to the end of what was read** | replace text at addr with the bytes written; addr advances likewise |
| `xdata` | as `data` but reads stop at the end of `addr` and do not advance it | same |
| `ctl` | a single line, no trailing newline: 10 fields — `id ntag nbody isdir isdirty width font tabwidth undo redo`. First 5 are fixed-width `%11d ` (so byte offsets 0, 12, 24, 36, 48); the font field is Plan 9 `%q`-quoted (wrapped in single quotes if it contains whitespace or specials) | one verb per line (see below) |
| `nctl` | 2 fields: `<emphfont> <emphcolor>` — font name is **unquoted `%s`** (an awk `$1` split breaks on multi-word fontsrv paths), color is six lowercase hex digits | `emph=`, `noemph`, `emphfont [path]`, `emphcolor [hex]` |
| `event` | events stream; while open, B2/B3 are deferred to the reader | write back a message to apply it |
| `errors` | — | append to `dir/+Errors` |
| `editout` | — | output of `Edit` commands |

### Global (`acme/…`)

| File | Use |
|---|---|
| `cons` | shared stdout/stderr of acme-spawned commands |
| `ctl` | reads two lines (varfont, fixfont); write `font path` / `varfont path` to change defaults |
| `index` | one line per window: 5 ints (id ntag nbody isdir isdirty) + tag |
| `log` | streaming event log: `<id> <op> <name>` for `new`, `zerox`, `get`, `put`, `del` |
| `new/…` | open a file here to create a new window; e.g. write to `new/body` |

Two precision points (from acme(4)):

- The 5 integers of `index` (and the first 5 of `ctl`) are formatted as
  **fixed-width 11-character fields followed by one blank**, so they sit
  at byte offsets 0, 12, 24, 36, 48; the tag starts at offset 60. `awk`
  on whitespace works fine, but if a script needs byte-precise parsing
  these are the offsets.
- **All offsets are in characters (runes), not bytes.** `addr` returns
  `#m,#n` where `m` and `n` count runes; `data` reads and writes whole
  runes only. Multi-byte UTF-8 inside the buffer never counts twice.

## ctl verbs (per-window)

One verb per line written to `acme/<id>/ctl`. Each is the script-level
equivalent of a user command.

Documented in `acme(4)`:

`addr=dot` — set addr to the user's selection ·
`dot=addr` — set the selection to addr ·
`cleartag` — remove text after the bar in the tag ·
`del` — `Del` (refuses if dirty) ·
`delete` — `Delete` (no dirtiness check) ·
`dump cmd` / `dumpdir dir` — set dump information ·
`font path` — set the body font ·
`get` — `Get` with no argument (refuses if dirty) ·
`limit=addr` — restrict subsequent regex context to `addr` ·
`mark` / `nomark` — undo grouping ·
`name X` — rename the window ·
`put` — `Put` ·
`show` — scroll to make the selection visible.

Implemented in `xfid.c:xfidctlwrite` but **not in the man page**:

`lock` — take exclusive control of the `ctl` file (other writers block on
`w->ctllock` until released) ·
`unlock` — release it ·
`menu` / `nomenu` — re-enable or suppress the automatic command menu in the
tag (the words to the left of the bar).

**Not these, ever:** `clean`, `dirty` — see Hard rules.

## nctl verbs (per-window emphasis)

`emph=REGEX` — activate emphasis on this sam-style regex ·
`noemph` — deactivate (regex retained; re-writing the same `emph=` reactivates) ·
`emphfont path` — pin a specific font for emphasis on this window ·
`emphfont` — toggle between matching the body font and using a distinct emphasis font ·
`emphcolor HEX` — 3 or 6 hex digits; `82f` → `8822ff` ·
`emphcolor` — reset to the global default set by `-C`.

Reading `nctl` returns one line: `<emphfont> <emphcolor>` (color as 6 hex
digits).

## Common idioms

### Listing, inspecting

Iterate all windows, skipping dirty:

```rc
for(line in `{9p read acme/index}) {
    id=`{echo $line | awk '{print $1}'}
    isdirty=`{echo $line | awk '{print $5}'}
    if (~ $isdirty 0)
        echo $id
}
```

Read current font of the current window (works only for paths without
whitespace, since `ctl` `%q`-quotes the field):

```rc
font=`{9p read acme/$winid/ctl | awk '{print $7}'}
```

### Writing to a window

Appending is trivial (offset is ignored):

```rc
echo 'hello' | 9p write acme/$winid/body
```

To **replace** existing content you must go through `addr` + `data` (writing to
`body` always appends). The `addr` file accepts any sam-style address:

```rc
# Replace the whole body with the output of `date`
echo , | 9p write acme/$winid/addr           # whole-file selection
date | 9p write acme/$winid/data
```

```rc
# Replace lines 3..5 in the current window
echo '3,5' | 9p write acme/$winid/addr
echo 'replacement text' | 9p write acme/$winid/data
```

Compact `addr` syntax cheatsheet (rune offsets, not bytes):

```
0           start of file
$           end of file
0,$         entire body
123         line 123
123,456     lines 123..456
#42         rune offset 42
#42,#100    rune range
/regex/     next match forward from current dot
-/regex/    previous match backward
.           current selection
.,+#10      from dot, ten runes to its right
.+/regex/   from dot, next match forward
```

(Sam compound addresses are valid in both `addr` and `Edit`; see
`9 man 1 sam` for the full grammar.)

### Driving sam-style Edit commands

`Edit` is a built-in *user* command, **not** a ctl verb. Three ways to drive
it from outside:

1. **`addr` + `data` directly** (cleanest; example just above). Use for
   anything you can express as an address followed by a replacement.
2. **Append the command to the tag and have the user click it** (when the
   script is meant to *suggest* an action).
3. **Through the event protocol** (rare; preferred only for long-running
   programmatic controllers — use `libacme` from C or `9fans.net/go/acme`).

The sam language understood by `Edit` (`9 man 1 sam`):

- Addresses: `:13`, `:0`, `:$`, `:,` or `:1,$`, `:/regex/`, `:-/regex/`
  (backwards), `:+/regex/`, `addr1,addr2`, `addr1;addr2`.
- Commands: `a/text/` append, `i/text/` insert, `c/text/` change, `d` delete,
  `s/pat/rep/g` substitute, `p` print, `=` line number, `x/re/ cmd` loop on
  matches, `g/re/ cmd` guard, `v/re/ cmd` complement-guard.
- Shell hooks on selection: `Edit ,< cmd` replace by stdout of cmd;
  `Edit ,> cmd` send to stdin; `Edit ,| cmd` pipe through.
- Nested blocks:

  ```
  Edit ,x/TODO/ {
      i/(
      a/)
  }
  ```

Quick one-offs (typed in the tag and middle-clicked):

- `Edit ,d` — clear the window body.
- `Edit , < cat file.txt` — load a file into the body.
- `Edit , | sort` — sort lines.
- `Edit =` — print the current line number into `+Errors`.

### Creating windows

```rc
# A blank window:
9p read acme/new/ctl >/dev/null

# A window pre-populated with text:
echo 'Hello, world' | 9p write acme/new/body

# A window labelled with a name (rename right after creating):
id=`{9p read acme/new/ctl | awk 'NR==1{print $1}'}
echo 'name /tmp/scratch' | 9p write acme/$id/ctl
```

### Watching activity

```rc
# Blocks until next event; loop to tail
while () {
    line=`{9p read acme/log}
    echo $line
}
```

Lines look like `<id> <op> <name>` with op ∈ `new`, `zerox`, `get`, `put`, `del`.

### Per-window event stream (the `event` protocol)

```rc
9p read acme/$winid/event
```

**As soon as `event` is open, button-2 and button-3 actions on that window
are deferred to the reader** — they no longer have an immediate effect. The
reader is expected to handle them (or to write them back unchanged so acme
applies them). Two operational consequences:

- **Unread events block acme.** If a controller opens `event` and stops
  reading, the user's mouse clicks queue up and the window appears frozen.
  Never open `event` without an active reader loop, and close it when done.
- **A normal `Del` does not close `event`.** A controller that owns the
  event file must handle the `Dx`/`Dx` chord (or the writeback of a `Dx`
  message) itself.

Each event is one line, ending with `\n`:

```
origin type q0 q1 flag nr [text]
```

- **origin** (1 char): `E` write to body/tag from outside, `F` action via
  another window file, `K` keyboard, `M` mouse.
- **type** (1 char): `D` body delete · `d` tag delete · `I` body insert ·
  `i` tag insert · `L` button-3 in body · `l` button-3 in tag ·
  `R` shift-button-3 in body · `r` shift-button-3 in tag · `X` button-2 in
  body · `x` button-2 in tag.
- **q0 q1** (decimals): character (rune) addresses of the action.
- **flag** (decimal, bitwise OR):
  - For `X` / `x`: `1` if the indicated text is a recognised acme built-in,
    `2` if expansion follows (a second event will give the expanded text),
    `8` if a chorded extra argument follows (two more events: the argument
    text, then its origin address).
  - For `L` / `l`: `1` if acme can handle the action without loading a new
    file, `2` if expansion follows, `4` if the text is a file or window
    name (possibly with address) rather than literal text.
  - For `D`/`d`/`I`/`i`: always `0`.
- **nr** (decimal): number of characters in the optional text. If the
  payload would be larger than 256 characters the field is `0`; the text is
  elided and must be fetched from `data` using the q0/q1 address.
- **text**: free-form, may contain newlines (the parser uses `nr`, not
  `\n`, as the boundary).

**Acknowledging events.** For events whose `flag & 1` is set (i.e., acme
*could* have handled it itself), writing the message *back* to `event` —
with the flag, count, and text fields stripped — applies the action as if
the event file had not been open:

```
origin type q0 q1
```

This is the standard idiom for an event-driven controller: read an event,
decide whether to intercept it, and on accept ack it back to acme.

For non-trivial controllers prefer a higher-level library that parses all
of this for you: C `libacme` (`include/acme.h`) or Go `9fans.net/go/acme`.
The rc-level interface above is fine for quick filters but quickly turns
unwieldy.

### Plumber

Send a "click to open" message from a script:

```rc
plumb -d edit foo.c:42
```

Reload rules after editing `$home/lib/plumbing`:

```rc
cat $home/lib/plumbing | 9p write plumb/rules
```

See `9 man 7 plumb` (format) and `9 man 4 plumber` (file server).

## Writing rc scripts for acme

Place scripts in `$HOME/acme/` (or anywhere on `$PATH`); they appear as
button-2 commands typed in any tag. Conventions used by every script in this
directory:

- Shebang `#!/usr/local/plan9/bin/rc`.
- Read configuration from the **environment**, never from a config file:
  `$font`, `$emphfont`, `$winid`, `$samfile`, `$color`, `$varcolor`,
  `$antialias_threshold`.
- Per-window scripts target `acme/$winid/...`; whole-instance scripts iterate
  `9p read acme/index` and call a helper function per id.
- Extract fields from `ctl`/`nctl`/`index` with `awk`. The font field of
  `ctl` is column 7; of `nctl` it is column 1; the dirty flag is column 5 of
  both `ctl` (first line) and each `index` line.
- Suppress noisy stderr from `9p write` on best-effort writes:
  `... | 9p write $ctlfile >[2]/dev/null`.
- Functions stay short. Compose with `awk`, `sed`, `grep`, `expr`.
- Plan 9 `rc` syntax: `if (cond) ...` / `if not ...`, ``` `{cmd} ``` for
  command substitution, `for(x in list)` for loops, `~ a b` for string
  equality, `! ~` for inequality, `$#var` for list length, `''` for the
  empty string.
- No emojis. Program messages in English. Comments sparing; docstrings if a
  function's purpose is non-obvious.

### Skeleton

```rc
#!/usr/local/plan9/bin/rc

# Argument validation
if (~ $1 '') {
    echo 'usage: ScriptName arg' >[1=2]
    exit usage
}

# Per-window context
ctlfile=acme/$winid/ctl

# Read state; refuse on dirty
ctl=`{9p read $ctlfile}
isdirty=`{echo $ctl | awk 'NR==1{print $5}'}
if (! ~ $isdirty 0) {
    echo 'window is dirty; refusing' >[1=2]
    exit dirty
}

# Act
echo 'name newname' | 9p write $ctlfile >[2]/dev/null
```

### fontsrv assumption in the `A*`/`F*`/`Bold*` scripts

These scripts assume fonts come from `fontsrv` (mounted at `/mnt/font`) with
the layout `/mnt/font/.../Family/SIZE/font`. They parse field 4 (family) and
field 5 (size) of the path. Bitmap fonts and other layouts will not work.
They honour `$antialias_threshold` (default 16): sizes ≥ it get the `a`
suffix.

## Built-in commands worth remembering

User commands (button-2 on a word in the tag) — full list in `9 man 1 acme`:

`Cut`, `Paste`, `Snarf`, `Undo`, `Redo`, `Look`, `Edit`, `Get`, `Put`,
`Putall`, `Del`, `Delete`, `Delcol`, `New`, `Newcol`, `Send`, `Sort`, `Tab`,
`Zerox`, `Dump`, `Load`, `Exit`, `Font`, `Indent`, `Incl`, `Kill`.

Emphasis (local to this tree, see `man/man1/acme.1`):

`Emph [regex]`, `EmphMe`, `EmphAll`, `EmphNone`, `EmphFont [path]`,
`AutoEmph`. The rc script `EmphAs <lang>` extends this by looking up
`$PLAN9/lib/emph.regexp` patterns by language code (`sh`, `py`, `c`, `go`,
…).

Shell I/O on selection (prefix the command in the tag):

- `< cmd` — replace selection with stdout of cmd.
- `> cmd` — send selection to stdin.
- `| cmd` — pipe selection through cmd.

Copy/cut selection to a file (idiomatic):

```
| sed '' > file.txt   # cut: pipe through sed and redirect
> sed '' > file.txt   # copy: send-to-stdin variant
```

## Debugging acme itself with acid

Not for scripting; for investigating bugs inside the editor's C source.
Plan9port ships `acid` and `acidtypes`, but **this tree's
`src/cmd/acme/mkfile` has no `syms` / `acme.acid` targets** (the example
recipes from upstream Plan 9 docs do not apply as-is). Procedure here:

```
9 acidtypes $PLAN9/bin/acme >acme.acid     # generate struct definitions
9 acid -l acme.acid <pid>                  # attach
```

Once attached:

- `bpset(funcname)` — break on a function.
- `bpset(filepc("/path/file.c:NNN"))` — break at a source line.
- `bptab()` — list breakpoints.
- `cont()` — continue. `next()` / `step()` — single-step.
- `src(*PC)` / `Bsrc(*PC)` — show source at PC (`Bsrc` opens it in an acme
  window).
- `(StructType)var` — cast and dump a struct. Scope a variable to a function
  with `func:var`.

Most acme bugs reproduce best by running a second acme inside the first via
`9 win acme`. See `9 man 1 acid` and `9 man 1 acidtypes` for the full
language. (Plan9port's acid is the same engine as Plan 9's; only the
symbols-generation workflow differs from the `mk syms` recipe found
online.)

## Auto-syncing acme with Claude Code edits

This skill ships a PostToolUse hook so that an acme session stays in sync
with what Claude has changed on disk. Installed by `cd llm && mk install`
(under this tree); the moving parts are:

| Path | Role |
|---|---|
| `~/.claude/skills/acme/bin/acme-update` | rc script. Hook mode reads the event JSON on stdin; manual mode takes `FILE [TOOL]`. |
| `~/.claude/skills/acme/bin/acme-sync-recent` | Iterates `git diff --name-only HEAD` + untracked files, calls `acme-update` on each. Backbone of the slash command. |
| `~/.claude/commands/update-acme.md` | Slash command `/update-acme` — runs the manual driver. |
| `~/.claude/settings.json` | Carries the `PostToolUse` matcher for `Edit\|Write\|MultiEdit\|NotebookEdit` → `acme-update`, written by `install-hook.rc`. |

Per-file behaviour, in priority order (see `bin/acme-update`):

1. Acme not reachable, or `jq` missing — silent no-op (exit 0).
2. Existing window with this file (tag's first whitespace-separated token
   matches the file path) and `isdirty == 0` — write `get`, then a
   best-effort `dot=addr` to the line that contains the first non-blank
   line of `new_string`, then `show`.
3. Existing window with `isdirty == 1` — write a `[acme-update] <path>
   was edited by Claude but the acme window is dirty; not reloaded.`
   line to `acme/cons`, which surfaces in `+Errors`. The buffer is **not
   touched**, per the project's hard rule.
4. No matching window, file exists on disk, tool was `Write` — create a
   window via `acme/new/ctl`, rename it (`name PATH`), `get`, then move
   dot to the first line of the new content. Edits/MultiEdits on
   non-open files leave them alone (Claude can touch files in the
   background; the hook never forces them onto the user's screen).

Best-effort selection is line-based: `grep -n -F -m1 <first-line> <path>`
on the file as it now exists on disk; if that fails (e.g. the first
non-blank line is a generic `}` that matches many lines), the address
falls back to line 1. The user's requirement allowed approximate
matches.

### Slash command — manual driver

`/update-acme` runs `acme-sync-recent`, which iterates over
`git diff --name-only HEAD` plus untracked files, calling `acme-update`
for each. Use it when the hook is disabled, when a batch of changes
piled up before the hook was in place, or to recover after the user
manually closed and reopened acme.

### Disabling or reinstalling

```
cd $PLAN9/llm
mk uninstall       # removes everything, including the settings entry
mk install         # reinstalls
```

The settings merger backs up `~/.claude/settings.json` to
`~/.claude/settings.json.bak` before each modification and refuses to
overwrite the live file if the merge result is not valid JSON.

## Conventions for this codebase

- Reply to the user in **French**. Code, identifiers, file content, error
  messages, comments: **English**.
- No emojis anywhere.
- Functional, short rc functions. One responsibility each. Compose.
- Match the surrounding code style when editing a script.
- When adding or removing a script in `scripts/`, update both `SCRIPTS=` in
  `scripts/mkfile` and the tables in `scripts/README.md`.
- The man pages — `9 man 1 acme`, `9 man 4 acme`, `9 man 7 plumb`,
  `9 man 1 sam` — are the source of truth for any verb, address, or message
  format. Consult them before guessing.

## Pointers to companion docs

- `scripts/README.md` — every shipped script, with usage tables.
- `src/cmd/acme/CLAUDE.md` — guidance when editing acme's C source.
- `src/cmd/acme/codebase_analysis.md` — internals: data stack, ctl/nctl
  dispatch, emphasis pipeline.
- `CODE_CHANGES.md` (tree root) — local divergence from upstream plan9port
  (emphasis feature).
- `lib/emph.regexp` — language → pattern mapping used by `EmphMe`,
  `EmphAll`, `AutoEmph`, and `EmphAs`.
