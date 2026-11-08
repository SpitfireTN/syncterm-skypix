# SkyPix emulation: no dedicated setting, font-name leak, fonts never load, text mispositioned, line editing broken, AREA FILL unimplemented

**Component:** syncterm (SDL/terminal, SkyPix emulation in `ripper.c`)
**Tested on:** git rev 61a343a641, Linux x64 SDL2 build (local build "SyncTERM 1.10i")
**Board:** Reign of Fire BBS — C-Net/Amiga BBS 5.37b, `call.rofbbs.com:6800` (Telnet), selects **[S] SkyPix** at its terminal-preferences prompt (website: https://www.rofbbs.com)
**Reference terminals that render this board correctly:** RGTerm, IcyTERM

Six issues found using SyncTERM against a live SkyPix board — the first four each block the next. **All six are fixed/implemented locally and in the attached patch, verified against the live board**; SkyPix screens now render like RGTerm/IcyTERM apart from one documented non-deterministic edge case. Each is written up below with cause and fix.

---

## 1. SkyPix has no setting of its own — it only works if you select RIPv3

The connection graphics dropdown is `Off / RIPv1 / RIPv3` (`bbslist.c` `rip_versions[]`). SkyPix parsing lives inside the RIP engine and is gated by `rip.enabled`, which `init_rip_ver()` sets from the RIP version. So the only way to make SyncTERM act on SkyPix `ESC[…!` sequences is to mark the connection **RIPv3** — a different, unrelated protocol (RIPscrip/PC vs SkyPix/Amiga).

**Suggestion:** give SkyPix a first-class entry (a `RIP_VERSION_SKYPIX` value / dedicated `skypix` mode) in the dropdown so an Amiga board can be marked SkyPix without masquerading as RIPscrip, and so a SkyPix connection does not advertise a RIPSCRIP version string. (Prototyped locally; small and self-contained.)

## 2. Set-Font commands truncate at the first `!`, leaking the font name as text

The board's set-font command is:

```
ESC[10;<size>!<fontname>!        e.g.  ESC[10;36!Courier.font!
```

i.e. numeric params, then a **string argument** (the font name) between two `!`. `do_skypix()` is written for exactly this two-`!` form (it extracts the name between them). **But** the ESC sub-machine that feeds it (`cb_esc_feed`) only accumulates `[0-9;]` and **terminates on the first `!`**. Any other byte (the letters of the font name) hits the "flush accumulator to the text window" path.

Result: the parser dispatches a truncated `ESC[10;36!` (no name → font never set), then `Courier.font!` spills out as literal text. This is the on-screen leak (`Courier.font!`, `Diamond.font!`, `Emerald.font!`, `Pearl.font!`, …).

**Fix:** when a Set-Font command's first `!` arrives, recognize that a string argument follows (command 10 with size ≠ 0) and keep accumulating — letters included — through to the closing `!`, then dispatch the whole buffer. (Prototyped locally; the name then reaches `do_skypix` intact.)

**Same latent bug, command 16 (SkyPix XModem):** it also carries a string (filename) in the same two-`!` form and would truncate identically. Its string just never arrives today because of this bug.

## 3. Even with the name, the font can't be found, and none ship

`do_skypix` case 10 does a bare `fopen("courier.font", "rb")` (and then `fopen("courier/24")` for the sized glyph file) — relative to the process working directory, which for a GUI launch is wherever the desktop launcher put it. It also fails because **SyncTERM ships no SkyPix/Amiga fonts at all**, so the load hits `TODO: Unable to open font`.

**Fix (two parts):**
- Resolve font files against a known directory (I used `<config-dir>/fonts/`) instead of CWD.
- **Ship the canonical SkyPix fonts.** RGTerm and IcyTERM render this board because they bundle the Amiga font set; SyncTERM has none. A complete, freely-redistributable Amiga `.font` set (the faces/sizes boards actually request) is attached to this report — see the companion font archive and its README. Boards request, e.g., Courier 36, Times 15, Diamond 20, Emerald 20, Ruby 8, Pearl 8 on a single login screen.

## 4. SkyPix text drawn at the wrong position (FIXED — see patch)

With 1–3 addressed locally (dedicated SkyPix setting, parser accumulates the name, fonts present and found), the board **does** render — and this is the key diagnostic: the **graphics are pixel-correct** (box, registration crosses, rules, the © ring all land exactly where RGTerm/IcyTERM put them) while the **text is mispositioned** (runs pin toward the left and overlap vertically).

The two paths differ: graphics use `rip.x/rip.y` via `scale_setpixel`; text uses `amiga_x/amiga_y` via `draw_glyph`/`scale_setpixel`. Only the text path is wrong, so the defect is in the text-cursor/glyph positioning, not the coordinate mapping itself.

The board positions **every** text run with command 19 (`ESC[19;x;y!`, "text cursor position in pixels", `do_skypix` case 19) immediately before it. Captured sequence from this board's login screen:

```
FONT Diamond.20   TEXTPOS(170,32)   "Reign of " "Fire " "BBS"
FONT Ruby.8       TEXTPOS(175,54)   "C-Net/5 Amiga SkyPix World Headquarters"
FONT Ruby.8       TEXTPOS(205,66)   "Located in Chattanooga Tennessee"
FONT Pearl.8      TEXTPOS(215,78)   "A Dotoran File Beta Site Since 1991"
FONT Times.15     TEXTPOS(85,95)    "JoinLink - Messages - Games - ..."
                  TEXTPOS(145,111)  "AmigaNet 39:902/518  *  ArakNet 10:100/1"
                  TEXTPOS(75,127)   "CommodoreNet 64:500/1  *  ..."
```

Each run has a distinct, increasing `y`, so correct case-19 handling stacks them without overlap (as RGTerm/IcyTERM do). The actual cause is a **parse/draw ordering** problem, in two parts:

1. **Text was buffered, positions applied inline.** `do_skypix` case 19 sets `amiga_x/amiga_y` immediately while parsing, but pass-through text is accumulated into `origbuf` and only drawn later when term.c flushes it through `cterm_ctputs` → `font_render`. When several `TEXTPOS + text` pairs fall in one network read, every run draws at the **last** pen position. Because it depends on read boundaries, some runs happen to land right and others don't. **Fix:** when `cterm->skypix && amiga_font`, draw each pass-through byte with `draw_glyph` inline (in stream order) instead of buffering it for the batched renderer.

2. **SkyPix `!` terminators were being deferred.** The main loop defers a `!` seen with `cb.len == 0` (RIP's bare-`!` introducer, so `cb_accept_introducer` runs post-flush). But a SkyPix terminator (`ESC[..!`) also arrives with `cb.len == 0` — mid-ESC-sequence (`cb.esc_len > 0`) — so it was deferred too, splitting a command from its following text across parse passes. **Fix:** gate that defer on `cb.esc_len == 0` so SkyPix terminators reach `cb_esc_feed` in the same pass.

With both applied, the login screen renders correctly — title centered, subtitles stacked, network list evenly placed — matching RGTerm/IcyTERM. Before/after screenshots and the two-part fix are attached (`syncterm-skypix-fixes.patch`, `screenshots/`). Verified against the live board at `call.rofbbs.com:6800` (choose **S** at the prefs prompt).

## 5. Line editing broke after a SkyPix screen (FIXED)

On font reset (`ESC[10;0!`) `do_skypix` freed `amiga_font` but left `cterm->font_render` pointing at the SkyPix renderer (the clear was commented out), so normal prompt text, **backspace**, and line deletion kept routing through the SkyPix path at the stale pen position. **Fix:** clear `cterm->font_render` (and `amiga_font`) on reset — even when the display mode is unrecognized — and in the inline-draw path only intercept printable text + CR/LF, letting control bytes reach cterm.

## 6. AREA FILL (command 3) was unimplemented (NOW IMPLEMENTED)

`ESC[3;m;x;y!` (flood/area fill) was a `printf("TODO")` stub, so any screen that outlines a shape and fills it (e.g. a board drawing big outline letters then filling them) stayed hollow. **Implemented** as a seed flood fill: read the seed pixel's colour from the rendered buffer, flood the connected matching region, replace with Pen A; coordinates via `map_rip_x/y`; a leak guard discards a run that exceeds ¼ of the viewport (a gap in the outline at screen scale would otherwise ooze into the background). Works on well-formed outlines.

## Known remaining issue (for the maintainers)

Some **script-heavy SkyPix screens render non-deterministically** — the same screen drawn twice can differ. Symptoms: positioned text labels occasionally run together, and area fills occasionally leak (an outline that is complete on one draw has a 1-pixel gap at screen scale on another). This looks like a **render-pipeline timing/coherence** issue (e.g. `getpixels` for the fill reading before line draws are flushed; or the SkyPix-font-active state depending on where a network read boundary falls), not a single command bug — and it needs iteration with visible pixels to pin down. A raw protocol capture of one such screen (C-Net's "ACIS supporter" screen: outline letters via `ESC[2`, then `ESC[3` fills, then `ESC[19`-positioned labels) is available on request.

---

### Summary for triage
Fixed locally and in the attached patch, verified against the live board:
1 & 2 — parser fixes (dedicated SkyPix setting; accumulate the font-name string). 3 — resolve fonts against a known dir + ship the canonical Amiga font set (provided). 4 — parse/draw-ordering (inline SkyPix text; don't defer SkyPix `!`). 5 — restore normal rendering on font reset (line editing). 6 — implement AREA FILL. Open: non-deterministic rendering on script-heavy screens (needs maintainer with visible-pixel iteration). Happy to revise any of these for upstream style.

## Test targets

`reign-of-fire-phonebook.lst` (attached, SyncTERM dialing-directory format, credentials stripped) has five Reign of Fire boards that between them exercise a wide range of terminal paths — handy for regression testing:

| Entry | Port | Mode | Exercises |
|---|---|---|---|
| Reign of Fire BBS | 6800 | ANSI | ANSI-BBS, codepage 437 |
| Reign of Fire II BBS | 6400 | C64 | PETSCII / Commodore 64 |
| Reign of Fire III BBS | 8502 | (current) | ANSI |
| Reign of Fire Skypix | 6800 | SkyPix | the full SkyPix path above (choose **S** at the prefs prompt) |
| Reign of Fire 64 Mode | 6800 | C64 | PETSCII over the Amiga board |

All resolve to `call.rofbbs.com`. The SkyPix entry is the one these fixes target.
