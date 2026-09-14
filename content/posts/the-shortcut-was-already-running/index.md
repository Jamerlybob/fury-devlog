---
title: "The Shortcut Was Already Running"
date: 2026-09-14
draft: false
tags: ["fury", "unreal-engine-3", "networking", "reverse-engineering"]
series: ["Reviving Fury"]
summary: "I finally checked the project's biggest shortcut test and realised the working game had already answered it."
ShowToc: true
---

There was one old question sitting under all the server work: could I get into a
match without rebuilding Fury's original login and matchmaking system?

That system uses a separate, custom network protocol buried in the game's native
code. It is the expensive branch of this whole project. A week ago I wrote down
a hard time limit for reverse engineering it, because otherwise it had all the
makings of a very tidy ten year side quest.

The planned test was to build a game address by hand, point it at my own arena
server, and see whether the client could get all the way in without the launcher
or any of the old account services.

Then I looked at how I had been testing the arena server.

```text
Fury.exe 127.0.0.1:7777
```

That was it. No launcher. No login server. No realm server. The same command had
already put a visible character into Mortem and let me walk around with WASD.

The risky shortcut test had passed while I was busy debugging trousers.

## The smaller stub that remains

There is still a `realmurl` field in the avatar creation screen. The decompiled
script stores it in a native object, and exposes a console command named
`UpdateRealmInfoRequest`. The response parser expects little status tags for
each realm, but the actual web request lives in compiled C++ code, so I am not
inventing that part.

I pointed the avatar screen at a passive listener on localhost and waited. Zero
bytes.

Then I read the timer properly. It waits sixty seconds after the map finishes
loading, not sixty seconds after the program starts. The request finally arrived
about eighty three seconds into the next run:

```text
GET / HTTP/1.1
Accept: */*
User-Agent: Fury
Host: 127.0.0.1:18080
Cache-Control: no-cache
```

No mystery packet format. No account token. Just a web request for `/` from a
user agent called `Fury`.

I added a tiny service which answers with the smallest thing the decompiled
parser accepts:

```text
<realmID=1 status=0>
```

The final check was inside the running client. Before the response, its realm
array was empty. Afterwards it contained one entry: realm 1, status 0, which
means normal.

There was one last fossil in the code. The old realm selection screen cannot
actually appear in this build. Its `show()` function selects realm 1 and returns
before drawing the list. The updater still works underneath it. The screen is
just already skipping the question.

For now the important bit is settled. Direct arena play does not need Fury's
original account protocol at all. One very unpleasant branch of the roadmap has
quietly become optional.
