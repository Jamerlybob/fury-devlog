---
title: "Why I'm Reviving a Dead MMO"
date: 2026-08-31
draft: false
tags: ["fury", "intro", "unreal-engine-3"]
series: ["Reviving Fury"]
summary: "Kicking off a project to reverse-engineer and revive Fury, a 2007 MMO that's been dead since 2008."
ShowToc: true
---

## What was Fury?

Fury was a PvP-focused MMO built by Auran on Unreal Engine 3, published by Gamecock Media Group in October 2007. It shut down less than a year later, in mid-to-late 2008, after Auran laid off its staff. It's obscure enough now that there's essentially no existing preservation or private-server community around it — unlike bigger dead MMOs, nobody's done this before, as far as I can find.

## Why this project

I still have a full client install from years ago. That's it — no server, no source code, just the client and everything that shipped inside it. I wanted to see how far reverse-engineering could actually get me toward a playable revival.

I'm also treating this as a structured way to build real software engineering skills — networking, reading unfamiliar code, system design — while I weigh a career move into development. Whatever happens with the game itself, the process is the point.

## What's actually in the client

Turns out, quite a lot:

- `.upk` files — Unreal package files with models, textures, sounds
- A `Script Final Release` folder full of `.u` files — compiled UnrealScript bytecode, including what look like genuinely custom gameplay packages (`GOGame.u`, `GOCharacter.u`, `GOWeapons.u`, `GOSkills.u`)

UnrealScript compiles to bytecode rather than native machine code, which means tools like UE Explorer can decompile it back into something close to readable source — classes, functions, state machines largely intact. That's a much better starting point than reverse-engineering compiled C++ would be. See the [glossary](/glossary/) for more on how this works.

## What I don't have

No server binary, no network protocol documentation, no live server to capture traffic against, no account/auth backend, no database schema. All of that has to be built from scratch, using the decompiled client logic as a reference rather than reused code.

## The plan, roughly

1. Decompile and catalog the gameplay logic in the `.u` files
2. Design and build a new server around that logic
3. Reverse-engineer the network protocol by testing the real client against my own server
4. Build auth/account/matchmaking
5. Get an actual client to log in and play

Step 3 is the real unknown — there's no live server left to observe, so a lot of this will be trial and error.

## Next up

First real technical post: what UE Explorer actually shows when you point it at `GOSkills.u`, and whether the class/skill structure is as legible as I'm hoping.
