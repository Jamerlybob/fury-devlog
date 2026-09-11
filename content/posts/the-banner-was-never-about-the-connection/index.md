---
title: "19. The Banner Was Never About the Connection"
date: 2026-09-11T22:15:00+12:00
draft: false
tags: ["fury", "reverse-engineering", "networking", "unreal-engine-3"]
series: ["Reviving Fury"]
summary: "Last post I promised not to guess why the game says Connection Interrupted the moment the loading screen drops. Turns out it has nothing to do with the connection. It's a clock my server never wound."
ShowToc: true
---

Last post ended with the game actually running, lava arena and all, and one line
sitting dead centre of the screen in big blue letters: **Connection Interrupted**.
I said I wasn't going to guess why. So this post is just me going and finding out.

## A clock, not a connection check

Your player controller, the bit of the game that represents you specifically as
opposed to your character's body, keeps a running number: how many seconds of
game time have passed. It ticks up by itself, every single frame, forever,
whether or not anything is actually arriving from the server.

There's a second number sitting right next to it in the code, meant to track the
same thing but only updated when the server actually says so, like a heartbeat.
The HUD compares the two, every frame, and if they've drifted more than twenty
seconds apart it draws the banner. Nothing ever turns it back off once it's lit.
It just keeps checking, forever, and it's still failing the check.

So the banner was never asking "is the network connection okay". It was asking
"has the server told me the time recently", and the honest answer was no, because
my server has never once told it the time.

## Server homework, still undone

The function that's supposed to tell the client the time exists right there in
the script, doing exactly what you'd expect: send the time, then set a timer to
do it again in fifteen seconds. But the bit of code that actually schedules the
first call was sitting inside one of those `if(false)` blocks I keep running
into, the ones marking logic Auran's real server ran and the shipped client
never does. So the client's own copy of this class never even tries. It's a
chore for the server, and I'm the server now, and I've never done it.

Which is a nice, clean explanation, and it cost me nothing but reading the
script. Two variables, one comparison, one missing phone call.

## Except I can't make the phone call yet

To fix it I need to send that "here's the time" message, and its first argument
is an eight byte number, a `double`. Every other type this server has ever put on
the wire, whole numbers, plain floats, yes or no flags, has been checked against
the running game and confirmed to be exactly what I assumed. This one hasn't. I
don't actually know that Fury writes a double as eight raw bytes the boring way.
It might. It's the obvious guess. But "the obvious guess" is precisely the phrase
that's burned me twice already on this project, so it's not going in until I've
watched the real client do it, or found the code that does.

So the banner stays lit for one more post. Small consolation: it's a solved
mystery wearing an unsolved costume.

## Two silences, not one

Which also means the cliffhanger from last post is still exactly where I left
it. The message the client is meant to send back the moment the loading screen
drops, its own "I'm in, stop worrying about me", still never reaches my server.
That was never the banner's fault, and finding out why is a proper reverse
engineering job: going into the compiled client itself to find the code that
decides when to send a function call over the network, and working out what it's
checking before it will.

One banner, explained, not yet silenced. One much bigger silence, still
completely unexplained. Next up: pin down what a double looks like on Fury's
wire and shut the banner up for good, then go looking for whatever's holding
that other message back.
