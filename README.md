# lumen-editor

The Text editor app for **AspisOS**, a capability-based, no-ambient-authority
operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

editor is a small load/edit/save text editor. It is a standalone component of
the Lumen desktop, distributed as a
[herald](https://github.com/AspisOS/AspisOS) package, and runs as an external
client of the [lumen](https://github.com/AspisOS/lumen) compositor (it connects
to `/run/lumen.sock` over the external window protocol rather than being an
in-process compositor built-in).

## Role in the system

- A `/apps` bundle app: launched from the desktop via its `app.ini` descriptor
  (`name=Editor`, `exec=editor`), it opens a 640x460 window.
- Line-buffer model (up to 1024 lines, 512 columns) with a cursor:
  insert/delete/newline/backspace, arrow-key navigation, vertical and horizontal
  scrolling, and tab expansion.
- File path is `argv[1]` if given, otherwise `$HOME/untitled.txt` (so a non-root
  session saves into its own home, not `/root`); a missing file opens as an
  empty new document. Save via Ctrl+S or the on-screen Save button. Clicking the
  title text turns it into an editable path field for rename / save-as.
- The text area uses the fixed 10x20 bitmap font so cursor-to-cell math is
  exact; the window chrome uses the TTF UI font. The background fill avoids
  `C_TERM_BG`, the compositor's color key, so the window does not render as
  translucent glass.

## Capabilities

editor's cap policy (`pkg/etc/aegis/caps.d/editor`) is the baseline:

```
service
```

No elevated capabilities. Its file reads and writes go through ordinary `open`/
`read`/`write` against the session's own filesystem authority — the editor is
granted nothing beyond what a normal service-profile client already has.

Because its herald package id (`lumen-editor`) intentionally differs from the
bundle/binary name (`editor`), and it installs across `/apps` and
`/etc/aegis/caps.d`, editor is a `class=system` package: first-party and
signature-trusted, installed verbatim by herald.

## Building

editor fetches a pinned [glyph](https://github.com/AspisOS/glyph) toolkit
artifact (the GUI libraries it links) and builds against it, then packs a signed
herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `lumen-editor.hpkg` (a `class=system` herald package) +
`lumen-editor.hpkg.sig`.

## Package payload

```
/apps/editor/editor             the app binary
/apps/editor/app.ini            the bundle descriptor (launcher metadata)
/etc/aegis/caps.d/editor        its capability policy
```

## Repository layout

```
src/        editor source
pkg/        install-tree skeleton shipped verbatim (apps bundle + caps.d)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — editor is an external client of the compositor, so installing
it pulls [lumen](https://github.com/AspisOS/lumen) (which also supplies the
desktop fonts).
