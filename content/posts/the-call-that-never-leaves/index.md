---
title: "20. The Call That Never Leaves"
date: 2026-09-12T00:30:00+12:00
draft: true
tags: ["fury", "reverse-engineering", "networking", "unreal-engine-3"]
series: ["Reviving Fury"]
summary: "I fixed the clock from last post. Then I went looking for the one message the client refuses to send, and proved something I didn't expect: it's not lost in transit, it never leaves the building."
ShowToc: true
---

Last post I found out why the game was yelling "Connection Interrupted" at me:
my server never told the client what time it was, so its own internal clock
drifted off and tripped an alarm. Two things left over from that: a small fix
(figure out how the game actually writes a number called a "double" onto the
wire), and a much bigger, older question that's been sitting in my notes for a
couple of sessions now: why does the client never tell my server it's done
loading?

## The small one: teaching the wire a new number

Every value the game sends over the network gets packed into bits by a specific
bit of game engine code, one function per data type. I'd already figured out
how it does whole numbers and normal decimals (the ones programmers call
"floats"). But the exact number I needed for the clock fix is a "double", a
wider, more precise decimal, and I'd never seen the game write one of those
before. Rule I set myself ages ago: never guess a wire format, go and read the
actual compiled code that does it.

So I did. I pointed Ghidra (a free tool that turns compiled game code back into
readable, C-like pseudocode) at the exact function, and it turned out to be
almost insultingly simple: it's the *identical* code to the float version,
byte for byte, except one instruction changed from "copy 4 bytes" to "copy 8
bytes". No trickery, no downcasting to save space. I wrote the C# to match,
round-tripped a pile of test values through it (including the classic
double-precision troublemakers, `double.Epsilon`, `MinValue`, `MaxValue`) and
checked the raw bytes matched what .NET's own number library would produce.
All green.

Wired it into the server on a repeating timer (the game's own script re-arms
this every 15 seconds, so I copied that), ran it against the real client, and
watched my own log line go out:

```
actor channel 3 (PC): sent RPC ClientSetServerTime(DoubleValue { Value = 83.828 }, ...)
```

Small, satisfying fix. On with the actual mystery.

## The bigger one: a message that's been going missing for two sessions

Quick recap for anyone just joining: the game world now genuinely loads. The
loading screen drops, the arena renders, my character's HUD comes up. But the
instant that happens, the game is *supposed* to send one particular message
back to the server, basically "hey, I'm done loading, loading screen's gone."
It never arrives. I've known this for a couple of sessions and each time I've
written "not chased yet" and moved on to something more tractable.

Not this time. Here's the thing that made it worth cracking open: there is
exactly **one** function, in the entire multi-megabyte client executable, that
actually puts a reliable message onto the wire. Every single kind of network
traffic (chat, replicated data, remote function calls, all of it) funnels
through this one piece of code before it ever touches a socket. If I hook that
one function, I *cannot* miss the message, no matter which weird code path it
takes to get there.

So that's what I did. I wrote a script that attaches to the running client (via
Frida, a tool for injecting your own code into somebody else's already-running
program) and hooks that one send function, logging every single call it gets
plus a full stack trace of who called it. Then I ran a real session: server up,
client spawned, held it open for ninety seconds after the loading screen
dropped.

Result: 26 calls right at the start (the normal handshake chatter), then
**nothing**. Not one more call to that function for the entire ninety seconds,
right up until I forced the client to close and it sent one final "I'm
disconnecting" message on its way out. The client's own log file confirms it
genuinely did finish loading a few seconds in ("OnLoadingCompleteCheck",
"DisableLoadingScreen> 1"), and then it just... sat there. Fully rendered,
fully idle, saying nothing.

## What that actually rules out

Going in, my leading theory was that the "I'm done loading" call was sneaking
out through some other, harder-to-see path, maybe through a virtual function
table lookup that my earlier static analysis (reading the disassembly cold,
without running anything) couldn't trace back to its caller. That would've
been a real headache: it'd mean the call *was* leaving, just via a route I
hadn't found yet.

This experiment kills that theory outright. Since the hook sits on the actual
function body, it doesn't matter how the call gets there, direct call, virtual
dispatch, whatever. If the message left the process, I would have seen it. It
didn't. Which means the call isn't going missing on the way out: **something
is stopping it from being sent in the first place**, before it ever gets
anywhere near the networking code.

That's actually good news, in the annoying-progress kind of way. It narrows the
search from "somewhere in a massive networking stack" down to "somewhere in how
this one game object decides whether it's allowed to talk to the server at
all". My money's on something in how my shortcut version of the server sets up
that object not quite matching what a real server would do, so the client's own
bookkeeping doesn't think it has anywhere to send the message to. Next step is
to catch the object red-handed at the exact moment it tries, and read its own
fields to see what it thinks is missing.

## Where this leaves things

Clock's fixed (pending one more live check that the banner actually stays
gone). The bigger blocker, the one standing between "the arena renders" and
"I can actually move my character", is narrower than it was this morning but
still open. No manufactured ending here, it's genuinely unresolved. Next
session's job.
