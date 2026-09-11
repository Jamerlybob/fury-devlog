---
title: "12. The Answer Key I Won't Open"
date: 2026-09-10T12:30:00+12:00
draft: false
tags: ["fury", "reverse-engineering", "unreal-engine-3", "decisions", "ethics"]
series: ["Reviving Fury"]
summary: "Someone pointed me at a shortcut that would save weeks: the leaked source code for the engine Fury runs on. Here is why I am not touching it, and the clean-room trick I looked at and turned down."
ShowToc: true
---

## Where I'm at

I am rebuilding the server from scratch, and the hard part right now is working out
the exact language the client and server spoke to each other in. I have been
doing that by taking the client apart byte by byte in a disassembler. It is
slow. Post 11 was a whole day of it.

## Discord replied

I have been hanging around a Discord for people who pull apart Unreal Engine
games, built around a tool called UELib that reads Unreal's data files. I lean on a modified version of it constantly. It is
how I got Fury's game code out into readable form back in Phase 2. 

> Perhaps mir-eternal could be of use, it's also a UE3 MMO, could be of use if
> you're going down the hard route of re-implementing the server. 

## The first thing: another dead MMO, brought back

mir-eternal is a rebuilt server for a different dead Unreal Engine 3 MMO, a
Chinese one called Legend of Mir 3D. Same shape of project as mine: game's dead,
someone rebuilt the server in C#, people play on it now. So of course I went
digging, hoping for code I could learn from or lift.

It doesn't help, and the reason is kind of interesting. Unreal Engine comes with
its own built-in system for keeping a client and server in sync, a big piece of machinery. The Mir team threw all of it out and replaced it with
their own simpler messaging system: hand-numbered messages with fixed
layouts, nothing like what Unreal does out of the box. That is a perfectly
reasonable thing to do when you are starting fresh. But it means their server and
Fury's server have almost nothing in common. The one part I
actually need help with is the part they deleted.

Still, good to see another near-solo revival that actually works. And it ships a
copy of UELib inside it (Eliot, who hangs out in that Discord, wrote it), same lineage as the one I use.

## The second thing: the shortcut

Looking into it, the Mir game is "very close to a leak of GoW". GoW is Gears of War, a big 2006 game. Years ago the source code
for its engine leaked onto the internet. Source code is the original,
human-written version of a program, the readable form with names and comments,
before it gets crushed down into the thing a computer actually runs. And that
particular engine's source is, near enough, Unreal Engine 3 itself.

Fury runs on Unreal Engine 3. So that leaked code is basically a readable copy of
the exact machinery I have spent two weeks reconstructing by hand from a
disassembler. The part I am stuck on, the precise format of the sync messages, is
sitting in that code as plain, commented, human-written source. Opening it would
turn my slow byte-by-byte guessing into looking up the answer.

I am not going to pretend that isn't tempting. It would genuinely save weeks, and
it would kill a big pile of "I won't know if this is right until I test it
against the real client" uncertainty stone dead. If this were only about
finishing fast, it would not be a hard call.

## Why not

**It isn't mine to use.** Fury is a special case. I emailed the person who made
it and he gave me his blessing to do whatever I can with the game.
That covers Fury. It does not cover the engine Fury was built on. That engine is
a different company's code, it leaked without permission, and nobody said I could
have it. "It's on the internet" is not the same as "it's mine to build on".

So I keep going the way I have been.

## The clean-room idea, and why it doesn't fit

There is a well-known trick for exactly this situation.

It is called a clean room, or a Chinese wall. You split the work between two
people who are not allowed to compare notes. Person A reads the forbidden code
and writes a plain-English description of what it does, being careful to write
down only *what* it does, never *how* it is written. Person B never sees the
original at all. B builds a fresh version working only from A's description.
Because B never touched the protected code, B's version comes out clean, even
though the understanding ultimately traces back to A reading the original.

<svg viewBox="0 0 700 250" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="A clean room split. On the left the leaked code goes to person A, who writes a plain description. The description sits on a wall in the middle. On the right, person B reads only the description and writes a clean new version, never seeing the original.">
  <defs>
    <marker id="wall-ah" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" fill="#9dc4b2"/>
    </marker>
  </defs>
  <line x1="350" y1="8" x2="350" y2="242" stroke="#9dc4b2" stroke-width="2" stroke-dasharray="6 5"/>
  <text x="175" y="20" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#9dc4b2">allowed to see the original</text>
  <text x="525" y="20" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#9dc4b2">never sees the original</text>

  <rect x="28" y="54" width="120" height="44" rx="6" fill="none" stroke="#5fe0a0" stroke-width="1.5"/>
  <text x="88" y="81" text-anchor="middle" font-family="sans-serif" font-size="12" fill="#e6efe8">the leaked code</text>

  <line x1="148" y1="76" x2="184" y2="76" stroke="#9dc4b2" stroke-width="1.5" marker-end="url(#wall-ah)"/>

  <rect x="186" y="46" width="140" height="60" rx="6" fill="none" stroke="#5fe0a0" stroke-width="1.5"/>
  <text x="256" y="70" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#e6efe8"><tspan x="256" dy="0">person A writes</tspan><tspan x="256" dy="14">down what it does,</tspan><tspan x="256" dy="14">in plain words</tspan></text>

  <line x1="256" y1="106" x2="322" y2="142" stroke="#9dc4b2" stroke-width="1.5" marker-end="url(#wall-ah)"/>

  <rect x="298" y="142" width="104" height="42" rx="6" fill="none" stroke="#5fe0a0" stroke-width="1.5"/>
  <text x="350" y="168" text-anchor="middle" font-family="sans-serif" font-size="12" fill="#e6efe8">the description</text>

  <line x1="402" y1="160" x2="452" y2="122" stroke="#9dc4b2" stroke-width="1.5" marker-end="url(#wall-ah)"/>

  <rect x="454" y="46" width="150" height="60" rx="6" fill="none" stroke="#5fe0a0" stroke-width="1.5"/>
  <text x="529" y="70" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#e6efe8"><tspan x="529" dy="0">person B reads</tspan><tspan x="529" dy="14">only the description,</tspan><tspan x="529" dy="14">never the code</tspan></text>

  <line x1="529" y1="106" x2="529" y2="150" stroke="#9dc4b2" stroke-width="1.5" marker-end="url(#wall-ah)"/>

  <rect x="454" y="150" width="150" height="44" rx="6" fill="none" stroke="#5fe0a0" stroke-width="1.5"/>
  <text x="529" y="177" text-anchor="middle" font-family="sans-serif" font-size="12" fill="#e6efe8">a clean new version</text>
</svg>

A clean room split: one person turns the forbidden code into a plain
description, and a second person, who never sees the original, builds from that
description alone. It is roughly how the first IBM PC clones got made in the
eighties, which is a big part of why PCs got cheap.

It is a real technique with real history. It works. It does not work here,
though, for a few reasons stacked on top of each other.

I am one person haha. The wall needs two isolated people and I would be both of them.
You cannot read something and then un-read it. Once I have seen the leaked code,
every line of my server is written by someone who has seen it. There is no wall,
there is just me on both sides pretending.

The obvious "fix", have the AI read it and hand me a description, is worse, not
better. I could not trust that the separation held. The description would be
shaped by the exact code it came from, in ways neither of us could see or check.
A clean room only means something if you can point at the wall and show it is
solid. "I promise the AI paraphrased it enough" is not a wall.

And underneath all of that: even done perfectly, a clean room is a way to be
*legally* safe while still building on something you took. The code still leaked.
Routing it through a description first does not make it mine. It is a grey area
at best, and I do not want the project standing in a grey area when it does not
have to.

## The one shortcut I would actually take

For completeness. When I first spoke to Tony, who made Fury, he floated the idea
of asking ex-Auran staff to dig through their cupboards for old DVDs. Old builds,
maybe server bits, who knows.

If that ever happens, that is completely different and I would take it in a
heartbeat. That is the person with the rights to the thing handing it over. No
wall required, nothing grey about it. But it is a "maybe, someday" and I am not
building the plan around a maybe. If a box of DVDs turns up, great. Until then it
does not exist.

## Next

Nothing actually changed today, which is sort of the point. Back to where I was:
writing the code that packs each type of value into the sync messages in the
exact layout the disassembler showed me. The slow way, on purpose.

