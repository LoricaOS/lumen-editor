# lumen-editor

The Text Editor app for **AspisOS**, a capability-based, no-ambient-authority
x86-64 operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

editor is a small load/edit/save text editor over a fixed-cell text grid. It is
a standalone component of the Lumen desktop, distributed as a
[herald](https://github.com/AspisOS/AspisOS) package, and runs as an **external
client** of the [lumen](https://github.com/AspisOS/lumen) compositor — it
connects to `/run/lumen.sock` over the Lumen window protocol rather than being an
in-process compositor built-in.

## Where editor fits

AspisOS is decomposed into independent repositories. editor sits at the leaf of
the graphical stack:

| Repo | Role |
|------|------|
| [`AspisOS/Aegis`](https://github.com/AspisOS/Aegis) | The kernel: capability model, the filesystem, the syscalls the desktop runs on. |
| [`AspisOS/lumen`](https://github.com/AspisOS/lumen) | The compositor / display server. Owns the framebuffer; every GUI app is one of its clients. |
| [`AspisOS/glyph`](https://github.com/AspisOS/glyph) | The GUI toolkit editor links against: the bitmap and TTF font renderers (`draw_*`, `font_*`), theme values, and the client side of the Lumen protocol (`lumen_client.h`). |
| `AspisOS/lumen-editor` | **This repo.** The text editor app. |

## What it does

Grounded in `src/main.c`:

- Opens a fixed **640x460** window titled "Text Editor" and edits a line-buffer
  document (up to **1024 lines x 512 columns**, held in a fixed array).
- Editing primitives: insert, newline (splits the line), backspace (merges into
  the previous line), tab expansion (4 spaces), arrow-key cursor navigation, and
  vertical/horizontal scrolling that follows the cursor.
- **File path:** `argv[1]` if given, otherwise `$HOME/untitled.txt` (Lumen
  propagates the session's `HOME`, so a non-root session saves into its own home
  rather than `/root`). A missing file opens as an empty new document; an
  over-long file is truncated with a status message.
- **Save:** Ctrl+S or the on-screen Save button (`open(O_WRONLY|O_CREAT|O_TRUNC)`
  then line-by-line write). Clicking the title text turns it into an editable
  path field for rename / save-as; clicking the text area positions the cursor.
- A bottom status bar shows `Ln/Col`, the last action, and a key hint; the title
  shows the basename with a `*` modified marker.
- **Rendering detail:** the text grid uses the fixed 10x20 bitmap font so
  cursor-to-cell math is exact, while the chrome uses the TTF UI font. The
  background fill deliberately avoids `C_TERM_BG` (the compositor's color key for
  frosted-glass terminal windows), so the editor does not render as translucent
  glass.

## Capabilities

AspisOS grants a process no ambient authority; it can touch the system only
through capabilities declared for it at exec time. editor's policy
(`pkg/etc/aegis/caps.d/editor`) is the baseline:

```
service
```

The `service` profile and **no** elevated capabilities. Its `open`/`read`/
`write` go through the session's own filesystem authority — the editor is
granted nothing beyond what an ordinary client already holds, so it can only
touch files the session itself can.

## Status

editor is a basic load/edit/save editor: a single document, a fixed line/column
ceiling, no undo, search, or syntax highlighting. It is deliberately minimal and
expected to grow as AspisOS matures. What ships today is honest about its scope.

## Building

editor builds with a musl cross-compiler against a **pinned**
[glyph](https://github.com/AspisOS/glyph) toolkit artifact (the GUI libraries it
links), then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `make` runs `tools/fetch-glyph.sh $(GLYPH_VERSION)` to download and unpack the
  pinned toolkit into `toolkit/`, compiles `src/*.c` against it, then packs.
- `MUSL_CC` is the musl cross-compiler (defaults to `musl-gcc` on `PATH`; the
  only toolchain assumption — point it at an Aegis-native `cc` to build on-device
  in the future).
- `HERALD_KEY` is the ECDSA-P256 key that signs the `.hpkg`.
- `GLYPH_VERSION` pins the toolkit release; `VERSION` is this app's own version.

Output: `lumen-editor.hpkg` (a `class=system` herald package) +
`lumen-editor.hpkg.sig`.

## Package payload

`lumen-editor.hpkg` is a **herald `class=system` package**: a manifest-first
uncompressed POSIX `ustar` archive with a detached ECDSA-P256/SHA-256 signature
(`tools/pack.sh`). Its herald id (`lumen-editor`) deliberately differs from the
bundle/exec name (`editor`), and it installs across two trees — which is exactly
why it is `class=system` (first-party, signature-trusted, installed verbatim)
rather than an ordinary single-prefix package:

```
/apps/editor/editor             the app binary
/apps/editor/app.ini            the bundle descriptor (name=Editor, exec=editor)
/etc/aegis/caps.d/editor        its capability policy
```

## Repository layout

```
src/        editor source (main.c)
pkg/        install-tree skeleton shipped verbatim (apps bundle + caps.d)
tools/      fetch-glyph.sh (pinned toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this app's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — editor is an external client of the compositor, so installing
it pulls [lumen](https://github.com/AspisOS/lumen). lumen also ships the desktop
fonts (Inter, JetBrains Mono), so editor inherits them transitively; there is no
separate font package.
