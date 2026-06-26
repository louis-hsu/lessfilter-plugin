# lessfilter — handler plugin system

`lessfilter` extends or overrides lesspipe's file-type handling **without
modifying `lesspipe.sh`**. You add or replace handlers by dropping small
executable scripts into a plugin directory.

## How it hooks into lesspipe

`lesspipe.sh` already consults a `lessfilter` program before running its own
built-in handlers (see the tail of `lesspipe.sh`):

```bash
[[ -x "${HOME}/.lessfilter" ]] && "${HOME}/.lessfilter" "$1" && exit "$retval"
if has_cmd lessfilter; then
    lessfilter "$1" && exit "$retval"
fi
# ... built-in handling (show) runs only if the above did not handle the file
```

The `lessfilter` script shipped here is installed into `PATH`. When lesspipe
calls `lessfilter "$1"`:

- **exit 0** → the file was handled; lesspipe stops (built-ins skipped).
- **nonzero** → not handled; lesspipe falls through to its built-ins.

Because lesspipe checks `~/.lessfilter` first, that personal script (if present)
still takes priority over this plugin system. Nothing in `lesspipe.sh` changes.

## Why a dispatcher (not a flat script)

A naive `lessfilter` would `cat`/`if`-chain every format inline, or loop over
every script in a directory and try each one. Both are slow and grow without
bound. Instead, this dispatcher computes the file's mime type **once** and looks
up a single handler **by name** — at most three `stat`-like checks per
directory, no scanning.

## Handler lookup

### Search directories (first existing match wins)

| Order | Directory | Purpose |
|-------|-----------|---------|
| 1 | `~/.config/lesspipe/handlers.d` | user-local; overrides everything below |
| 2 | `$LESSPIPE_HANDLERS_DIR` | single dir override (optional) |
| 3 | `/etc/lesspipe/handlers.d` | system-wide / package-installed |

User-local always wins over the system directory, so a user can override a
distro-provided handler just by placing a file with the same name in `1`.

### Handler name within a directory (most specific first)

For a file, `lessfilter` runs `file --mime-type` to get `<fcat>/<ftype>` and
derives `<fext>` from the file name, then tries:

| Order | Name | Example | When it matches |
|-------|------|---------|-----------------|
| 1 | `<fcat>-<ftype>` | `image-jpeg`, `application-pdf` | most specific |
| 2 | `<ftype>` | `jpeg`, `pdf`, `json` | usually sufficient |
| 3 | `<fext>` | `csv`, `md` | mime type is generic (`text/plain`) |
| 4 | `<fcat>` | `image`, `audio`, `video` | one handler for a whole category |

This solves the three common cases cleanly:

- **Binary / well-typed files** (jpg, png, pdf, json) report a meaningful mime
  subtype, so `<ftype>` alone selects the handler.
- **Generic text files** (csv, md, tsv) all report `text/plain`, so the
  **extension** `<fext>` selects the handler.
- **A whole category handled identically** (e.g. preview any image with `chafa`):
  a single handler named `image` (the `<fcat>` catch-all) covers `image/jpeg`,
  `image/png`, `image/gif`, ... — no need for separate `jpeg`/`png`/`gif`
  scripts. Because it is the least specific candidate, a more specific `<ftype>`
  or `<fext>` handler still wins when present.

A handler that exists but exits nonzero (declines) lets the next candidate name,
then the next directory, then lesspipe's built-ins take over.

## Writing a handler

A handler is just an executable file named per the table above.

It receives:

| | |
|---|---|
| `$1` | path to the file |

Exported environment variables:

| Variable | Meaning | Example |
|----------|---------|---------|
| `LESSPIPE_FCAT` | mime category | `text`, `application`, `image` |
| `LESSPIPE_FTYPE` | mime subtype | `plain`, `pdf`, `json` |
| `LESSPIPE_FEXT` | file name extension | `csv`, `md`, `sh` |
| `LESSPIPE_COLOR` | `always` if less can render ANSI, else `auto` | `always`, `auto` |

Contract:

- Must be **executable** (`chmod +x`); non-executable matches are skipped.
- **exit 0** = handled, output already written to stdout.
- **nonzero** = not handled, fall through.
- **stderr is merged into stdout** (so errors are visible in the pager).
- For piped/stdin input (`$1` is `-`) the dispatcher exits nonzero immediately;
  handlers are file-based only.

## Color

This is the most common surprise, so it gets its own section.

A handler's output is **piped into `less`** — it is never a terminal. Tools that
colorize by default (`bat`, `jq`, `ls`, `grep`, ...) detect the non-terminal and
**silently strip color**. So colored output requires two things:

1. **The handler must force color.** Use the tool's explicit "always" flag
   (`jq -C`, `bat --color=always`, `ls --color=always`, ...). Without it, output
   is plain regardless of how `less` is run.

2. **`less` must render ANSI**, i.e. be invoked with `-R` (raw control chars),
   usually via `export LESS=-R`. Without `-R`, forced color shows up as literal
   `^[[…m` escape sequences instead of color.

Because `lesspipe.sh` does not export its own internal color decision (and this
plugin system does not modify `lesspipe.sh`), `lessfilter` recomputes it from
`$LESS` and exports **`LESSPIPE_COLOR`**:

- `LESSPIPE_COLOR=always` — `$LESS` contains `-R`/`-r` (raw-control-chars);
  force color.
- `LESSPIPE_COLOR=auto` — otherwise; emit plain output (or the tool's mono mode).

Gate the force-color flag on it, so the same handler does the right thing whether
or not the user runs `less` with `-R`:

```bash
[[ "$LESSPIPE_COLOR" == always ]] && color=always || color=never
sometool --color="$color" "$1"
```

### Example: JSON (matched by mime subtype)

`~/.config/lesspipe/handlers.d/json`

```bash
#!/usr/bin/env bash
command -v jq >/dev/null 2>&1 || exit 1     # decline if jq missing
[[ "$LESSPIPE_COLOR" == always ]] && jqcolor=-C || jqcolor=-M
jq "$jqcolor" . "$1"
exit 0
```

### Example: CSV (matched by extension)

`~/.config/lesspipe/handlers.d/csv`

```bash
#!/usr/bin/env bash
command -v qsv >/dev/null 2>&1 || exit 1
command -v bat >/dev/null 2>&1 || exit 1
[[ "$LESSPIPE_COLOR" == always ]] && batcolor=always || batcolor=never
qsv table "$1" | bat -l csv --color="$batcolor" --paging=never
exit 0
```

### Example: Excel / spreadsheets (one script, several extensions)

Excel mime subtypes are long, differ per format, and `.xlsx` can even report
`application/zip` - so match by **extension**. Several extensions share one
renderer via symlinks, named with a leading `_` so the script itself is never
matched directly:

`~/.config/lesspipe/handlers.d/_excel` (with `xlsx`, `xls`, `xlsm`, `ods`
symlinked to it)

```bash
#!/usr/bin/env bash
command -v xleak >/dev/null 2>&1 || exit 1   # decline if xleak missing
xleak -n 0 "$1"                              # -n 0 = all rows
exit 0
```

```bash
cd ~/.config/lesspipe/handlers.d
ln -s _excel xlsx && ln -s _excel xls && ln -s _excel xlsm && ln -s _excel ods
```

Ready-to-copy versions of all of these live in `handlers.d/examples/`.

## Installing a handler

```bash
mkdir -p ~/.config/lesspipe/handlers.d
cp handlers.d/examples/json ~/.config/lesspipe/handlers.d/json
chmod +x ~/.config/lesspipe/handlers.d/json
```

Verify the right handler is chosen for a file (and emits ANSI when colored):

```bash
LESS=-R LESSPIPE_HANDLERS_DIR=~/.config/lesspipe/handlers.d \
    lessfilter some.json | cat -v   # exit 0 = handled; ^[[..m = color codes
```

Then view it for real through less:

```bash
LESS=-R less some.json
```

## Overriding a built-in format

To replace lesspipe's own handling of a format, add a handler whose name matches
that format's mime subtype or extension and exit 0. Since `lessfilter` runs
before the built-ins, your handler wins. To **suppress** filtering for a format
entirely, write a handler that emits the file unchanged and exits 0:

```bash
#!/usr/bin/env bash
cat "$1"
exit 0
```
