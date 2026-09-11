---
title: "Glossary"
layout: "single"
ShowToc: false
---

A running reference for terms that come up across posts, so I don't have to re-explain them every time. Updated as the project goes.

### UnrealScript
Epic's proprietary scripting language for Unreal Engine 3. It compiles to bytecode run on an in-engine virtual machine rather than native machine code, which is why decompilers like UE Explorer can reconstruct near-original, readable source from shipped `.u` files.

### .u file
A compiled UnrealScript bytecode package. Contains class definitions, functions, and state machines for one part of the game. Fury's are `GOGame.u` (most of the game logic), plus `GOAI.u`, `GOEffect.u`, `GOGameType.u`, `FuryUI.u`, `GOCLI.u`, `GOAnim.u`, `GOEditor.u`.

### .upk file
An Unreal package file containing content (meshes, textures, sounds), as opposed to `.u` files, which contain script and logic.

### Replication
UE3's built-in networking framework for keeping client and server state in sync. It decides which variables and function calls get sent over the network, and to whom.

### UE Explorer / UModel
Third-party tools for browsing and decompiling Unreal Engine 3 package files. UE Explorer focuses on `.u` script decompilation. UModel focuses on asset extraction (models, textures, sounds) from `.upk` files.
