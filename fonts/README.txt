===========================================================================
  SkyPix Terminal Font Collection
  Canonical Amiga SkyPix bitmap fonts for SkyPix-capable terminals
  Assembled by Reign of Fire BBS (https://www.rofbbs.com) for the terminal
  authors of SyncTERM, RGTerm, and IcyTERM and the wider BBS community.
===========================================================================

WHAT THIS IS
------------
SkyPix is the Amiga graphics protocol used by C-Net/Amiga BBSes. When a
board sends a SkyPix "Set Font" command it names an Amiga font (e.g.
"Courier.font") at a point size; the TERMINAL is expected to have that
font and render with it. A terminal that is missing a requested
font+size cannot display that part of the screen correctly.

This archive collects the canonical SkyPix font faces at the sizes boards
actually request, in the two formats terminals use today.

LAYOUT
------
  amiga-font/   Amiga native font format: a "<name>.font" descriptor plus
                a "<name>/" directory of per-size bitmap files (the Amiga
                FontContents + TextFont on-disk format). This is what
                SyncTERM's SkyPix loader reads, and the original form of
                these fonts. Load relative to this directory: the loader
                opens "courier.font", reads the size table, then opens the
                sized glyph file it names (e.g. "courier/24").

  rgterm-spf/   RGTerm's ".SPF" format (SPF2): a compact per-glyph bitmap
                format (8-byte header: height, first-char, baseline,
                glyph-count; then width+bitmap per glyph). This is what
                RGTerm loads. Included here for RGTerm users and as the
                reference source for the three sizes not yet available in
                Amiga .font form (see GAPS).

  syncterm-skypix-report.md    Write-up of six SyncTERM SkyPix bugs found
                against this board, each with cause and fix (all fixed /
                implemented locally), plus one documented non-deterministic
                edge case for the maintainers.
  syncterm-skypix-fixes.patch  The functional fixes as a unified diff
                against src/syncterm (apply with `patch -p1`).
  reign-of-fire-phonebook.lst  Five Reign of Fire dialing entries
                (SyncTERM list format, credentials stripped) covering ANSI,
                PETSCII/C64, and SkyPix - a quick terminal test matrix.
  screenshots/  This board in SyncTERM vs RGTerm vs IcyTERM: before/after
                the fixes, plus welcome, Emberwood, message-base, the filled
                "spyder" ellipse, and the area-fill known-issue screen.

FACE / SIZE COVERAGE (amiga-font)
---------------------------------
  courier   11 13 15 18 24 30 36
  times     11 13 15 18 24 30 36
  helvetica 9 11 13 15 18 24
  diamond   12 20
  emerald   17 20
  ruby      8 12 15
  garnet    9 16
  opal      9 12
  sapphire  14 19
  topaz     11

GAPS (present in rgterm-spf, not yet in amiga-font)
---------------------------------------------------
  sapphire 15, sapphire 18, topaz 8
These exist in this archive only as .SPF. Converting SPF2 -> Amiga
TextFont (a relocatable Amiga hunk binary) is straightforward in
principle but should be verified visually against a real board before
publishing; left as a TODO for anyone who can see the rendered output.

PROVENANCE
----------
The Amiga .font files here are the standard SkyPix-era Amiga bitmap
fonts (gathered from a ClassicWB Workbench font drawer and a live
C-Net/Amiga system). The .SPF files are from the RGTerm distribution.
Topaz is Commodore-Amiga's system font.

LICENSING / REDISTRIBUTION
--------------------------
These are decades-old Amiga bitmap fonts distributed freely with
SkyPix/Skyline-era BBS software. In the United States, copyright does
not protect typeface DESIGNS or bitmap font DATA (only scalable font
PROGRAMS), so bitmap fonts of this kind are generally free to copy and
redistribute. They are provided here for free use and redistribution by
SkyPix-capable terminal software and their users. This note is a
good-faith summary, not legal advice; the files are offered as-is with
provenance stated so each project can satisfy itself before bundling.

CONTACT
-------
Reign of Fire BBS - https://www.rofbbs.com
Questions / additions welcome; the goal is one complete font set so
every SkyPix terminal renders boards identically.
===========================================================================
