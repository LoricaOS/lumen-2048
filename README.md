# lumen-2048

The 2048 game app for **LoricaOS**, a capability-based, no-ambient-authority
x86-64 operating system built on the from-scratch
[Aegis](https://github.com/LoricaOS/Aegis) kernel.

2048 is the classic sliding-tile puzzle: a 4x4 board of powers of two that you
slide and merge to reach the 2048 tile. It is a leaf component of the Lumen
desktop, distributed as a [herald](https://github.com/LoricaOS/LoricaOS) package,
and runs as an **external client** of the
[lumen](https://github.com/LoricaOS/lumen) compositor — it connects to
`/run/lumen.sock` over the Lumen window protocol and is handed a shared-memory
buffer to draw into, rather than being an in-process compositor built-in.

## Where 2048 fits

LoricaOS is decomposed into independent repositories. 2048 sits at the leaf of
the graphical stack:

| Repo | Role |
|------|------|
| [`LoricaOS/Aegis`](https://github.com/LoricaOS/Aegis) | The kernel: capability model, `AF_UNIX` sockets, `memfd`, the syscalls the desktop runs on. |
| [`LoricaOS/lumen`](https://github.com/LoricaOS/lumen) | The compositor / display server. Owns the framebuffer; every GUI app is one of its clients. |
| [`LoricaOS/glyph`](https://github.com/LoricaOS/glyph) | The GUI toolkit 2048 links against: the software renderer (`draw_*`, `font_*`), theme/accent values, and the client side of the Lumen protocol (`lumen_client.h`). |
| `LoricaOS/lumen-2048` | **This repo.** The 2048 game app. |

## What it does

Grounded in `src/main.c`:

- Opens a fixed **360x474** window titled "2048" via `lumen_window_create` and
  draws a header (title + SCORE/BEST boxes), a 4x4 tile grid, and a footer hint
  into the shared surface.
- Pure userspace logic — a 4x4 `int board[4][4]` of powers of two with classic
  slide-and-merge mechanics (`slide_line` compacts and merges one line toward
  index 0; `cell_of`/`do_move` apply it in all four directions). No new kernel
  support, no file I/O, no syscalls beyond the compositor socket.
- A self-contained **xorshift32 PRNG** (`rng_seed`/`rng_next`) seeded from
  `getpid()` and `time()` so each launch differs; `spawn_tile` drops a new 2
  (90%) or 4 (10%) into a random empty cell after every move.
- **One input dispatch** (`feed_key`): arrow keys (the synthetic
  `KEY_UP`/`DOWN`/`LEFT`/`RIGHT` codes Lumen delivers) or `W`/`A`/`S`/`D` slide
  the board, `R` starts a new game, and Esc / the window close request quit.
- A small state machine over the move: `handle_move` ignores input once
  `has_moves()` fails (game over), tracks `best` against `score`, and raises a
  one-shot **You Win!** banner the first time a 2048 tile appears
  (`win`/`win_seen`), dismissed on the next move.
- Rendering is dirty-flagged (`render` only redraws and calls
  `lumen_window_present` when `g.dirty` is set). Tiles use the classic 2048
  palette (`tile_bg`), with the number font size stepping down as the value
  grows wider and a centered **Game Over** / **You Win!** banner drawn over the
  board when the game ends.

## Capabilities

LoricaOS grants a process no ambient authority; it can touch the system only
through capabilities declared for it at exec time. 2048's policy
(`pkg/etc/aegis/caps.d/2048`) is the baseline:

```
service
```

The `service` profile and **no** elevated capabilities — 2048 is pure compute
over a window surface and touches nothing beyond the compositor socket.

## Status

2048 is intentionally small: a single-purpose, self-contained game. What ships
today is the complete classic ruleset — slide, merge, spawn, win at 2048, game
over when the board locks — and is honest about its scope rather than
feature-padded. There is no persistence yet, so the best score lives only for
the session.

## Building

2048 builds with a musl cross-compiler against a **pinned**
[glyph](https://github.com/LoricaOS/glyph) toolkit artifact (the GUI libraries it
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

Output: `lumen-2048.hpkg` (a `class=system` herald package) +
`lumen-2048.hpkg.sig`.

## Package payload

`lumen-2048.hpkg` is a **herald `class=system` package**: a manifest-first
uncompressed POSIX `ustar` archive with a detached ECDSA-P256/SHA-256 signature
(`tools/pack.sh`). Its herald id (`lumen-2048`) deliberately differs from the
bundle/exec name (`2048`), and it installs across two trees — which is exactly
why it is `class=system` (first-party, signature-trusted, installed verbatim)
rather than an ordinary single-prefix package:

```
/apps/2048/2048              the app binary
/apps/2048/app.ini          the bundle descriptor (name=2048, exec=2048)
/etc/aegis/caps.d/2048      its capability policy
```

## Repository layout

```
src/        2048 source (main.c)
pkg/        install-tree skeleton shipped verbatim (apps bundle + caps.d)
tools/      fetch-glyph.sh (pinned toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this app's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — 2048 is an external client of the compositor, so installing it
pulls [lumen](https://github.com/LoricaOS/lumen). lumen also ships the desktop
fonts (Inter, JetBrains Mono), so 2048 inherits them transitively; there is no
separate font package.
