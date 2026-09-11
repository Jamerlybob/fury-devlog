---
title: "7. The Server Is In There. Someone Just Flipped It Off."
date: 2026-09-08T08:30:00+12:00
draft: false
tags: ["fury", "unreal-engine-3", "reverse-engineering", "decompilation"]
series: ["Reviving Fury"]
summary: "I started reading the decompiled arena code and found the server logic sitting right next to the client logic in the same files, fully written, wired to a switch that's soldered to 'off'."
ShowToc: true
---

Last post the decompiler finally dumped the whole game to disk: about 2,000 files
of readable UnrealScript. This post is me actually starting to *read* it, looking
for how a match server was meant to work.

I found the server. It's just... not plugged in.

## One codebase, two jobs

Here's a thing about Unreal Engine games that took me a while to get. The client
and the server are not separate programs. They're the same program, compiled
from the same code, and a launch option decides which one you are today. Playing
or hosting is a mode, not a download.

So when I decompile `Fury.exe`'s game code, the match-server logic should be in
there somewhere, mixed in with the client logic. I knew that in theory. What I
didn't expect was how *literally* it's mixed in.

Take the class that sets up an arena match. It's called `GOGameInfo`. One class,
and inside it there's a function that runs when a match starts:

```
event InitGame(string Options, out string ErrorMessage)
{
    ...
    if (false)
    {
        ServerInitGame(Options);           // <- everything a server does to boot
    }
    if (true)
    {
        GetClientSystems().Initialise();   // <- everything a client does to boot
    }
}
```

Read that again. `if (false)`. The entire server startup, spawning the player
manager, opening the database layer, starting the match timers, is right there in
the file, as a real function with a real body. And the code that decides whether
to call it is the literal word `false`.

<svg viewBox="0 0 640 300" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Diagram: one class named GOGameInfo with two code paths leaving it. The client path is open and lit. The server path is drawn as a real, complete road that ends at a bricked-up doorway labelled 'if (false)'.">
  <style>
    .lbl { font: 14px ui-sans-serif, system-ui, sans-serif; fill: #e6efe8; }
    .lbl-dim { font: 13px ui-sans-serif, system-ui, sans-serif; fill: #9dc4b2; }
    .box { fill: none; stroke: #5fe0a0; stroke-width: 2; }
    .box-dim { fill: none; stroke: #9dc4b2; stroke-width: 2; stroke-dasharray: 5 4; }
    .road { stroke: #5fe0a0; stroke-width: 2; fill: none; }
    .road-dim { stroke: #9dc4b2; stroke-width: 2; fill: none; }
    .brick { stroke: #9dc4b2; stroke-width: 1.5; fill: none; }
  </style>
  <rect class="box" x="250" y="24" width="140" height="44" rx="6" />
  <text class="lbl" x="320" y="51" text-anchor="middle">GOGameInfo</text>

  <path class="road" d="M290 68 L150 150" />
  <path class="road-dim" d="M350 68 L490 150" />

  <rect class="box" x="40" y="150" width="220" height="60" rx="6" />
  <text class="lbl" x="150" y="176" text-anchor="middle">client startup</text>
  <text class="lbl-dim" x="150" y="196" text-anchor="middle">if (true): runs</text>

  <rect class="box-dim" x="380" y="150" width="220" height="60" rx="6" />
  <text class="lbl-dim" x="490" y="176" text-anchor="middle">ServerInitGame()</text>
  <text class="lbl-dim" x="490" y="196" text-anchor="middle">full function, real body</text>

  <path class="road-dim" d="M490 210 L490 250" />
  <rect class="brick" x="430" y="250" width="120" height="34" />
  <line class="brick" x1="430" y1="267" x2="550" y2="267" />
  <line class="brick" x1="460" y1="250" x2="460" y2="267" />
  <line class="brick" x1="520" y1="250" x2="520" y2="267" />
  <line class="brick" x1="445" y1="267" x2="445" y2="284" />
  <line class="brick" x1="505" y1="267" x2="505" y2="284" />
  <text class="lbl-dim" x="490" y="300" text-anchor="middle">if (false)</text>
</svg>

<span class="caption">The road is fully built. The door at the end is bricked shut with one word.</span>

And it's not a one-off. Every place the code forks between "do the server thing"
and "do the normal client thing", same pattern:

| When the game... | Client build runs | Sitting right next to it, switched off |
|---|---|---|
| starts a match | client startup | `ServerInitGame` (player manager, DB layer, match timers) |
| logs a player in | stock engine login | `GODevLogin` / `GOLogin` (the session-key check I wrote about before) |
| finishes logging them in | stock engine | `GOPostLogin` (load their character, put them on a team) |
| a player leaves | stock engine | `GOLogout` |
| every tick | nothing | `ServerTick` |

The server half of *Fury's* arena is not missing. It was never deleted. It's in
the box I already have, fully assembled, with the power switch glued to off.

## So what flipped the switch

A build-time constant. Somewhere in the code there's a `const` (probably more
than one, but there's a dominant one) that means "this is a client build". Auran
set it when they compiled the copy that shipped to players. The compiler sees
`if (SOME_CONSTANT)`, works out the constant is false for this build, and bakes
the literal word `false` straight into the compiled bytecode. The constant's
name doesn't survive that. All I get to see is the `false` it left behind.

The server build of the exact same files would have that constant the other way,
and every one of those bricked doors would be open.

Can I just flip it myself? Here's where I have to correct something. My first
instinct, and what I said in an earlier draft, was "no, flipping it means
recompiling and I don't have the compiler". That's not right, and a UE3 modder
would have called it out. In Unreal's bytecode, "true" and "false" are two
neighbouring one-byte instructions. Swapping one for the other doesn't change the
size of anything, so nothing downstream shifts. In principle it's a single-byte
edit in a hex editor, and my decompiler already prints the exact file offsets.

There's one real catch, and it's an honest unknown. *Fury's* version of the file
format has some extra 4-byte values bolted onto each record that look like
checksums (I hit them while getting the decompiler working). If flipping that
byte makes a checksum wrong, the game might reject the file until I recompute it,
and I don't know yet whether those checksums even cover this part. So: maybe a
one-byte patch, maybe a one-byte patch plus some checksum math. Either way, not a
recompile.

Which actually makes this the *interesting* obstacle. The other reason the
shipped game can't host a server, the one from post 4 where it crashes if
you ask it to, lives down in the C++ I can't touch. This one lives in the script,
and script I can read and, it turns out, maybe poke.

Still, none of that changes the plan. Whether or not the byte flips, what I've
got now is a detailed, trustworthy reference for the server I have to build
anyway. I'm no longer guessing what the server did. I can read it.

[SIDE NOTE] This does add a second lock to an idea a few people suggested: just
run the shipped client as a server. The first lock (it crashes when asked to
host) was covered in post 4. Even past that, the match-setup code
would hit this `if (false)` and run the client path. Two independent doors on the
same corridor.

## What's behind the door

I went through the server-only functions to see how a character actually gets
into a match. Rough shape:

The master server hands the arena a little packet, basically "player 4021, on the
blue team, spawn them at the north gate". That packet is *identity and
placement only*. It does not include the character. No gold, no gear, no
abilities.

Then the arena server goes and fetches the character itself, from the database,
by ID. And it does it as a little assembly line: it queues up about a dozen
jobs, one per slice of the character, personal options first, then the core data
row, then abilities, the current persona, inventory, equipped items, awards,
factions, and so on. Each job is one call to a stored procedure in the database.
The core-data one, `AVA_GetAvatar`, comes back with 28 separate fields just for
the basics: name, currencies, four kinds of "essence", subscription level, which
face and hair you picked, your rest-gold timer, your rating.

I pulled every database call out of the code. In the game's own code there are
142 distinct stored procedures, and another 29 that only the map editor uses, so
call it north of 170. Buying from a vendor, placing an auction bid, sending mail,
repairing a sword, unlocking a faction rank, banning a player, every one is a
named procedure with a typed list of arguments. No loose SQL anywhere. That's
the friendliest possible thing to find, because that list of names with their
arguments is close to a spec for the database I need to stand up.

## Where this leaves me

Still in the reading phase, and it's paying off. Before this week the server was
a black box I was going to have to reinvent. Now it's a box with the lid off. I
can see the arena's startup sequence, its login handshake, the exact order it
loads a character in, and the full list of things it asks the database for.

Not written yet: what the other big class, the player controller, does on the
server side (it has 17 more of these switched-off branches I haven't read), and
where a couple of the fringe database groups get called from. And the next real
job is the 197 data tables, the ones holding every ability and item and vendor
in the game. Those are a different format and I haven't cracked them yet.

