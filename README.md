# SyncTERM SkyPix fixes

Fixes and additions for **SkyPix** emulation in [SyncTERM](https://gitlab.synchro.net/main/sbbs/-/tree/master/src/syncterm) (part of Synchronet / `sbbs`), developed against a live SkyPix board — **Reign of Fire BBS** (`call.rofbbs.com:6800`, C‑Net/Amiga 5.37b; choose **[S] SkyPix** at the prefs prompt). Website: <https://www.rofbbs.com>.

Before these changes, SyncTERM could barely display a SkyPix board; with them it renders the board essentially the same as RGTerm and IcyTERM.

---

## Status — these fixes are upstream. Work continues in SyncTERM itself.

**This is not a fork and it is not where SkyPix development happens.** It was
only ever somewhere to point a maintainer at, because the Synchronet GitLab
needs a `*.synchro.net` account to file from. Deuce (Stephen Hurd) found it,
and the six fixes below are in the SyncTERM **v2** tree; v2 has since gone
considerably further than anything here — notably proper 640x200 → 640x400
handling.

**Where to go instead:**

| For | Go to |
|---|---|
| SyncTERM v2 source | <https://github.com/RealDeuce/SyncTERM> (read-only mirror of v2+) |
| Filing a SkyPix bug | <https://sf.net/p/syncterm> — the official tracker, and it **allows anonymous filing**. Not the GitLab. |
| The v1 SkyPix report | SourceForge **#299**, which the maintainer pinned as a conformance test |
| The live v2.0α thread | [RealDeuce/SyncTERM issue #1](https://github.com/RealDeuce/SyncTERM/issues/1) |

**What this repo is still good for**, because it is not duplicated upstream:
the **Amiga font set** (SyncTERM ships none), the write-up in
[`docs/report.md`](docs/report.md), and the **comparison screenshots against
RGTerm and IcyTERM** — the maintainer can't produce those himself, as RGTerm
doesn't run on his OS.

> **`docs/v2-screenshots/` is linked directly from an open upstream issue.**
> Those four PNGs are served to that thread by raw URL. Do not move, rename
> or remove them, and do not delete this repository while that issue is open
> — the images would 404 inside a live conversation, and retroactively.

---

## What was fixed here (now upstream in v2)

| # | Issue | Fix |
|---|---|---|
| 1 | SkyPix only worked if the graphics setting was **RIPv3** (a different protocol) | Added a dedicated **SkyPix** option (`RIP_VERSION_SKYPIX`) to the graphics dropdown |
| 2 | Set‑Font commands (`ESC[10;size!name!`) **truncated at the first `!`**, leaking the font name as on‑screen text | ESC sub‑machine keeps accumulating the font‑name string to its closing `!` |
| 3 | Fonts opened relative to CWD, and **SyncTERM ships no Amiga fonts** | Resolve fonts against `<config>/fonts/`; **canonical Amiga font set provided** (`fonts/`) |
| 4 | Positioned text drew at the wrong place (runs collapsed/left‑pinned) | Two parse/draw‑ordering fixes: draw SkyPix text inline in stream order; don't defer SkyPix `!` terminators |
| 5 | **Backspace / line editing broke** after a SkyPix screen | Clear `cterm->font_render` on font reset; inline draw only intercepts printable + CR/LF |
| 6 | **AREA FILL (command 3)** was an unimplemented stub, so outlined shapes/letters stayed hollow | Implemented a seed flood fill with a leak guard |

Full write‑up with causes, code references, and before/after screenshots: **[`docs/report.md`](docs/report.md)**.

## Known remaining issue

Script‑heavy SkyPix screens (e.g. C‑Net's "ACIS supporter" screen that outlines letters then flood‑fills them) render **non‑deterministically** — different on each draw; fills occasionally leak and positioned labels occasionally run together. This looks like a render‑pipeline timing/coherence issue (fill `getpixels` reading before line draws flush, and/or the SkyPix‑font‑active state depending on network‑read boundaries), not a single command bug — it needs iteration with visible pixels. A raw protocol capture is available on request. See `docs/report.md`.

## Applying

```sh
cd sbbs                 # your Synchronet checkout
patch -p1 < /path/to/syncterm-skypix-fixes.patch
# rebuild SyncTERM (from src/syncterm):  make RELEASE=1
```
The patch touches `src/syncterm/{bbslist.h,bbslist.c,ripper.c,wren_bind_menu.c}`. The modified files are also committed under `src/syncterm/` here for direct viewing, and the exact code diff is this repo's "fixes" commit.

## Fonts

SkyPix boards name Amiga fonts (Courier, Times, Diamond, Emerald, Pearl, Ruby, …) at specific point sizes; the terminal must have them. `fonts/amiga-font/` is the Amiga `.font` set SyncTERM reads (install to `~/.config/syncterm/fonts/`). `fonts/rgterm-spf/` is RGTerm's `.SPF` set, included for reference and for the few sizes not yet available as `.font` (sapphire 15/18, topaz 8). See `fonts/README.txt` for provenance and licensing (freely redistributable Amiga bitmap fonts).

## Test targets

`reign-of-fire-phonebook.lst` (SyncTERM dialing‑directory format, credentials stripped) — five Reign of Fire boards covering ANSI, PETSCII/C64, and SkyPix. Handy for regression testing.

---
*Contributed by [SpitfireTN](https://www.rofbbs.com) / Reign of Fire BBS.
Kept as a reference and evidence store; SkyPix work itself continues in
SyncTERM v2 and on the SourceForge tracker — see **Status** above.*
