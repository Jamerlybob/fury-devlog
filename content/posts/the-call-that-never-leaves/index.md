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

## Catching it red-handed

So that was the plan: catch the object at the exact moment it decides not to
send anything, and read its own fields to see what it thinks is wrong.

First problem, I didn't actually know which of the game's thousands of live
objects to read, or where in memory to find the one field I cared about
("who's my controller"). Games don't ship with a helpful map saying "this
byte means that". I had to build a small tool that walks the game's own
internal bookkeeping, the same lookup table the game itself uses to figure
out which byte means what, and ask it "where does the property called
Controller live on this particular object".

I built it, ran it, and it told me it couldn't find the field at all. Not "the
field is empty", genuinely "I searched and there's nothing here called
Controller". That's the kind of result that should make you suspicious of
your own tool before you get excited about what it's telling you, so I added
a sanity check: ask the same tool to find a *different* field I already knew
the answer to from earlier work. It failed that one too, in a suspicious way
(the same object kept reporting the exact same number of fields at six
different, genuinely different points in its family tree, which isn't how
real data behaves). So the tool was lying to me. Good thing I checked.

Rather than debug that approach further, I switched to a technique from two
sessions ago that I already knew worked: instead of walking the game's static
blueprint of a class, watch the *live* lookup the game itself performs while
it's actually receiving data over the network, and borrow the answer it comes
up with. Same sanity check against the field I already knew, and this time it
came back exactly right.

With a trustworthy way to find the field, I read it, twice, on two separate
runs of the game. Both times: the field isn't empty. My leading theory was
dead on arrival, the "who's my controller" question, which I'd worried might
be pointing at nothing and crashing the function silently, genuinely has an
answer.

But then I read the two fields sitting right next to it, and this is where it
gets strange. Every actor in this game engine carries two flags that answer
"who's actually in charge of me, the server or the client I'm running on".
For the game's own controller, on the client's own screen, both of those
flags say "the server". Both of them. On the client. About its own local
copy of the thing it's supposedly a client's-eye view of.

That shouldn't happen, or at least, it doesn't match either of the two ways
I'd have expected it to go wrong. And there's a good reason to think it
matters: if this game object genuinely believes it already has full server
authority over itself, it would have no reason to ask permission before
doing something, which is exactly what "send a message asking the server to
acknowledge I'm done loading" is. It would just quietly do the thing locally
and never bother the network at all. That would explain the silence
perfectly. It's not proof yet, but it's the first theory this session that
actually fits every single piece of evidence I've collected so far.

## Where this leaves things

Clock's fixed (pending one more live check that the banner actually stays
gone). The bigger blocker, the one standing between "the arena renders" and
"I can actually move my character", took two real steps forward today: one
dead theory buried with actual evidence instead of a guess, and one new,
genuinely strange clue that fits everything I know so far. Still unresolved.
Next job is figuring out exactly where in the handoff from server to client
those two "who's in charge" flags are supposed to flip, and whether my
server needs to hand them over already flipped instead of trusting the
client to do it.

## Chasing the flags, and finding something else entirely

So that was the plan. I had a specific, concrete idea of *how* those two
flags might end up both saying "the server": my earlier reading said the
client applies a batch of properties one at a time, in a fixed order, and if
it stops partway through that batch for the controller object specifically,
it would land on exactly the wrong pair of values by accident. There's a
single spot in the client's code that would cause exactly that kind of
partial stop, and this time I could actually watch it happen live instead of
reading cold disassembly and guessing.

I hooked that one spot, both the check itself and the place execution lands
if it fires, and ran a full session: server up, client spawned, all four of
my game objects opened and replicated, right through the loading screen
drop. Then I watched the hook trace.

It never fired. Not once, on any of the four objects, across their opening
messages or their follow up updates. Clean theory, wrong theory. That's the
second dead end this thread has produced, and honestly the more satisfying
kind: I built the exact instrument needed to catch it red-handed, and it
came back with a clear no instead of an ambiguous maybe.

But watching that same trace turned up something I wasn't looking for. Three
of my four game objects (the two scoreboard style ones and my character's
body) each send exactly the two values I told the server to send for them,
and the client applies both, cleanly, every time. My player controller, the
one object at the centre of this whole mystery, sends the same two values
but the client only ever applies **one** of them. Not "applies the wrong
one", not "crashes", just quietly stops after the first and moves on to the
next message like nothing's missing.

That's new, and it's specific to exactly the one object I already suspected.
It also means my "both flags happen to end up on the wrong values by
accident" theory can't be the *whole* story either, since the mechanism I
thought would cause that never runs. Something earlier in the pipeline, the
part that turns a raw number on the wire into "this is property number 18,
the Role field" is where I need to look next: whether the number I'm sending
even survives to that point unchanged for this one particular, unusually
large object.

## Where this actually leaves things

Two theories down today, not one, and the search area is smaller and
stranger each time: it's not the loading screen logic, it's not this
particular truncation check, it's something upstream of both, and it only
shows up on the one object with the longest family tree of the four. Next
job: catch the raw number as it comes off the wire for that specific message,
before anything tries to look up what it means, and see whether it's already
wrong by the time it gets there.
