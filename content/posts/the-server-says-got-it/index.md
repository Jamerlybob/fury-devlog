---
title: "The Server Says Got It"
date: 2026-09-14
draft: true
tags: ["fury", "unreal-engine-3", "networking"]
series: ["Reviving Fury"]
summary: "Fury was sending two movement updates at a time because my server never told it that the first one arrived."
ShowToc: true
---

The character could walk, but the network traffic looked like someone leaning
on a doorbell.

Fury was sending roughly two movement updates in nearly every message. That
wasn't a special movement mode. It was a backlog. My server received each move,
printed it, and then sat there in complete silence, so the client bundled the
old move with the new one and tried again.

There are actually two different ways the server has to say "got it". One
confirms that the network packet arrived. The other calls a function inside the
client named `ClientAckGoodMove`, handing back the timestamp of the newest move
the server accepted.

```text
client:  ServerMove(t=1.24) + ServerMove(t=1.29)
server:  packet received
server:  ClientAckGoodMove(t=1.29)
client:  clears everything up to 1.29
```

The old capture had 2,278 `DualServerMove` calls and seven plain
`ServerMove` calls. After I added both acknowledgements, I ran the untouched
client again. It sent 1,191 plain moves and zero dual moves. Nothing malformed,
nothing piling up behind them.

The server still isn't simulating those moves itself. That is a much larger
job involving collision, physics and position corrections. For now it can read
what the real client says, prove that every parameter ended on the right bit,
and stop asking the client to repeat itself forever.

Next I need to make sure the same server process can survive a disconnect and
let the client back in. One conversation at a time.

That plan got bumped by a more important question: can I put the character in
the actual arena instead of the little holding platform where matches start?

The map has four official player starts. Every one of them is tagged
`Deathschool`, and the match start code doesn't move the player somewhere else.
It just changes the match from the warmup phase to the fighting phase. Useful,
but not much help when I'm still missing most of the objects that run a match.

I went back into the map file and used one of its navigation points instead.
Those are spots the original developers placed for characters to walk through,
so it gives me a real coordinate without making one up and hoping there's a
floor under it.

The first unattended run sat there for a full minute. The client reported the
map coordinate plus exactly 30 units for the character's collision capsule,
3,019 times in a row. No falling through the world. No network tantrum. I still
need to stand at the keyboard and make sure this particular spot is actually in
the playable arena and isn't tucked behind a decorative tomb, but at least it
has a floor. Standards are moving quickly around here.
