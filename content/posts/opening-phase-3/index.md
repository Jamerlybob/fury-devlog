---
title: "9. Time to Build the Server. Nobody Kept One."
date: 2026-09-08T16:30:00+12:00
draft: false
tags: ["fury", "reverse-engineering", "networking", "unreal-engine-3"]
series: ["Reviving Fury"]
summary: "Reading the game is done. Now I have to build the thing that's actually missing. First I went shopping for someone else's Unreal Engine 3 server to build on. Every one I found cheats in a way Fury won't let me."
ShowToc: true
---

For the last week or so the job has been *reading*. Decompile the game, work out
the login flow, the character loading, the 197 data files, the server code
sitting switched off inside the client. That's done. Phase 2 is closed.

So now the reading turns into building. Phase 3: **a minimal arena server.** One
match, one map, running on my own machine, that an unmodified copy of Fury will
connect to. If I can get a character standing in a level that *my* program is
running, Phase 3 is done.

## Quick recap: client and server

A game like Fury is two programs. The **client** is the one you run: it draws the
world, reads your mouse, plays the sounds. The **server** is the referee: it holds
the real state of the match, decides what actually happened when two people swing
at each other, and tells every client what to draw. Take the server away and the
client is a very pretty puppet with nobody's hand in it.

## Why not just switch the server code back on?

Last few posts found the server logic *right there* in the client, switched off.
So the obvious thought is: flip it back on. Done. Three reasons it isn't:

1. **The client refuses to be a server.** Every "you be the server" door is welded
   shut (post 4). Even with perfect server code inside, there's no way to start it
   in server mode.
2. **The off switch is baked in.** The compiler hard coded "false, skip this bit"
   into the finished program. Possibly a one byte hex edit, possibly plus some
   checksum maths, and even then I hit reason one.
3. **The interesting bits aren't readable.** The server script I *can* read calls
   out to "talk to the database" and "send this over the network", and those are
   compiled C++. Any server I build has to supply its own versions.

So: write a new program from scratch that does what the switched off code
describes. The decompiled script is the blueprint. I just have to be the one
saying the lines.

## The option I wish I had

There's a way this phase mostly evaporates, and I can't have it. Studios that
licensed Unreal Engine 3 got the whole thing: full C++ source, and the tools to
build a proper dedicated server. With that I could flip the server back on *in the
source* and hit compile.

It was never public. It went to paying studios under NDA and nowhere else. Tony's
blessing covers what Auran made, not Epic's engine, which was never Auran's to
hand out. The one thin thread: Tony floated asking ex staff to dig through their
cupboards for old DVDs. Could be nothing. Could be a server build. It stays parked
until I've got something worth showing him.

## Shopping for someone else's server

Before writing mine from scratch, I went looking for one somebody else had already
written for *another* Unreal Engine 3 game. There's a healthy little revival
scene:

- **Tribes: Ascend** has [taserver](https://github.com/Griffon26/taserver), a
  rewrite of its login and server list.
- **Unreal Tournament 3** has a community replacement master server.
- **Blacklight: Retribution** has [BLRevive](https://gitlab.com/blrevive).
- **Mass Effect 3** multiplayer has [Pocket Relay](https://github.com/PocketRelay/Server).
- **APB: Reloaded** has [rAPB](https://github.com/hedger/rAPB).
- **TERA** has [Almetica](https://github.com/almetica/almetica) and friends.

Real, working projects. People play these games today because of them. Hats off.

## The catch

Every single one cheats the same way, and it's a way Fury already told me I can't.

They don't rebuild the referee. They rebuild the *lobby around* the referee, and
let the **shipped game** be the referee, by running the game's own server mode.
BLRevive's docs say it out loud: the match runs on *"the replication server which
comes with UE3 by default"*, started by literally running the game's `.exe` with
the word `server` and a map name.

The exact command that, on Fury, loads a map called "server" and crashes.

<svg viewBox="0 0 640 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Diagram: other UE3 revivals reuse the shipped game's built in server mode; Fury has no server mode, leaving a gap the community project has to fill itself.">
  <style>
    text { font: 13px system-ui, sans-serif; fill: #e6efe8; }
    .lbl { font-size: 12px; fill: #9dc4b2; }
    .box { fill: none; stroke: #5fe0a0; stroke-width: 1.5; }
    .ghost { fill: none; stroke: #9dc4b2; stroke-width: 1.5; stroke-dasharray: 5 4; }
    .arr { stroke: #9dc4b2; stroke-width: 1.5; marker-end: url(#a); }
  </style>
  <defs>
    <marker id="a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto">
      <path d="M0,0 L10,5 L0,10 z" fill="#9dc4b2"/>
    </marker>
  </defs>
  <text x="20" y="24">Other UE3 revivals</text>
  <rect class="box" x="20" y="36" width="180" height="46" rx="4"/>
  <text x="34" y="56">Community backend</text>
  <text class="lbl" x="34" y="72">accounts, server list</text>
  <rect class="box" x="20" y="110" width="180" height="46" rx="4"/>
  <text x="34" y="130">Shipped game, server mode</text>
  <text class="lbl" x="34" y="146">the referee, built in</text>
  <line class="arr" x1="110" y1="82" x2="110" y2="108"/>
  <text x="230" y="118" fill="#5fe0a0">= works</text>

  <text x="360" y="24">Fury</text>
  <rect class="box" x="360" y="36" width="180" height="46" rx="4"/>
  <text x="374" y="56">Community backend</text>
  <text class="lbl" x="374" y="72">I have to build this</text>
  <rect class="ghost" x="360" y="110" width="180" height="46" rx="4"/>
  <text x="374" y="130" fill="#9dc4b2">Shipped game, server mode</text>
  <text class="lbl" x="374" y="146">welded shut (post 4)</text>
  <line class="arr" x1="450" y1="82" x2="450" y2="108"/>
  <text x="360" y="182" fill="#9dc4b2">the referee: nobody's. I build this too.</text>
</svg>

<span class="caption">Everyone else reuses the shipped game as the referee. Fury's is welded shut, so I build that part as well.</span>

Two projects did aim at the referee itself, and they miss for opposite reasons.
**rAPB**, years in, still has this line in its README:

> You cannot enter districts yet.

A "district" is APB's version of a match. After all that work, the part they
can't do yet is exactly the part I need to do *first*. Not a knock on them. It's
the clearest possible sign this is genuinely hard.

**TERA**'s emulators are complete from scratch servers, which is proof it can be
done. But TERA's studio ripped out Unreal's networking and bolted on their own,
so those emulators speak a language that has nothing in common with Fury's. Proof
of concept, zero lines I can reuse.

## What I can still use

Not code, but not nothing. Epic's free **UDK** ships the *script* for the engine's
own networking layer, the half that's invisible C++ inside Fury. So I can read how
a UE3 client and server say hello from Epic's side and line it up against Fury's
decompiled script. Unreal Tournament 3's script helps the same way, for telling
"this is just how UE3 does it" apart from "this is a thing Auran changed".

## The decision

No shortcut. I write the server from scratch, in C#. One reason: my decompiler is
already C# and already understands Fury's exact file format, and the network
protocol reuses big chunks of that format. The blueprint is Fury's script plus
Epic's. The typing is all mine.

[SIDE NOTE] one thing I checked while I was here. Those extra four bytes Auran
bolted onto every file header (post 6)? They look like a checksum but don't match
any standard checksum recipe I threw at them. No idea what they're for. Remember
this number. It comes back, and it costs me two days.

First real job: point an unmodified client at a port with nothing on it and
listen to what it says when it tries to join a match.
