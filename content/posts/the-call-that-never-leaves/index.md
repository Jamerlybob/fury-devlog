---
title: "20. Somebody's Home"
date: 2026-09-13T12:00:00+12:00
draft: false
tags: ["fury", "reverse-engineering", "networking", "unreal-engine-3", "frida"]
series: ["Reviving Fury"]
summary: "Two separate mysteries about a client that refuses to talk to my server, a wire-format bug hiding inside another wire-format bug, and a character who spent two weeks flat-out refusing to appear on screen. Then I actually looked at the screen, and there was someone standing there, walking around, on my own server."
ShowToc: true
---

Fury's loading screen was finally dropping, but the client still had two
problems: after about twenty seconds it raised "Connection Interrupted", and
it never sent the server the RPC which says loading is finished. The first was
a missing clock update. The second turned out to involve an enum that was one
bit too wide, an authority flag sent from the wrong point of view, and a
player controller silently deciding that its network call belonged locally.

Then the controller started talking and I fell through the map. Anyway.

## The small one: teaching the wire a new number

Every value the game sends over the network gets packed into bits by a
specific bit of game engine code, one function per data type. I'd already
figured out how it does whole numbers and normal decimals (the ones
programmers call "floats"). But the exact number I needed for the clock fix
is a "double", a wider, more precise decimal, and I'd never seen the game
write one of those before. Rule I set myself ages ago: never guess a wire
format, go and read the actual compiled code that does it.

So I did. RTTI gave me the `UDoubleProperty` vtable at `0x114280B4`.
`NetSerializeItem` is slot `+0x134`, which resolves to `0x10D961C0`. Ghidra's
entire useful output for it was this:

```asm
MOV  EAX,[ESP+0x0c]
PUSH ESI
MOV  ESI,[ESP+0x08]
PUSH EAX
MOV  EAX,0x8
CALL 0x10912550        ; FArchive::ByteOrderSerialize(Data, Count)
MOV  EAX,0x1
POP  ESI
RET  0x0c
```

The float and integer versions have the same 29 byte shape and call the same
serializer. Their immediate is `0x4`; the double version's is `0x8`. That's
the whole format: copy eight bytes through the archive. No trickery, no
downcast to a four byte float.

On this little endian machine, `83.828` becomes:

```text
double 83.828
bits  = 0x4054F4FDF3B645A2
wire  = A2 45 B6 F3 FD F4 54 40
```

The server's bit writer only accepts 32 bits per call, so the C# writes the
low word and then the high word. That produces the same eight bytes without
inventing a special case in the bit writer. I round-tripped a pile of test
values through it (including the classic
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

## The bigger one: a message that's been going missing

Quick recap for anyone just joining: the game world now genuinely loads. The
loading screen drops, the arena renders, my character's HUD comes up. But the
instant that happens, the game is _supposed_ to send one particular message
back to the server, basically "hey, I'm done loading, loading screen's gone."
It never arrives. I've known this for a couple of sessions and each time I've
written "not chased yet" and moved on to something more tractable.

Not this time. Here's the thing that made it worth cracking open: there is
exactly **one** function, in the entire multi-megabyte client executable,
that actually puts a reliable message onto the wire. Every single kind of
network traffic (chat, replicated data, remote function calls, all of it)
funnels through this one piece of code before it ever touches a socket. If I
hook that one function, I _cannot_ miss the message, no matter which weird
code path it takes to get there.

So that's what I did. Frida attached directly to `UChannel::SendBunch` at
`0x10B3CAD0`. The useful core of the probe was small:

```javascript
const BASE = ptr('0x10900000');
function va(address) { return BASE.add(address - 0x10900000); }

Interceptor.attach(va(0x10B3CAD0), {
  onEnter(args) {
    const channel = this.context.ecx;
    const chIndex = channel.add(0x2c).readS32();
    send({
      event: 'SendBunch',
      chIndex,
      trace: Thread.backtrace(this.context, Backtracer.ACCURATE)
        .map(DebugSymbol.fromAddress)
    });
  }
});
```

This is a 32 bit MSVC build, so `this` arrives in `ECX`; `channel+0x2c` is the
channel index. The executable is fixed at image base `0x10900000`, which is
why those virtual addresses can be used directly instead of resolving an
ASLR slide first.

Then I ran a real session: server up, client spawned, held it open for ninety
seconds after the loading screen dropped.

The trace had a very distinctive shape:

```text
startup       SendBunch ch=0  calls 1..26
load complete OnLoadingCompleteCheck
load complete DisableLoadingScreen> 1
next 83 sec   no SendBunch calls
shutdown      SendBunch ch=0  call 27, UChannel::Close
```

So: 26 calls right at the start (the normal handshake chatter), then
**nothing**. Not one more call to that function for the entire ninety
seconds, right up until I forced the client to close and it sent one final
"I'm disconnecting" message on its way out. The client's own log file
confirms it genuinely did finish loading a few seconds in
("OnLoadingCompleteCheck", "DisableLoadingScreen> 1"), and then it just...
sat there. Fully rendered, fully idle, saying nothing.

## What that actually rules out

Going in, my leading theory was that the "I'm done loading" call was sneaking
out through some other, harder-to-see path, maybe through a virtual function
table lookup that my earlier static analysis (reading the disassembly cold,
without running anything) couldn't trace back to its caller. That would've
been a real headache: it'd mean the call _was_ leaving, just via a route I
hadn't found yet.

This experiment kills that theory outright. A virtual call loads a function
pointer from the object's vtable and does an indirect `CALL` through it.
Ghidra may not have a static cross reference because the target is only known
at runtime. Frida patches the target function's first instructions, so both
`CALL 0x10B3CAD0` and `CALL [EAX+slot]` land on the same hook. If the message
left the process, I would have
seen it. It didn't. Which means the call isn't going missing on the way out:
**something is stopping it from being sent in the first place**, before it
ever gets anywhere near the networking code.

That's actually good news, in the annoying-progress kind of way. It narrows
the search from "somewhere in a massive networking stack" down to "somewhere
in how this one game object decides whether it's allowed to talk to the
server at all". My money's on something in how my shortcut version of the
server sets up that object not quite matching what a real server would do,
so the client's own bookkeeping doesn't think it has anywhere to send the
message to.

## Catching it red-handed

Plan: catch the object at the exact moment it decides not to send anything,
and read its own fields to see what it thinks is wrong.

First problem, I didn't actually know which of the game's thousands of live
objects to read, or where in memory to find the one field I cared about
("who's my controller"). Games don't ship with a helpful map saying "this
byte means that". I had to build a small tool that walks the game's own
internal bookkeeping, the same lookup table the game itself uses to figure
out which byte means what, and ask it "where does the property called
Controller live on this particular object".

I built it, ran it, and it told me it couldn't find the field at all. Not
"the field is empty", genuinely "I searched and there's nothing here called
Controller". That's the kind of result that should make you suspicious of
your own tool before you get excited about what it's telling you, so I added
a sanity check: ask the same tool to find a _different_ field I already knew
the answer to from earlier work. It failed that one too, in a suspicious way
(the same object kept reporting the exact same number of fields at six
different, genuinely different points in its family tree, which isn't how
real data behaves). So the tool was lying to me. Good thing I checked.

Rather than debug that approach further, I switched to a technique from two
sessions ago that I already knew worked: instead of walking the game's
static blueprint of a class, watch the _live_ lookup the game itself
performs while it's actually receiving data over the network, and borrow the
answer it comes up with. Same sanity check against the field I already knew,
and this time it came back exactly right.

With a trustworthy way to find the field, I read it, twice, on two separate
runs of the game. Both times: the field isn't empty. My leading theory was
dead on arrival, the "who's my controller" question, which I'd worried might
be pointing at nothing and crashing the function silently, genuinely has an
answer.

But then I read the two fields sitting right next to it, and this is where
it gets strange. Every actor in this game engine carries two flags that
answer "who's actually in charge of me, the server or the client I'm running
on". For the game's own controller, on the client's own screen, both of
those flags say "the server". Both of them. On the client. About its own
local copy of the thing it's supposedly a client's-eye view of.

That shouldn't happen, or at least, it doesn't match either of the two ways
I'd have expected it to go wrong. And there's a good reason to think it
matters: if this game object genuinely believes it already has full server
authority over itself, it would have no reason to ask permission before
doing something, which is exactly what "send a message asking the server to
acknowledge I'm done loading" is. It would just quietly do the thing locally
and never bother the network at all. That would explain the silence
perfectly.

## Chasing the flags, and finding something else entirely

I had a specific, concrete idea of _how_ those two flags might end up both
saying "the server": my earlier reading said the client applies a batch of
properties one at a time, in a fixed order, and if it stops partway through
that batch for the controller object specifically, it would land on exactly
the wrong pair of values by accident. There's a single spot in the client's
code that would cause exactly that kind of partial stop, and this time I
could actually watch it happen live instead of reading cold disassembly and
guessing.

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
Something earlier in the pipeline, the part that turns a raw number on the
wire into "this is property number 18, the Role field" is where I need to
look next: whether the number I'm sending even survives to that point
unchanged for this one particular, unusually large object.

## Hand-decoding the whole message, bit by bit

So I did exactly that, the slow way: took the exact raw bytes for my player
controller's very first message and decoded every single bit of it by hand,
against my own written spec for how each piece is supposed to be packed.
Tedious, but it can't lie to me the way a half-trusted tool can.

The live bit reader told me exactly where it was:

```text
FBitReader @ ESI
  Data     [ESI+0x78]
  NumBits  [ESI+0x84] = 67
  Pos      [ESI+0x88] = 54
mask table 0x115A1C74 = 01 02 04 08 10 20 40 80
```

That mask table shows that the stream is LSB first. Bit zero is
`byte & 0x01`, bit one is `byte & 0x02`, and so on. It looks backwards if you
write the bytes as ordinary hex, so I wrote the fields out in consumption
order instead:

```text
bits 00..31  archetype object reference       NetIndex 32405
bits 32..42  compressed location              (0, 0, 0)
bits 43..51  property handle                  18 = Role
bits 52..54  server's Role payload            1 1 0
bits 54..62  client's next handle, misaligned 0 1 1 0 0 1 0 0 0 = 38
```

The first forty three bits matched my spec exactly: an identifier for which
object this message is about, then a compressed 3D position. Good, that part
of my understanding is solid. Then came the two values I'd been chasing:
first the "who's in charge" field again (nine bits saying which property
this is, then some number of bits for the actual answer), then, immediately
after it, the second field.

Here's the catch. That "some number of bits" isn't fixed. It depends on how
many possible answers the field has. For this particular field there are
four real answers ("nobody", "the server", "a remote copy", "the local
player"), so I'd assumed three bits worth of room, since three bits can
count up to eight and the game's own compiler, I knew, quietly adds one
extra, unused placeholder answer to every list like this, making five
entries total, and I was rounding up from five.

My server was sending three bits. When I read what the client actually
consumed, bit by bit, it only ever took two. One bit short. And that missing
bit doesn't just vanish, it becomes the first bit of the _next_ thing the
client reads, the identifier for the second field. Every single bit after
that point is shifted one place to the left. I checked the exact number that
produces: the second field's identifier is supposed to be 19, and shift it
left by one bit and you get 38. That's the number I'd been seeing in every
failed decode for two sessions. Not a coincidence, not a rounding error, the
exact fingerprint a one-bit-too-narrow field leaves on everything that comes
after it.

## Finding out why the field is one bit too narrow

Knowing the field was one bit short is not the same as knowing why. My own
notes, and the game's own diagnostic tool that I'd built earlier, both
agreed: five possible answers (four real ones plus the compiler's silent
placeholder) should need three bits. So either my count of five was wrong,
or the rule "count every possible answer, including the placeholder" was
wrong.

First I checked the count itself, no assumptions. I wrote a small tool that
walks every one of the eleven compiled script files the game loads and
lists every single place any of them defines something with this field's
exact name, in case two different files define two different, unrelated
things that happen to share a name and the game was quietly picking the
wrong one. There is exactly one. Five entries, in the one file I expected.
That theory's dead.

So the rule itself had to be wrong, and the only way to know for sure was to
go back to the actual compiled game code and read, instruction by
instruction, what it does. I pointed my disassembler at the exact function
that decides how many bits a field like this gets, and there it was: right
before it works out the bit count, it takes the count of five and subtracts
one, every time, no exceptions. Then it does the "how many bits to fit this
many values" math on _that_ number, four, not five.

```asm
MOV  EAX,[ECX+0x80]    ; UByteProperty::Enum
TEST EAX,EAX
JZ   plain_byte
MOV  EAX,[EAX+0x48]    ; Enum->Names.Num() = 5
SUB  EAX,0x1           ; drop compiler-generated _MAX
CALL appCeilLogTwo     ; ceil(log2(4)) = 2
```

Four values need two bits. That's it. That's the whole bug. The placeholder
answer the compiler adds gets counted for bookkeeping purposes, but the game
never actually sends it as a real answer over the network, so it never
budgets wire space for it at all. My spec had the right idea (count the
placeholder) but the wrong conclusion (give it room on the wire too).

I fixed the one line of code that had it wrong, reran my full test suite (it
now checks the corrected rule instead of the old broken one), and
recomputed that same player controller message by hand: 43 bits for the
position, 11 bits for each of the two fields I'd been chasing, forty three
plus eleven plus eleven, sixty five bits total. Which is exactly the number
my earlier hand-decode said the whole message needed to add up cleanly.
Three completely separate checks (the shipped game files, a live recording
of the real client reading real bits, and the compiled code itself) all
agree on the same two-bit answer. I'm about as sure of this one as reverse
engineering ever lets you be.

## Running it live, and a decoy

Ran it live. The two-bit fix held up exactly as the paper math said it
would: every message decoded clean, on all four of my game objects, with
zero of the "stopped partway through" breaks I'd been chasing for a week.
Nice. And as a bonus, that same fix quietly cured the "who's in charge"
field too, the one that was reading "the server" on both sides of itself. It
was never a separate bug. It was one property landing on the truncated side
of the exact same two-bit cut, every time.

Which meant I got to cross a theory off the list, except it was the wrong
one to cross off. My leading suspect for "why won't the client tell me it's
done loading" had been: it thinks it already has full authority over
itself, so why would it ask permission for anything. Fixed the field.
Watched it read the correct values this time. The client still never sent
the message. Dead theory, in the most annoying way a theory can die: it was
real, it was wrong on the wire, fixing it was worth doing on its own merits,
and it had nothing to do with the thing I actually wanted fixed.

## The one function that can't lie to me

Back to the drawing board, except this time with a much better tool than
last week. There is exactly one function, out of the entire client
executable, that decides whether a script call to another object runs
straight away on your own machine or gets shipped off to the server
instead. Every "hey, do this thing" message in the entire game funnels
through it. Hook that one, and it doesn't matter how weird the path getting
there is, you catch it.

I'd actually found this function two weeks ago and misread what it did,
mistook a completely unrelated bit of bookkeeping code sitting in the same
neighbourhood for it. This time I went back and found the real one by a
trick I like more each time I use it: instead of guessing which function
does what from a string it happens to print, I diff two objects' entire
lists of internal functions against each other and see what's actually
_different_ between them. My message-sending object has ninety extra
functions the generic base object doesn't. One of those ninety, and only
one, contains this exact sentence, baked right into the compiled code as an
error message nobody's ever meant to see: "received script function call
for object not correctly attached to world, ignoring". That's not a guess.
That's the function introducing itself.

Read what it actually does, instruction by instruction, and near the top
there's a check on one single byte:

```c
// AActor::ProcessRemoteFunction, 0x10B48050
if (*(uint8_t *)(Actor + 0x56) == ROLE_Authority) {
    if (Actor->GetViewportClient() == NULL || demo_helper() == NULL)
        return 0;                       // no bunch is ever built
}

connection = GWorld->NetDriver->ServerConnection; // NetDriver + 0x50
return CallRemoteFunction(connection, Actor, Function, Params);
```

`Actor+0x56` is `Role`. If the object thinks it's "the server", the
function takes a special detour meant for split-screen and demo-recording
edge cases, checks two extra conditions that are never true in a normal
single-player-looking-at-a-server session, and quietly returns having done
nothing at all. Any other belief about its own authority, and it skips that
detour entirely and just sends the message.

Which is exactly the flag I fixed two sections ago. Except I fixed the
_bug_ in how it was decoded, not the _value_ my own server was choosing to
send in the first place. My server was telling my own player's controller
"you have full server authority over yourself", because that's genuinely
what the server object looks like from the server's own point of view, and
I'd assumed the client's own startup logic would sort out the perspective
flip on its own. It doesn't. That flip only happens once, before any of the
actual values arrive over the wire, and then those values just overwrite
whatever it produced. Whatever I send is what the client ends up believing,
full stop. A real server sends its own player's own controller a swapped
pair of these two flags, specifically because of that overwrite, and I was
sending the unswapped one.

The distinction here matters. On the server, the controller really is
`Role=Authority` and `RemoteRole=AutonomousProxy`; those values decide which
replication conditions run. On the owning client, the pair sent over the wire
must be swapped so the local copy sees itself as the autonomous proxy. I had
used one pair for both jobs.

Two lines of code, swap which value goes out for which flag, just for the
owning controller specifically. Rebuilt, ran it live: that exact function now
fires for the exact message I've been chasing since post 19, and it returns
"handled" instead of doing nothing. Traffic that used to be a flat line
after the first second turned into a steady stream, thousands of messages
over a forty second hold, roughly one every few milliseconds, which is
suspiciously close to "once a frame". I haven't decoded what's actually in
them yet. But something my client believed about itself was wrong for two
weeks, I found the one line that mattered, and the silence is over.

## The flood has a name

Wrote a little script that reads the raw bytes of a captured session and
looks up each message's number against a list of every function the game's
own compiled script knows about, so it can tell me "number 50 means this
one" instead of me making that up. Important bit: it pulls that list fresh
every time instead of using one I wrote down by hand weeks ago, because I
already had one of those lying around from an earlier session and it was
wrong, quietly, in a way that would have relabelled everything if I'd
trusted it. The numbers move every time I fix something upstream. Lesson
noted.

Pointed the fixed version at the exact capture from the fix above: 2288
messages, and 2279 of them are the same thing, a function called
DualServerMove. That's the client saying "here's where I'm trying to go,"
sent twice over for reliability, because it's never once heard back from
anything it's sent, which tracks, because my server doesn't listen to any of
this yet. It just lets the shouting land in a bin.

```text
actor channel 3, 2288 inbound bunches
  handle 50  DualServerMove  2279  99.6%
  everything else               9   0.4%
```

Three separate bugs in that one thread, and only the last one was the thing
I originally went looking for. Wire-level proof the message goes out, the
client's own network layer treats it as sent, and a name for what it's
saying every frame after that. What I still hadn't done was watch my own
character actually move on screen, which is the whole point of any of this.

## The next live check, and I fell through the world

So I did the live check. Sat there and watched my own client connect to my
own server, and the camera just kept falling. No ground, no floor, nothing
below me, just falling in silence for a couple of minutes while the map got
smaller and smaller behind me.

![The Fury client's camera, mid fall, with the EL1_Mortem map already tiny and far below.](fell-through-the-map.gif)

Turned out to be my own fault, and a dumb one. Weeks ago, when I first got a
character-shaped thing onto the server's books at all, I had to tell the
client _where_ to put it, and I didn't have a real answer for that yet, so I
typed in zero, zero, zero and told myself I'd come back to it. I did not
come back to it. Zero, zero, zero is just a point floating in empty space
above (or below, unclear) the actual level, so my poor pawn had been
faithfully falling through the void this entire time and nothing was ever
going to catch it.

The fix, once I actually looked, was almost insultingly easy: the map file
itself is just another one of these Unreal package files I've been reading
all month, and it has actual spawn points baked into it with actual
coordinates. Pointed the same decompiler-adjacent tool I use for the script
files at the map instead, asked it to list every `GOPlayerStart`, and got
four of them back, tagged "Deathschool," sitting at a real height with real
ground presumably underneath. Copied one in. Rebuilt. Ran it again.

```text
GOPlayerStart "Deathschool"
Location = (X=-6470.0273438, Y=3197.3193359, Z=4311.3168945)
float32  = 38 30 CA C5  1C D5 47 45  89 BA 86 45
```

Those last twelve bytes are the three IEEE 754 floats as they sit in memory.
They became useful later: instead of guessing the pawn's `Location` offset, a
Frida probe scanned the live actor for that exact triplet and got an address
grounded in a value I had actually sent.

![Standing (not falling) on the Deathschool platform, full combat HUD up: health bar, ability hotbar, minimap, the lot.](standing-on-deathschool.png)

No more falling. Actual HUD came up, too, which I wasn't expecting to just
work: health bar, ability bar, minimap, all of it, rendering fine against a
real position in the world for the first time.

Except there's no character in that screenshot. Zoomed all the way out,
looking straight down at where I should be standing, and it's just...
empty platform. Nobody home.

So we poked at it a bit more. Moved the mouse, camera turned, normal. Hit
WASD, camera moved, so something is definitely being driven around by my
inputs. Which is, technically, the entire point of this whole month of
work, an actual answer to "does the character move," except there's no
character to look at while it happens, which takes a lot of the
satisfaction out of it.

![Same platform, camera pulled all the way back, and still nobody there.](no-character.gif)

And the movement itself is weird. Not walking-weird, more like being lobbed.
My mate watching it happen called it "almost parabolic," which is exactly
right and also a very funny way to describe your own player character.

## A robot, two failed theories, and a good question from James

Two mysteries now: where did my body go, and why do I move like I'm made of
physics homework. Testing that second one properly means someone standing
there watching a number for a while, and I'm not always around to be that
someone, so I wrote something that stands in for me: it spawns the client
itself, points a memory scanning tool at whatever object the server just
spawned, and reads its numbers back without a human touching a key.

First run, it confidently reported that every single thing on the level, my
player, the HUD, the camera, all of it, was named "the world itself." Not a
great start. Turned out I'd told it to read the wrong one of a function's
several arguments. Fixed that, pointed it at my actual character, and
watched its height off the ground for twelve straight seconds with nobody
touching the controls. Didn't move once. One nearby byte flipped from five
to four about seven seconds in, no idea yet if that's the physics state or
just an animation counter.

Tried the direct approach next: stop guessing what physics state the
character is in and just tell the server to say, out loud, "this one's
walking," the same trick I already use for who owns the character. Rebuilt,
watched it for ninety seconds. Didn't fix it. Still no character on screen,
still moving weird, and now I could float straight up and down just by
pressing keys, which is the exact opposite of what "walking" is supposed to
do. Reran the robot to compare before and after: the value I told the server
to send never showed up anywhere in memory. The fix hadn't even arrived.

Before I went and built a proper hook to catch that in the act, James asked
the question that had been sitting underneath the whole theory without
either of us saying it out loud: what if there's no character being driven
here at all, and the "floaty camera" is just a camera doing what a camera
does when nothing is attached to it. In this engine, a controller with no
character to drive falls back to a free-floating spectator view that takes
raw keyboard input as plain movement in space, no ground, no collision, up
and down for free. Which is exactly the new symptom.

Cheap to check for real: "who's driving whom" is always three pointers, the
controller pointing at the character, the character pointing back at the
controller, and the camera pointing at whatever it's currently watching.
Read all three straight out of the client's memory, twice, from a fresh
spawn each time. Same answer both times: everything points at everything
correctly. Nobody's home wasn't it. Somebody's very much home, the wiring is
fine, and the floaty feeling and invisible body are real bugs in how that
somebody moves and renders. One theory down clean, in about twenty minutes,
for free, using addresses I already had lying around from three other
things I'd checked that session. Back to the physics hook.

## The physics hook, and a much bigger bug hiding behind it

There's one piece of code, inside the same function that reads every
replicated value off the wire, that actually applies a value to a live
object: it looks up which property this next chunk of bits is for, then
hands the raw bits and a memory address to one more function that writes
the value in. Hook that one spot and you see, for every single property on
every single object, the exact address about to be written and what's
sitting there before and after. No more candidate bytes. Just the real
ones, caught in the act.

The call site is inside `UActorChannel::ReceivedBunch`:

```asm
10B3E4E6  MOV  EAX,[EDI]          ; UProperty vtable
10B3E4E8  MOV  EAX,[EAX+0x134]    ; NetSerializeItem
10B3E4EE  MOV  ECX,EDI            ; this = property
10B3E4F1  CALL EAX                ; write into EBX
10B3E4F3  TEST EAX,EAX
```

Frida could not hook the two byte `CALL EAX`; there was not enough instruction
space for its trampoline relocator. I hooked `0x10B3E4E6` for the before state
and `0x10B3E4F3` for the after state instead. That gave logs like this once the
message was fixed:

```text
GOCombatAvatar  property=Physics  target=Actor+0x54
bytes-before=00  bytes-after=01  value=PHYS_Walking
```

First run, no hands on the keyboard, just watching four objects open their
channels. And immediately something looked wrong that had nothing to do
with physics at all. The two scoreboard objects applied exactly the two
values I told the server to send for them, cleanly, same as always. My
player and its controller, the two objects actually at the centre of every
mystery this month, applied almost nothing sensible. One of them looked
like it was reading a completely different property than anything I ever
sent. The physics value I'd spent two sessions chasing wasn't there at all.
Neither was "who owns this character," the value I'd already fixed weeks
ago and had been trusting ever since.

That's a much bigger problem than "one flag is wrong." That's "this whole
message is being read starting from the wrong bit," which explains every
symptom at once, because everything after the mistake reads as nonsense
too.

Only two of my four objects have this problem, which gave me something to
compare instead of just being confused. The two clean ones never send a
starting direction the character should be facing. The two broken ones do,
because I bothered giving them a real spawn direction a few sessions back
instead of leaving it blank. That extra bit of "which way are you facing"
is optional, exactly twelve bits when it's there, and I checked the actual
compiled game code for the rule that decides whether the client should
expect it. The client doesn't look at anything the server sends to decide
that. It looks at its own already-loaded copy of the object's blueprint and
reads one single flag baked in at compile time. My server never checked
that flag, just decided "the character and its controller are the kind of
thing that should get a starting direction" and sent one anyway, twelve
bits the client was never going to read, and then read everything after
those twelve bits one bit too soon for the rest of the message. Every
mystery from this whole thread, missing physics, missing ownership flags,
an invisible body, was sitting downstream of the exact same twelve bits.

The actor's flag word is at archetype offset `+0x84`; bit 23 is
`bNetInitialRotation`:

```c
bool readRotation = ((*(uint32_t *)(Archetype + 0x84) >> 23) & 1) != 0;
if (readRotation)
    Bunch << CompressedRotator;    // 12 bits for this spawn value
```

There is no presence bit in the packet. Sender and receiver must consult the
same class default and therefore already agree. My server's `isSpatial`
shortcut said yes. The real player and controller archetypes said no. The
client left those twelve bits unread and treated their first bit as the start
of the first property handle. From there the parser was doing exactly what I
asked, which was unfortunately nonsense.

Checked which of this game's objects actually turn that flag on, by name,
in the game's own decompiled defaults, out of curiosity as much as
anything: exactly three, and all three are the kind of thing you'd expect,
physics props and vehicles that need to fall over convincingly the instant
they spawn. Nothing shaped like a person or a controller is in that list.
My own two objects had no business getting that flag at all.

Deleted the guess, told the server to stop sending that direction,
rebuilt, and ran the exact same hook again. My player's controller now
applies exactly its two real values, cleanly, no garbage in between. My
player applies all three of its real values, including physics, and the
byte sitting at that address goes from whatever the game spawns you with
by default straight to the exact number that means "walking." Caught the
write itself doing it, not inferred from a candidate scan. Two sessions of
chasing one wrong byte, and the byte was never broken. The message it lived
inside of was being read from the wrong starting point the entire time.

Traded something away to get there, though: my character no longer gets a
starting facing direction sent this way, since that's the very thing I
turned off. Small, separate fix owed later.

## A shadow, at least

The next check caught an assumption I should have questioned sooner. My
player had a mesh component, so I had treated that as evidence that there
ought to be a body inside it. There wasn't. Looking directly at the running
client showed an empty mesh, and all nine body parts still marked as data
that had never arrived.

The game builds a person from those parts. My server had been telling it
that loading was finished without sending them. I added a plain test
appearance using the face, hair and default clothing already shipped with
the client. Checked the order of the fields against the running game
first, which felt particularly necessary after the rotation mistake.

The data now arrives correctly, and the empty mesh becomes a real mesh. On
screen, though, there is still no visible character. There is a shadow.
That is progress, and a much narrower problem, but it is not a character
standing in the arena yet.

## The fix that wasn't

The next memory check looked almost embarrassingly literal. The camera had
a distance of zero, which puts it inside the character. Fury responds by
switching to first person and hiding your own body from you. It still casts
a shadow. That was such a neat match for what James saw that I added the
normal spawn message which tells the player to face the right way. One run
then showed a camera distance of six and the hiding flag switched off.

James tried it. Movement felt normal. Still no person.

I came back later and ran the same check from a fresh client. Distance
zero. First person on. Body hidden. Five checks over thirty seconds all
said the same thing. The packet sent by my server was byte for byte the
same as the apparently successful run, so the packet hadn't fixed the
camera at all. Something else in that earlier run had moved it and I had
given the wrong thing credit. Lovely.

The decompiled game code explains why. That spawn message resets the
camera angles, but once the character already exists it deliberately keeps
the current zoom. Starting at zero means staying at zero. I can force the
zoom to six in a diagnostic and watch first person turn off immediately,
but that only removes one reason not to draw. It doesn't produce the
missing body.

## Following the body factory

Fury doesn't load one finished character model. It loads a face, hair,
shirt, arms, hands, legs and feet, then a native bit of the engine stitches
those seven pieces into one new model while the game is running. The other
two available slots, shoulders and helmet, are empty for this plain test
outfit.

I found that whole factory in the original executable and put a hook on
every stage. The request reaches it. All seven named parts load. The
pre-build step succeeds. The worker thread builds four levels of detail.
The post-build step succeeds and hands the result back to the character. No
missing chest warning, no broken mesh warning, no failed texture warning.

```text
0x10E123E0  CharacterManager::RequestMesh
0x10F635B0  CharacterManagerLoader::StartAsyncLoading
0x10E1C9D0  CharacterManagerWorkerThread::PreBuild       ret=1
0x10E1C360  CharacterManagerWorkerThread::Run
0x10E1D070  CharacterManagerWorkerThread::PostBuild      ret=1
0x10E12B00  CharacterManager::InitializeMeshComponent

LOD0 vertices=2775  UV finite=2775  outside [0,1]=0
LOD1 vertices=1717  UV finite=1717  outside [0,1]=0
LOD2 vertices=754   UV finite=754   outside [0,1]=0
bones=56
```

Then I kept going because apparently I no longer know when to leave a
perfectly healthy corpse alone. The finished model has 56 bones and
thousands of valid vertices. Its animation transforms are finite numbers.
Its scene object exists. Its materials are compiled for skeletal meshes.
The generated colour and normal textures are on the graphics card, and
every sampled character vertex lands on an opaque part of the colour
texture. The light environment is attached too. Even the engine's last
rendered timestamp advances.

Which leaves a very specific and slightly rude result: the body factory
works. The render setup looks healthy. Fury still shows a shadow and no
body.

## The corpse had one more complaint

I found the exact bit of the renderer responsible for deciding whether this
one character gets drawn. Not every character, not every model, this
particular stitched body sitting on this particular platform. The old
executable still carries enough scraps of its original class names to
identify the right scene object, and the running game gave me the matching
address.

The renderer asks two questions before it bothers drawing a player. First,
is this body meant to be visible to this camera? In first person the answer
is no, on purpose. I moved the diagnostic camera back before the body was
created and that answer became yes.

Then the second question quietly killed it anyway.

The player was still in what the game calls its reference pose. That's the
raw arms-out pose a character starts in before an animation system takes
over. Fury has a native flag for it, and the renderer is wonderfully blunt:
if this is a real game rather than the editor, don't draw that player at
all. It increments a little counter beside the flag, returns zero, and
carries on rendering the rest of the arena. Hence the excellent shadow cast
by a person the game had decided not to show me.

The live object layout and branch ended up being this compact:

```text
GOCombatAvatar +0x234  bIsInRefPose       bit 0 = 1
GOCombatAvatar +0x238  RefPoseDrawCounter
scene proxy    +0x104  owning avatar pointer
global       0x116092A8 editor mode       = 0
```

```c
if (!editorMode && avatar->bIsInRefPose) {
    avatar->RefPoseDrawCounter++;
    return 0;                       // no dynamic skeletal draw relevance
}
```

I cleared that flag once inside the diagnostic hook, just to prove the
branch. The renderer immediately changed its answer and entered the
skeletal draw code with relevance bits `0x12`. Useful proof, terrible fix.
Changing memory in a hook is a very elaborate way to lie to yourself if you
leave it there.

The actual cause was back in my pretend loadout. A new Fury player starts
with its weapon type set to `UNSET`. After the body factory finishes, the
client uses the weapon type to choose its animation sets and animation
tree, the machinery that decides which pose comes next. For `UNSET`, the
original code explicitly does nothing. My server supplied a face, hair,
clothes and skin colour, but never supplied that last choice. The finished
person stayed in the raw starting pose forever, so the renderer kept
refusing to draw it forever. Very principled.

I changed the test loadout to Fury's `UAR` animation family. The shipped
socket table gives that family no weapon models, which suits this plain
test character, and it gives the client a real animation tree to
initialise. Fresh run, no poke at the pose flag: reference pose false.
Camera visibility yes. Renderer answer nonzero. The exact skeletal draw
function starts firing continuously.

That was the first clean run where the server's real data reaches the
original client and the whole body pipeline ends in draw calls, without
patching the client or propping the result up inside a memory hook. What
was still owed was the only test that matters to a normal person: look at
the actual screen, zoom out, walk around, and confirm there is finally a
moving human attached to the shadow.

## The last stupid thing before the payoff

Naturally, my first attempt at that final check did not work, for a reason
that had nothing to do with any of the above. I started the server, gave
James a command to run the client, and it sat on the loading screen doing
nothing. Handshake looked perfect in the logs, actor channels opened
cleanly, and then silence, same shape as three separate real bugs earlier
in this post, which is a fantastic way to get a small adrenaline spike for
no reason.

Turned out to be embarrassingly simple: a much earlier playtest (the one
where James first confirmed the shadow and the camera fix, a few sections
back) had needed one extra flag on the server's command line to get the
client past a package-negotiation step, and I'd just forgotten to type it
this time. The working combination was `--actors all --uses --basepkg omit`.
Without `--uses`, `JOIN` still completes and all four actor channels open, but
the client waits forever for the `USES` and `HAVE` package exchange. Added the
flag back, no code changes anywhere, ran it again.

## Somebody's home

It worked.

The client connected, sailed straight through the loading screen, and sat
there sending a steady stream of "here's where I'm trying to go" messages,
the same DualServerMove traffic from a few sections ago, except this time
there was an actual body attached to it. James zoomed the camera out from
its default first-person distance, and:

![Standing in the middle of the Mortem arena, full name tag reading "Unknown Entity" hovering overhead, HUD and minimap up, and for the first time an actual visible person underneath all of it.](we-have-a-character.gif)

A person. Standing on a platform. In an arena, on a server, that did not
exist an hour before this project started. And then, because standing
still is only half the point:

![The same character mid-stride, walking across the platform under real WASD input, on a server I wrote from nothing but a decompiled game and a lot of stubbornness.](actual-movement.gif)

Walking. Actual walking, not the parabolic freefall hop from a few sections
ago, not an empty mesh casting a shadow for nobody, an actual visible human
being moved around by actual keyboard input on a server that has no idea
what a database is yet and doesn't care.

The server still throws those movement bunches away. There is no simulation,
correction or database behind this person yet. But the unmodified client has
joined a local match, instantiated the right pawn, built and animated its
mesh, accepted WASD input and emitted real `DualServerMove` traffic while the
body moved on screen. That is the milestone 4 test, finally returning true.
