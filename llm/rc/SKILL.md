---
name: rc
description: Write rc scripts and one-liners for plan9port. Covers Plan 9 rc shell syntax (lists, ~ match, free carets, `{}` substitution, `>[2]` redirections), p9p flavours of test/sed/awk/grep/tr (regexp(7), not POSIX ERE/BRE), portable shebang `#!/usr/local/plan9/bin/rc`, and the idioms/footguns that bite when the script driver has bash habits. Companion to acme/SKILL.md for scripts that talk to a live acme over /mnt/acme.
---

# rc: scripting in the Plan 9 shell on plan9port

`rc` is the Plan 9 shell. Plan9port ships it as `$PLAN9/bin/rc` and uses
it as the language for every glue script in the tree (acme tag scripts,
`mk` recipes when `MKSHELL` selects it, the system's `lib/profile`,
the `bin/9.rc` wrapper). Its syntax and semantics are **not** a dialect
of `sh`/`bash`. Treat it as a different language with a superficially
similar look — the gotchas listed below are mostly cases where a Bourne
reflex produces silently-wrong rc.

The authoritative references are the man pages. Consult them; this
document summarises and ranks what trips up a script driver in practice.

- `9 man 1 rc` — grammar, control flow, built-ins, variables, redirection.
- `9 man 1 test`, `1 sed`, `1 awk`, `1 grep`, `1 tr`, `1 wc`, `1 mc`,
  `1 seq`, `1 cat`, `1 mk`, `1 9p`, `1 plumb`, `1 9`, `1 9term`,
  `1 acmeevent`.
- `9 man 7 regexp` — the regex flavour used by every Plan 9 tool
  (grep, sed, awk, libregexp). It is **not** POSIX BRE/ERE.

There is no `9 man -k` in plan9port (the wrapper just prints `man`'s
usage); to find a page, `ls $PLAN9/man/manN | grep something` or read
`man/man1/0intro.1`.

The script catalogue in `scripts/` (`A`, `A+`, `A-`, `F`, `F+`, `F-`,
`Bold`, `BoldAll`, `Color`, `ColorAll`, `EmphAs`) is the canonical worked
example for every idiom below. Read those first when starting a new
script.

## Hard rules

These are the surprises that produce silently-broken scripts. They come
up often enough to be lifted out.

- **`$status` is a string, not a number.** Empty (`''`) is success. Any
  non-empty string is failure (it is the wait-message of the last
  command, e.g. `error`, `usage`, `127`). Test with `~ $status ''` or
  let `&&` / `||` / `if` do it for you. The conventional success form is
  `exit` (no argument); for failures use a short descriptive label such
  as `exit usage` or `exit dirty` — the string surfaces in `$status` of
  the caller and in `-x` traces. (`exit 0` does work on plan9port —
  the underlying Unix exit code is parsed from the string — but
  arbitrary tags like `exit usage` are far more informative.)
- **`~` mutates `$status`.** Using `~ $status ''` to *inspect* status
  clobbers it. Save first: `s=$status; if (~ $s '') ...`. This is
  documented under BUGS in `rc(1)` and is the single most common rc
  scripting bug.
- **Variables hold lists, not strings.** `x=(a b c)` is a 3-element
  list. `$#x` is `3`. `$x` expands to 3 separate arguments. Use
  `$"x` to get a single space-joined string. A scalar is just a
  1-element list; the empty string `x=''` is still a 1-element list
  (`$#x == 1`); the empty list `x=()` has zero elements (`$#x == 0`)
  but both expand to nothing in argument position — `~ $x ''` cannot
  tell them apart.
- **No `local`.** All variable assignments inside `fn` are global and
  outlast the call. Scope by giving the function a unique name prefix,
  or wrap mutations in `@{ ... }` to run them in a subshell.
- **Heredocs do not survive inside `{}` blocks or function bodies.**
  `if (cond) { cat <<EOF ... EOF }` parses the heredoc body as
  commands. Either move the heredoc to a top-level statement,
  drop the braces (`if (cond) cat <<EOF`), or replace it with a series
  of `echo` lines inside the function. `rc(1)` BUGS lists this
  explicitly: "Functions that use here documents don't work."
- **`[ … ]` is not the test command in rc.** Use `test …` (no brackets).
  And use `~ var pat` rather than `test "$var" = "literal"`: faster,
  shorter, no quoting hazard, and equality of strings is what `~` was
  built for.
- **Double quotes are *not* a quoting mechanism in rc.** `"hello"`
  is a literal string of seven characters including the two quote
  marks. The only quote is the single quote `'…'` (a doubled `''`
  inside denotes one literal quote). Metacharacters that must be
  single-quoted to appear literally in an argument position: `# ; & |
  ^ $ ' { } ( ) < >`, the backtick, and whitespace.

## Plan 9 cheat sheet vs bash/POSIX

| Concept | bash / POSIX | rc |
|---|---|---|
| Exit status | `$?` (integer; 0 = success) | `$status` (string; `''` = success) |
| Command substitution | `$(cmd)` or `` `cmd` `` | `` `{cmd} `` (returns a list, split on `$ifs`) |
| Variable expansion | `"$x"` | `$x` (lists already; no word-splitting hazard) |
| String length | `${#x}` (string), `${#a[@]}` (array) | `$#x` (list element count) |
| Array/list | `arr=(a b c)` / `${arr[@]}` | `arr=(a b c)` / `$arr` |
| If/else | `if cond; then …; else …; fi` | `if (cond) …` / `if not …` |
| For | `for i in …; do …; done` | `for(i in …) …` (no `do`/`done`) |
| While | `while cond; do …; done` | `while (cond) …` |
| Case | `case x in pat) … ;; esac` | `switch(x){ case pat …  }` |
| Function | `f() { … }` | `fn f { … }` |
| String equality | `[ "$x" = y ]` | `~ $x y` |
| Pattern match | `[[ $x = a* ]]` | `~ $x 'a*'` |
| Redirect stderr | `2>/dev/null` | `>[2]/dev/null` |
| Redirect stderr→stdout | `2>&1` | `>[2=1]` |
| Redirect stdout→stderr | `1>&2` | `>[1=2]` |
| Close a descriptor | `>&-` | `>[2=]` |
| Process substitution | `<(cmd)` (bash) | `<{cmd}` (rc) |
| Concatenate string + var | `"a$x"` or `a$x` | `'a'^$x` (or `a$x` — see free carets) |
| Boolean negate | `! cmd` | `! cmd` (same) — and `! ~ var pat` |
| Read 1 line | `read line` (shell builtin) | ``line=`''{read}`` (rc has no `read` builtin; use p9p `read(1)`, with empty IFS so spaces stay in `$line`) |
| Test integer | `[ $n -gt 5 ]` | `test $n -gt 5` (same primitives) |
| ERE/BRE | POSIX BRE in `grep`, ERE in `grep -E` | `regexp(7)` only — no `{n,m}`, no `\d`, no POSIX classes |

## The rc language

### Variables are lists

Every variable is a list of strings.

```rc
x=(a b c)
echo $#x         # 3
echo $x          # a b c        (three arguments to echo)
echo $"x         # 'a b c'      (single argument, space-joined)
echo $x(2)       # b            (1-based subscript)
echo $x(2-3)     # b c          (slice)
y=()             # empty list
y=''             # 1-element list containing the empty string — NOT the same
echo $#y         # 1 (after y='')  vs  0 (after y=())
```

Assignment is by name without `$`: `x=value`. Reading is `$x`. There is no
`$x` vs `${x}` distinction; just `$x`. Tail of an expression: `$x^...`
or write it adjacent (free carets, below).

### Free carets `^`

`a^b` concatenates lexically. Rc inserts an implicit `^` between any two
arguments not separated by whitespace, so all of these mean the same
thing:

```rc
cmd=$prefix^.c
cmd=$prefix.c
cmd=$prefix'.'c
```

For lists, concatenation distributes pairwise (same length) or against a
single element:

```rc
x=(a b c); y=(1 2 3)
echo $x^$y       # a1 b2 c3
echo $x^z        # az bz cz
echo p^$y        # p1 p2 p3
```

This is the rc way to build strings. Avoid `"$a$b"` reflexes (rc has no
double-quote interpolation; `"` is literal).

### Quoting

Single quotes only. `''` inside a single-quoted string is one literal
quote.

```rc
echo 'it''s fine'      # it's fine
echo 'hello world'     # hello world (one argument)
echo "hello world"     # "hello world" (one argument including the quotes!)
```

Metacharacters that must be single-quoted to appear literally in
argument position: `# ; & | ^ $ { } ( ) < >`, the backtick, the single
quote (doubled), and whitespace. Single-quoting also disables file-name
pattern matching for that word.

### Pattern matching with `~`

`~ subject pat1 pat2 …` matches `subject` against the patterns
(file-glob style: `*`, `?`, `[class]`, `[~class]` for complement) and
sets `$status` to empty (true) on any match.

```rc
if (~ $1 '')                       echo no arg
if (~ $arg -h --help)              echo help requested
if (~ $file *.c *.h)               echo c source
if (! ~ $isdirty 0)                echo dirty
```

This is **the** idiom for string equality, prefix tests, and small
enumerations. Do not reach for `test`.

Patterns in `~` are not file-globbed before matching — they are tested
literally against the subject — so the patterns above need no extra
quoting unless they themselves contain shell metacharacters that rc
would expand before the call.

**Subject is *one* value.** Only the first argument after `~` is the
subject; everything else is patterns. If `$subject` expands to a list
of N elements, the call becomes `~ a b c pat1 pat2 …`, taking `a` as
the subject and `b c pat1 pat2 …` as patterns — which is almost
certainly not what was wanted. Quote-join with `$"x` to collapse a
list into a single subject: `~ $"list pat`.

### Control flow

```rc
if (list)             cmd_or_block
if not                cmd_or_block            # may follow any 'if ()'
for(name in args)     cmd_or_block            # 'in args' optional → $*
for(name)             cmd_or_block            # same as 'for(name in $*)'
while(list)           cmd_or_block
switch(arg){
case patA patB
    cmd
case patC
    cmd
case *                                          # default
    cmd
}
```

There is no `then`, no `do`/`done`, no `fi`, no `;;`, no `else`. The body
is either a single command or a `{…}` block.

`if not` must immediately follow the matching `if(…) …`; there is no
`elif`. Chain explicitly:

```rc
if (~ $x a)       echo apple
if not if (~ $x b) echo banana
if not             echo other
```

### Functions

```rc
fn greet {
    echo hello, $1
}
greet world
```

Arguments arrive in `$*`, `$1`, `$2`, …; `$#*` is the count. Functions
inherit and **mutate** the caller's environment — no `local`. Patterns:

- Prefix internal variable names with the function name (`greet_msg=...`).
- Or wrap state-changing work in a subshell: `@{ msg=…; do_it }`.
- Or save and restore: `saved=$x; …; x=$saved`.

Functions cannot contain heredocs (use `echo` lines instead).

### Substitution

```rc
list=`{cmd ...}             # capture stdout, split on $ifs (default ' \t\n')
list=`split{cmd}            # same, but split on 'split' chars instead
files=`{ls *.c}             # list of *.c
n=`{wc -l <file}            # n is a 1-element list with the number (still a string)
```

`$ifs` controls splitting. To override it for one substitution only,
use the variant form `` `'sep'{cmd} ``, which uses `sep` as the split
characters for this call alone:

```rc
fields=`':'{echo a:b:c}     # 3 elements: a  b  c
oneline=`''{echo a b c}      # 1 element: 'a b c' (empty sep ⇒ no split)
```

The empty-separator form is the canonical way to capture stdin
byte-for-byte (newlines and spaces preserved) — essential when the
payload is JSON or any structured text:

```rc
payload=`''{cat}             # whole stdin in one element, verbatim
data=`''{cat /path/to/file}  # whole file likewise
```

(Both `../bin/acme-update` and `../bin/acme-sync-recent` use this for
stdin JSON capture.)

Idiom for reading line-by-line:

```rc
for(line in `{cmd}) {
    ... process $line ...
}
```

This splits on whitespace by default. To safely split strictly on newlines (e.g. for `git` output where files may contain spaces), use the `sep` variant with a literal newline:

```rc
nl='
'
for(line in `$nl{cmd}) {
    ... process $line ...
}
```

Process substitution: `<{cmd}` and `>{cmd}` evaluate to a `/dev/fd/N`
path. `cmp <{old} <{new}` is the canonical example.

### Redirection

| Form | Meaning |
|---|---|
| `>file` | redirect fd 1 to file (overwrite) |
| `>>file` | append fd 1 |
| `<file` | redirect fd 0 from file |
| `>[2]file` | redirect fd 2 to file |
| `>[2=1]` | duplicate fd 1 onto fd 2 (stderr → stdout) |
| `>[1=2]` | stdout → stderr |
| `>[2=]` | close fd 2 |
| `<<eof` | here document (top-level statements only — see hard rules) |
| `<<<word` | rc has no here-string; pipe `echo` instead: `echo word \| cmd` |
| `\|[2]` | pipe fd 2 instead of fd 1 |

Redirections are evaluated left-to-right. The fd argument goes in square
brackets *attached to the operator*, not as a leading number.

### Built-ins

Most relevant when scripting:

- `~` — pattern match (above).
- `~ subject pattern …` — see above; sets `$status`.
- `.` *file* — source (like `source` in bash).
- `cd [dir]` — change directory.
- `eval arg …` — concatenate and execute as rc input.
- `exec cmd …` — replace this rc with cmd.
- `exit [status]` — exit; argument is a *string*, empty = success.
- `flag x [+|-]` — set/clear/test a one-letter command-line flag.
- `rfork [nNeEsfFm]` — start a new process group (see `fork(2)`).
- `shift [n]` — drop first n of `$*`.
- `wait [pid]` — wait for backgrounded process.
- `whatis name …` — print definition (function, builtin, path).
- `builtin cmd …` — bypass any shadowing `fn` of `cmd`.

### Notable variables

| Name | Meaning |
|---|---|
| `$*` `$1` `$2` … | argument list and elements |
| `$#var` | number of elements in `var` |
| `$"var` | components joined by single spaces, as one string |
| `$status` | wait-message of last command; `''` = ok |
| `$apid` | pid of last `&` background command |
| `$pid` | rc's own pid |
| `$path` `$PATH` | search path (kept in sync; rc uses `$path` as list) |
| `$home` `$HOME` | as on Unix |
| `$ifs` | input field separators for `` `{…} `` (default ` \t\n`) |
| `$prompt` | 2-element list: PS1 and PS2 |
| `$rcname` | name under which rc was invoked (path of running script) |
| `$cdpath` | search path for `cd` |

### Patterns and globbing

Glob characters in unquoted argument position:

| Pattern | Matches |
|---|---|
| `*` | zero or more of any character |
| `?` | one character |
| `[abc]` | one character in set |
| `[a-z]` | one character in range |
| `[~abc]` | one character **not** in set (`~`, not `^`) |

Important difference from sh: the complement marker is `~`, not `^`.

A pattern that matches nothing stands for itself (unlike bash's
`nullglob` default of expanding to nothing). Quote a literal `*` etc. as
`'*'`.

`/` and the leading `.` of `.`/`..` must always be matched literally.

## Plan 9 toolchain quirks worth knowing

Every text tool on plan9port operates on **runes** (UTF-8) and uses the
`regexp(7)` flavour. They are conceptually the same as Unix tools but
the flag sets and regex flavour differ.

### `regexp(7)`

The Plan 9 regex flavour used by `grep`, `sed`, `awk`, `libregexp`, sam,
acme's `Edit`. It is the ed/egrep style, simpler than POSIX ERE.

Supported: `.` (any rune except newline), `*`, `+`, `?`, `[…]`, `[^…]`,
`(…)`, `|`, `^`, `$`, `\c` (literal `c`).

**Not supported**: `{n,m}` quantifiers, `\d`/`\w`/`\s` shorthand,
`[[:alpha:]]` POSIX classes, lookahead/lookbehind, named groups, `\b`
word boundaries, multiline modifiers. Inside a character class the
escape rules differ: `-`, `]`, leading `^`, and the delimiter must be
backslash-escaped.

If you need POSIX ERE features, fall back to GNU `grep`/`sed` from
`/bin` instead of `9 grep` — or restructure the pattern.

### `test(1)`

Same primitives as POSIX but with extras. Plan 9 niceties:

- `-A`, `-L`, `-T` (append-only, exclusive-use, temporary — Plan 9 file
  modes).
- `a -nt b`, `a -ot b` (file mtime compare).
- `f -older 3d12h` — relative-time test (`y` years, `M` months,
  `d` days, `h` hours, `m` minutes, `s` seconds).
- `-l string` — string-length as an integer, usable in arithmetic
  comparisons (`test -l $s -gt 10`).
- `=` / `!=` for strings; `-eq -ne -gt -ge -lt -le` for integers.

Parentheses for grouping must be quoted: `test \( -f a -o -d a \)` or
just `test '(' -f a -o -d a ')'`. Operators and operands are **always**
separate arguments.

For string comparisons, prefer `~`: `if (~ $1 -c) …` is cheaper and
clearer than `if (test $1 = -c) …`.

### `awk(1)`

This is Brian Kernighan's `awk` (the "one true awk"). Mostly identical
to `gawk` for common scripts, but:

- No `gensub`. Use `gsub` and rebuild.
- `nextfile`, `getline`, `system`, `length` (on arrays too), `match`,
  `split`, `sub`, `gsub`, `sprintf`, `tolower`, `toupper` all present.
- `utf(n)` converts a code point to its UTF-8 string.
- Regex flavour is `regexp(7)` — same quantifier/class restrictions as
  above.
- BUGS section notes: `split` with empty-string separator copes with
  UTF; numeric/string coercion needs explicit `+0` or `""` concatenation
  to disambiguate.
- `-safe` disables shell-exec / file-open / `ENVIRON` access.

### `sed(1)`

- Addresses are line numbers, `$` (last line), or `/regexp/` (regexp(7)
  syntax, no `-E`/`-r` flag — there *is* no extended-regex mode).
- `-g` flag *globally* (every substitution is `g`-suffixed). `-n` to
  suppress default output. `-l` to flush after each newline.
- Multi-line text in `a\`/`i\`/`c\` commands uses backslash-newline
  continuations, exactly as in classic sed. Each script line is a
  separate command (no `;` between commands inside a single `-e`).
- Pre-existing `}` `{` `!` and label commands work as expected.
- The 120-`w`-file ceiling is the only hard limit worth remembering.

GNU-isms that **don't work**: `-E`/`-r`, `\+`, `\?`, `\|`, `\b`, `\<` /
`\>`, in-place edit `-i`, `;` as a command separator. To bulk-edit a
file, do `sed … <in >out && mv out in`.

### `grep(1)`

- Pattern is `regexp(7)`. There is no `-E` (no extended mode) and no
  `-P` (no Perl-compat).
- **No `-F` (fixed strings) or `-m` (max count)**. For fixed-string matching with an early exit, use `awk`: `awk -v s=$str 'index($0, s) { print NR; exit }'`.
- Flags worth knowing: `-c` count, `-h` no filename header, `-i` case
  fold, `-l` files matching, `-L` files **not** matching, `-n` line
  numbers, `-s` silent (status only), `-v` invert, `-f file` patterns
  from file, `-b` unbuffered.
- `g pattern` is `grep -n` plus mandatory file-name tagging — handy on
  the command line.
- A pattern starting with `*` is taken literally (everything after
  becomes literal characters). Quote patterns generously.

### `tr(1)`

- Operates on runes. `tr A-Z a-z` works for ASCII; for general case
  folding use `awk '{print tolower($0)}'`.
- Flags: `-c` complement string1, `-d` delete, `-s` squeeze.
- `\x{1,4}` hex-rune syntax inside strings. `\` followed by 1–3 octal
  digits is also a rune.

### `wc(1)`

`-l` lines, `-w` words, `-r` runes (UTF code points, not bytes), `-b`
broken UTF codes, `-c` bytes. Default is `-lwc`. There is no `-m`.

### `cat(1)` / `read(1)`

`cat` is standard. `read` (in the same man page) reads **one line** from
stdin (or `-m` for multi-line, `-n N` for up to N lines) and is the
idiomatic way to consume a single line in interactive scripts. Use the
empty-IFS substitution variant so the whole line stays in one element
(otherwise `$ifs` will split on spaces and tabs):

```rc
line=`''{read}      # whole line, preserved spaces
words=`{read}        # list, default IFS split on whitespace
```

### `mc(1)`

Multicolumn printer — fills the terminal width. `lc` is `ls -p | mc`.

### `seq(1)`

`seq [first [incr]] last`. No `-s` separator (use `tr '\n' ' '`).
Numbers are floating-point; integer output by default.

### `expr`

There is **no** `expr(1)` man page in plan9port, but the binary exists
(BSD-style). The example scripts use it for arithmetic, e.g.
``size=`{expr $a + $b}``. For string length use `expr length $s`. For
pattern matching, prefer `~` or `awk`.

### `9p(1)`

Trivial 9P client. Subcommands: `read`, `write [-l]`, `stat`, `ls`,
`rdwr`, `readfd`, `writefd`. Path is `service/subpath` (service =
namespace socket name, e.g. `acme`, `plumb`). Examples are everywhere
in `scripts/` and in `llm/acme/SKILL.md`.

### `mk(1)`

Plan 9's `make`. Two things to internalise when writing mkfiles that
call rc:

- `MKSHELL` chooses the recipe interpreter; default is `sh`. To get rc
  semantics inside a recipe, set `MKSHELL=$PLAN9/bin/rc` *before* the
  rule. mkfiles included via `<` or `<|` see their own private
  `MKSHELL` that resets to `sh` each time.
- The recipe lines are passed as **one** script to the shell (not line
  by line as in `make`). `&&`/`||` chaining works as normal; trailing
  `\` continuations are optional.

Attributes after `:` on a rule line: `V` virtual (no actual file),
`Q` quiet (don't echo the recipe), `P cmd` use `cmd` to compare
freshness instead of mtime, `N` "no-make" (treat missing target as
up-to-date), `R` regex meta-rule.

### `9` wrapper

`/usr/local/plan9/bin/9` is a `sh` script: prepends `$PLAN9/bin` to
`$PATH`, exports `$PLAN9`, then `exec "$@"`. Use it from a Unix login
shell that does not have plan9port on `$PATH`:

```sh
9 acme &
9 man 1 rc
```

Inside an already-set-up environment (`$PLAN9/bin` on `$PATH`) the
prefix is redundant.

## Common idioms

### Script skeleton

```rc
#!/usr/local/plan9/bin/rc

if (~ $#* 0) {
    echo 'usage: '^$0^' arg' >[1=2]
    exit usage
}

# Defensive defaults from the environment
if (~ $antialias_threshold '')
    antialias_threshold=16

# … work …
exit
```

`exit usage` sets `$status` to the literal string `usage` (visible to
callers and shown in some `-x` traces); use any short label that
describes the failure. `exit` with no argument is success.

### Argument with default

```rc
if (~ $#* 0)
    target=plain
if not
    target=$1
```

This is the rc analogue of `${1:-plain}`.

### Reading one field out of structured output

The mainstay of every script in this tree:

```rc
line=`{9p read acme/$winid/ctl}
font=`{echo $line | awk '{print $7}'}
```

For multi-line output, iterate:

```rc
for(line in `{9p read acme/index}) {
    id=`{echo $line | awk '{print $1}'}
    ...
}
```

(`for(line in …)` with the default `$ifs` produces one element per
whitespace-bounded word. If the records contain spaces, set `ifs` to a
newline-only string inside a subshell — see Substitution above.)

### Safely ingesting JSON / multi-line payloads

To capture stdin without `rc`'s `$ifs` squashing internal spaces and newlines (crucial for passing intact JSON to `jq`), use an empty separator:

```rc
payload=`''{cat}
echo -n $payload | jq .
```

### Building a string by concatenation

```rc
newpath=`{echo $fontpath | sed 's|'$family'|'$family'-Bold|'}
```

Caret-driven string building avoids the `"$x"` interpolation reflex.

### Silencing best-effort writes

The 9P fileserver may surface diagnostics that are irrelevant to a
batch script — typically when a window vanishes between two calls.
Suppress stderr per-command:

```rc
echo name newname | 9p write acme/$winid/ctl >[2]/dev/null
```

Don't blanket-silence the whole script; you want unrelated errors to be
visible.

### Composing short functions

Plan 9 style favours one-screen scripts built from one-screen functions
each doing one thing. The example scripts compose `is_bold`, `to_bold`,
`to_nonbold`, `toggle`, `apply` — none more than ~10 lines.

### Per-window vs all-windows scripts

Per-window scripts use `$winid` (set by acme when launching a tag
command); all-windows scripts iterate `9p read acme/index`:

```rc
for(id in `{9p read acme/index | awk '{print $1}'})
    apply $id
```

See `llm/acme/SKILL.md` for the dirty-flag guard rule before mutating.

### Boolean composition

`&&` and `||` work on `$status` semantics:

```rc
test -f $f && cp $f $dest
mkdir -p $dir || { echo cannot make $dir >[1=2]; exit mkdir }
```

Note the `{ }` block in the `||` branch must end on the *same line*
(rc statement boundary) or be on its own line — newlines inside the
block are fine.

### Avoid leaking function-local state

```rc
fn apply_bold {
    @{
        font=`{9p read acme/$1/nctl | awk '{print $1}'}
        ... mutate font ...
    }
}
```

The `@{ … }` runs the block in a subshell; any variable changes inside
disappear when it exits.

## Pitfalls

- **`exit 0` is misleading even when it works.** The string `0` is
  parsed back to Unix exit code 0, so it succeeds — but rc's whole
  status protocol is string labels (`exit usage`, `exit dirty`, …).
  Reserve `exit` (no arg) for success and a descriptive word for
  failure; the label travels up to callers and shows in traces.
- **`if (~ $status '')` clobbers `$status`.** Save first:
  `s=$status; if (~ $s '') …`.
- **Empty list vs empty string look the same to `~`.** Both `x=''`
  (1-element list with empty string) and `x=()` (zero-element list)
  satisfy `~ $x ''`. To distinguish them, test the length with
  `~ $#x 0` (true only for the empty list).
- **Heredoc inside `{}` or `fn` — silently parsed as commands.** Move
  to top level or use a series of `echo` calls (see
  `../install-hook.rc` for the worked workaround).
- **Double-quoting does nothing.** `"hello $x"` is a literal 10+
  character string including the two `"` marks and a literal `$x`.
  Use single quotes with carets: `'hello '^$x`.
- **`!` and `~` interact.** `! ~ x pat` negates the match. Read it as
  "no pattern matched", not "not equal".
- **`for(i in $list)` over an empty list runs the body zero times,**
  leaving `$i` unset (`''`). Guard with `if (! ~ $#list 0)` when
  downstream code assumes `$i` exists.
- **Background pipelines don't always close all fds.** Use `>[2=]` to
  explicitly close stderr when handing a `&` job over to a long-lived
  reader; otherwise the parent rc may hang on shutdown.
- **`SOH` (`\001`) is the environment list separator.** A variable
  with multi-element value is exported as one env entry with
  `SOH`-joined components. Re-importing it in another rc preserves
  the list; other shells see a single string containing literal
  `SOH` bytes.
- **`mk` recipes default to `sh`, not rc.** Set
  `MKSHELL=$PLAN9/bin/rc` before any rule whose recipe uses rc syntax
  (`for(i in …)`, `~`, `if not`).
- **`while () { … }` is the rc equivalent of `while true; do … done`.**
  An empty condition is null status, hence true. There is no
  `while true` keyword.

## Plumbing and 9P from rc

Two lines that turn up everywhere:

```rc
plumb -d edit foo.c:42                    # "click to open" foo.c at line 42
cat $home/lib/plumbing | 9p write plumb/rules    # reload plumber rules
9 9p ls acme                              # what's mounted
9p read acme/$winid/body                  # current window text
```

See `9 man 4 plumber`, `9 man 7 plumb`, `9 man 1 9p`.

## Writing scripts that drive acme

The 9P surface acme exposes (`/mnt/acme`), its ctl/nctl verbs, the
`event` protocol, dirty-window rules, and the
worked example scripts are all documented in **`../acme/SKILL.md`**.
That file is the authoritative reference for any rc script that reads
or writes inside `acme/`.

The two absolute rules repeated here for emphasis:

- **Never write `dirty` or `clean` to `acme/<id>/ctl` from an outside
  script.** They reset undo state and are reserved for the owning
  controller (e.g. `win`).
- **Never mutate the body of a dirty window.** Check field 5 of
  `acme/<id>/ctl` (or of the matching line in `acme/index`); skip or
  warn on `1`.

Environment pre-set by acme when it spawns a tag command:

- `$winid` — id of the launching window.
- `$samfile`, `$%` — file name in that window's tag.
- `$acmeaddr` — for 2-1 chords: a sam-style address of the chorded arg.
- `$font`, `$emphfont` — current default fonts.
- `$tabstop`, `$PLAN9`, `$home`.
- `$acmeshell` — shell rc binds for B2 commands (default `rc`).

Plus user-defined entries (read by the font/color/emph scripts):
`$color`, `$varcolor`, `$antialias_threshold`.

## Higher-level acme helpers (`acmeevent`, `acme.rc`)

For controllers that intercept the per-window event stream (button-2,
button-3, keyboard, body insert/delete), rc-level parsing is painful.
Plan9port ships `acmeevent(1)` and a sourceable function library
`$PLAN9/lib/acme.rc`:

```rc
. /usr/local/plan9/lib/acme.rc

newwindow                          # create a window; sets $winid
winname /tmp/scratch
winwrite body                      # stream stdin into the body
winctl 'addr=dot'
9p read acme/$winid/event | acmeevent | wineventloop
```

`acmeevent` translates the binary-ish event lines into a token stream
(`c1`, `c2`, `q0`, `q1`, optional chord args), and `wineventloop` calls
back into user-defined `fn` hooks (`exec1`, `exec2`, `look1`, …). Read
`acmeevent(1)` for the protocol; use this whenever a script needs to
react to user actions instead of just running once.

## Project conventions

These apply to every script added under this tree.

- Reply to the user in **French**. Code, identifiers, file content,
  program messages, error messages, comments: **English**.
- **No emojis** anywhere.
- Shebang `#!/usr/local/plan9/bin/rc`.
- Configuration from the environment, never a config file.
- Short functions, one responsibility each, compose with `awk`/`sed`/
  `grep`/`expr`. The shipping scripts average <40 lines.
- Use `~`/`! ~` for string equality and small enumerations; reach for
  `test` only for file-attribute tests.
- Suppress stderr only on best-effort `9p write`s (`>[2]/dev/null`).
- When adding or removing a script in `scripts/`, update both
  `SCRIPTS=` in `scripts/mkfile` and the tables in `scripts/README.md`.
- The man pages — `9 man 1 rc`, `1 test`, `1 sed`, `1 awk`, `1 grep`,
  `7 regexp`, plus `1 acme` / `4 acme` / `7 plumb` for acme work — are
  the source of truth. Consult them before guessing.

## Pointers

- `$PLAN9/llm/acme/SKILL.md` — 9P, ctl/nctl, event protocol, dirty rules.
- `$PLAN9/scripts/README.md` — the worked-example script catalogue.
- `$PLAN9/scripts/` — `A`, `A+`, `A-`, `F`, `F+`, `F-`, `Bold`,
  `BoldAll`, `Color`, `ColorAll`, `EmphAs` — read these alongside this
  file when starting a new script.
- `$PLAN9/llm/install-hook.rc` — non-trivial rc that does jq-based JSON
  merging into `~/.claude/settings.json`. Useful as an example of how
  to drive an external tool from rc, handle missing dependencies, and
  cope with the heredoc-in-`{}` limitation (it uses a function full of
  `echo` lines instead of a heredoc).
- `$PLAN9/llm/bin/acme-update`, `$PLAN9/llm/bin/acme-sync-recent` — rc
  scripts that read JSON from Claude Code on stdin, parse with `jq`,
  and write to `/mnt/acme`.
- `$PLAN9/rcmain` — what rc evaluates at startup on plan9port. Worth
  reading once to understand `$home`, `$prompt`, `$ifs`, the
  interactive `cd` redefinition for `awd`, and the `lib/profile`
  loading order.
- `$PLAN9/src/cmd/rc/` — the rc implementation itself.
