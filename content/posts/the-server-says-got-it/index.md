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
