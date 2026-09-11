---
title: "8. 197 Data Files, One Format Nobody Wrote Down, and Half the Numbers Missing"
date: 2026-09-08T09:00:00+12:00
draft: false
tags: ["fury", "unreal-engine-3", "reverse-engineering", "file-formats", "data"]
series: ["Reviving Fury"]
summary: "The client ships every ability, item, vendor and map in a custom binary format with no docs. I cracked it from a hex dump in an afternoon, and then noticed that a huge chunk of the tables aren't in the box at all."
ShowToc: true
---

## Where I left off

Catch-up in three sentences. I'm rebuilding the server for *Fury*, a PvP game
that pulled the plug on its servers in 2008. All I have is the client, the
program that ran on players' PCs, and over the last few days I've turned its
compiled game code into ~2,000 files of readable script. Last post I found the
server logic sitting *inside* that code, switched off.

Code is only half a game though. The other half is data: how much a fireball
costs, what's in a shop, how big each map is. In Fury that lives in a folder
called `Content/Export/Client/`, and it's 197 files with a `.bin` extension and
no documentation anywhere.

Time to open them.

## What's actually in the box

So a `.bin` file is just "bytes, format unspecified", and the extension tells
you nothing. Could be anything. The first thing you do is dump one as hex and stare
at it. Here's the smallest interesting one, `ABI_DamageTypes.bin`, all 87 bytes:

```
00000000: 1100 0000 2a00 0000 0500 0000 0103 0208  ....*...........
00000010: 0800 0000 0000 0109 0000 0002 1200 0000  ................
00000020: 031b 0000 0004 2400 0000 00e7 1d00 00a4  ......$.........
00000030: 2600 0001 e81d 0000 a526 0000 02e9 1d00  &........&......
00000040: 00a6 2600 0003 ea1d 0000 a726 0000 04e6  ..&........&....
00000050: 1d00 00d6 1f00 00                        .......
```

That left column is the position in the file, the middle is the raw bytes in
hex, the right is those same bytes as text (dots where the byte isn't a
printable character). Almost none of it is text, so this isn't a config file
someone typed. It's a *structured* format, packed by a program, for a program.

The way you read one of these is you find the numbers that are obviously
counts or sizes and see if the rest of the file lines up behind them. First
twelve bytes here are three 4-byte numbers: **17**, **42**, **5**. The file is
87 bytes long. 42 is a position *inside* it. And if you jump to byte 42, the
rest of the file is exactly five identical 9-byte chunks. Five rows. It's a
table.

<figure>
<svg viewBox="0 0 720 300" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Diagram of the GOTXT file layout: a 12-byte header, a short schema, a skippable index, then fixed-size rows.">
  <style>
    .lbl  { font: 13px/1.4 ui-sans-serif, system-ui, sans-serif; fill: var(--content, #2b3a2b); }
    .lbls { font: 11px ui-monospace, monospace; fill: var(--content, #2b3a2b); }
    .box  { fill: none; stroke: var(--primary, #3f6f3f); stroke-width: 1.5; }
    .fillA { fill: color-mix(in srgb, var(--primary, #3f6f3f) 12%, transparent); }
    .fillB { fill: color-mix(in srgb, var(--primary, #3f6f3f) 22%, transparent); }
    .fillC { fill: color-mix(in srgb, var(--primary, #3f6f3f) 6%, transparent); }
    .tick { stroke: var(--primary, #3f6f3f); stroke-width: 1; }
  </style>
  <rect class="box fillB" x="20" y="40" width="150" height="46"/>
  <text class="lbls" x="26" y="60">schemaEnd | dataOffset</text>
  <text class="lbls" x="26" y="76">| keyCount   (3 × u32)</text>
  <text class="lbl"  x="20" y="30">header (12 bytes)</text>

  <rect class="box fillA" x="170" y="40" width="90" height="46"/>
  <text class="lbls" x="176" y="60">colCount +</text>
  <text class="lbls" x="176" y="76">type bytes</text>
  <text class="lbl"  x="170" y="30">schema</text>

  <rect class="box fillC" x="260" y="40" width="150" height="46"/>
  <text class="lbls" x="266" y="63">index (skip it,</text>
  <text class="lbls" x="266" y="79">it's redundant)</text>
  <text class="lbl"  x="260" y="30">lookup table</text>

  <rect class="box fillB" x="410" y="40" width="290" height="46"/>
  <text class="lbls" x="416" y="63">row · row · row · row · row …</text>
  <text class="lbls" x="416" y="79">columns packed back-to-back, no gaps</text>
  <text class="lbl"  x="410" y="30">the actual data, read to end of file</text>

  <line class="tick" x1="70" y1="86" x2="70" y2="150"/>
  <line class="tick" x1="70" y1="150" x2="470" y2="150"/>
  <line class="tick" x1="470" y1="150" x2="470" y2="88"/>
  <polygon points="470,86 465,96 475,96" fill="var(--primary, #3f6f3f)"/>
  <text class="lbl" x="150" y="170">“dataOffset” in the header points straight at the first row</text>

  <rect class="box fillA" x="410" y="210" width="60"  height="34"/>
  <rect class="box fillA" x="470" y="210" width="110" height="34"/>
  <rect class="box fillA" x="580" y="210" width="110" height="34"/>
  <text class="lbls" x="420" y="231">byte</text>
  <text class="lbls" x="486" y="231">int32</text>
  <text class="lbls" x="596" y="231">int32</text>
  <line class="tick" x1="410" y1="210" x2="410" y2="244" stroke-dasharray="3 3"/>
  <line class="tick" x1="410" y1="244" x2="690" y2="244" stroke-dasharray="3 3"/>
  <text class="lbl" x="410" y="264">one row = one column per “type byte” in the schema</text>
</svg>
<figcaption>the whole format on one napkin. header, tiny schema, a lookup table you can ignore, then rows until EOF.</figcaption>
</figure>

That's basically the entire format. A header, a one-line schema that says "this
table has 3 columns: a byte, then two 4-byte integers", a lookup table for fast
searching that turns out to be a copy of information already in the file, and
then the rows. Strings, when a table has them, are just written straight into
the row and capped with a zero byte: English as plain bytes, other languages
as two-bytes-per-letter.

Six column types in the whole set: yes/no, small number, big number, decimal,
text, foreign-language text. That's it. Whoever built this kept it lean.

## The part where it just works

I wrote a decoder, a script that reads the header, reads the schema, then walks
the rows. The test I trusted: run it on all 197 files and check that it lands
*exactly* on the last byte of every single one, no leftovers, no running off the
end. It did. First real try, after I fixed one wrong assumption about a
multi-part key. 197 for 197.

Then a bonus I wasn't expecting. Remember those 2,000 files of decompiled
script? Buried in them are definitions like this:

```
struct GOTXT_CMN_GameMap {
    int   gameMapID;
    string gameMapName;
    byte  mapMode;
    int   locGameMapNameID;
    ...
}
```

That's the game's own description of a row in `CMN_GameMaps.bin`, and it has the
**field names**. 24 of them, and they line up one-for-one with the 24 columns my
decoder found. So I could match every `.bin` table to its struct by shape and
borrow the names. 191 of the 197 now come out with real headers, like
`challengeCost`, `teamSizeRequirement`, `bCanDuel`, instead of "column 14".

The character-creation map, decoded, first rows:

```
gameMapID  gameMapName    mapMode  teamSizeRequirement  bCanDuel
0          AVA_Creation   0        1                    0
71         BB_Ambush      0        1                    0
60         BB_BuriedCity  0        1                    0
```

`BB` is Bloodbath. `AVA_Creation` is the character screen. There are 49 maps in
there, a couple marked `_DEPRECATED`, which is a nice little fossil: someone
retired a map and left the tombstone in.

[SIDE NOTE] the base English string table has entries like `CHANGE ME` still
sitting in it, and every string that never got translated has a placeholder in
the other-language columns that's just... the English run through a Pig Latin
generator. "ANGECHAY EMAY". Shipping game. 2008. I love it.

## The half that isn't here

Here's the thing I keep coming back to. The folder is called
`Content/Export/Client/`. **Client.** And the code that opens these files has a
line in it that picks between two folders:

```
if (server build)  path = "Content/Export/Server/";
else               path = "Content/Export/Client/";
```

There is no `Server/` folder in what I have. It was on their machines.

So I made a list. Every table the code knows how to load, minus every table
that's actually in the box. **87 tables** come up missing. And it's not
leftovers. Look at what they are:

- **All 39 combat-effect tables.** The client ships the *tooltip* for every
  ability, the words you read on hover. It does not ship the numbers behind
  them. How much a hit actually takes off, how long a stun lasts, how a shield
  stacks: `Server/` only.
- **The server's ability table has 80 columns. The client's version has 58.**
  The 22 it drops are exactly the combat-relevant ones: damage scaling, aggro,
  interrupt rules.
- **Every monster table.** Templates, spawns, patrol routes, the lot. The whole
  data side of PvE.
- The reward maths. Score-per-kill tables. Bot config.

I already knew from earlier digging that the *code* to run a match is in the
client. Turns out a lot of the *data* that code needs to feel like Fury,
specifically the entire feel of combat, never left Auran's building.

That's not a wall. Every one of those 87 missing tables still has its struct in
the decompiled code, so I know the exact shape of every one: what columns, what
types, what they're called. I know precisely what I have to rebuild. It's just
going to be a lot of it, and a lot of it will be me guessing numbers and tuning
until a fight feels right instead of reading them off a file.

Which, honestly, might be the most fun part.

## Next up

Phase 2, reading the client and cataloguing everything, is basically done. I've got
the code, the data, and a map of the server-shaped hole. Next is writing that
all up properly and then actually starting Phase 3: standing up a tiny arena
server from scratch and getting one unmodified client to connect to it and move
a character around. That's the first moment this stops being archaeology and
starts being a game again.

