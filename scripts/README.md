# acme font/emphasis scripts

A set of `rc` scripts driven from the acme tag (or invoked with a
middle-click argument). They manipulate, through the 9P `ctl`/`nctl`
files, the font, size, weight and color of acme windows — including the
locally-added *emphasis* feature (see `../CODE_CHANGES.md`).

## Important: fontsrv fonts only

These scripts only work with fonts served by `fontsrv` and mounted at
`/mnt/font`:

```
fontsrv -m /mnt/font
```

They parse the font *path* by hand, assuming the fontsrv layout
`/mnt/font/.../Family/SIZE/font`: the family name is taken from path
field 4 and the pixel size from field 5, and the `Bold` scripts probe
for sibling files such as `-Bold`, `-Regular` and `-Medium`. They will
*not* work with the built-in Plan 9 bitmap fonts or other font paths.

The size scripts also honour `$antialias_threshold` (default 16): any
size greater than or equal to it gets the `a` (antialiased) suffix.

## Built-in commands

Distinct from the scripts below, this extended version of acme adds these
commands *built into the editor itself* (see acme(1)):

| Command | Effect |
|---------|--------|
| `Emph`     | Set this window's emphasis pattern to the selected text (`Emph` with no selection clears it) |
| `EmphMe`   | Apply the `lib/emph.regexp` pattern matching this window's file extension |
| `EmphAs`   | Apply the `lib/emph.regexp` pattern for a specific language (external script: `EmphAs sh`, `EmphAs py`, etc.) |
| `EmphAll`  | Apply the `EmphMe` logic to every non-scratch window |
| `EmphNone` | Clear emphasis from all windows |
| `EmphFont` | Pin the emphasis font for this window (no argument: reset to automatic) |
| `AutoEmph` | Toggle the global auto-emphasis flag (emphasise files on open) |

## Scripts

The `A`/`Color`/`BoldAll` family acts on **all** windows; the
`F`/`Color`/`Bold` family acts on the **current** window only.

### Size — current window

| Script | Without argument | With argument `N` |
|--------|------------------|-------------------|
| `F`  | Reset size to the size in `$font` | Set size to `N` |
| `F+` | Increase size by 1 | Increase size by `N` |
| `F-` | Decrease size by 1 | Decrease size by `N` |

### Size — all windows

| Script | Without argument | With argument `N` |
|--------|------------------|-------------------|
| `A`  | Reset size to the size in `$font` | Set size to `N` |
| `A+` | Increase size by 1 | Increase size by `N` |
| `A-` | Decrease size by 1 | Decrease size by `N` |

All six adjust both the regular font and the emphasis font of the
affected window(s).

### Weight (emphasis font)

| Script | Without argument | With argument |
|--------|------------------|---------------|
| `Bold`     | Toggle the current window's emphasis font between bold and non-bold | (argument ignored) |
| `BoldAll`  | Toggle every window, using the current window's emphasis font to pick the direction | `bold` or `plain` forces that state on every window |

### Language pattern (emphasis pattern by language)

| Script | Argument | Effect |
|--------|----------|--------|
| `EmphAs` | **required** language code (e.g., `sh`, `py`, `c`, `go`) | Apply the emphasis pattern for the specified language from `lib/emph.regexp` to the current window. Supported: `flix`, `eff`, `jl`, `c`, `h`, `py`, `rs`, `go`, `js`, `md`, `sh`, `rc`, `hs`, `lua`, `scala`, `cpp`, `glsl`, `vert`, `frag`, `gd`, `tscn`, `godot`, `toml`, `desktop`, `kt`, `xml`, and more. |

### Color (emphasis color)

| Script | Without argument | With argument |
|--------|------------------|---------------|
| `Color`     | Toggle the current window's emphasis color between `$color` and `$varcolor` | Use the argument as the color (hex; 3-char shorthand such as `1af` is expanded to `11aaff`) |
| `ColorAll`  | Same toggle, applied to every window | Same, applied to every window |

### Diff against disk

| Script | Effect |
|--------|--------|
| `Meld` | Copy the current window's body to a temporary file and open `meld` on that file and `$samfile` (the on-disk version). Waits until meld exits, then deletes the temporary file. Most useful on dirty windows to review unsaved changes before saving. |

### Tree view

| Script | Without argument | With argument `dir` |
|--------|------------------|---------------------|
| `T` | Show a tree of the current window's directory | Show a tree of `dir` |

`T` finds or creates a window named `<dir>/`, replaces its body with
`tree -F --prune --filesfirst --noreport -fnNF --compress=2 <dir>`,
then marks the window clean. If the window already exists it is
refreshed in place.

## Installation

```
mk install
```

installs the scripts into `$HOME/acme/`. This target is intentionally
standalone: it is not reached by `mk install` or `./INSTALL` at the
tree root.

## See also

The companion Claude Code skills under `../llm/`:

- `../llm/acme/SKILL.md` documents the full 9P control surface
  (`/mnt/acme`), every `ctl`/`nctl` verb, the `event` protocol, and
  the rules these scripts follow (never write `dirty`/`clean` to
  `ctl`; never mutate a dirty window).
- `../llm/rc/SKILL.md` documents the `rc` shell itself and the
  plan9port flavours of `test`/`awk`/`sed`/`grep`/`regexp(7)` that
  these scripts compose, plus the bash-vs-rc cheat sheet and the
  pitfalls a Bourne reflex produces.

Install both with `cd ../llm && mk install`.
