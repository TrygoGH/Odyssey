# Odyssey 2.0

**Status:** Work in progress. Not yet usable for creating games.  
This directory tracks development progress for Odyssey 2.0, a TurboWarp-optimized version of the Odyssey engine.

---

## Purpose

Odyssey 2.0 is being developed to take advantage of TurboWarp-specific features and compiler optimizations.  
It serves as a testbed for new subsystems, performance experiments, and architectural improvements before porting to the vanilla Scratch 1.0 version.

---

## Current Subsystems

- Runtime & memory management (heap, stack, handle-based reference counting)  
- Deterministic initialization (ensures predictable startup order)  
- Type system with structs and objects (partial implementation)  
- Renderer and camera pipeline (world -> camera -> screen)  
- Input, time, and sound systems (tick-based updates)  
- Save/load system (basic serialization)

**Note:** Many features are partially implemented or experimental. Expect frequent changes.

---

## Exploring the Engine

- Open the .sb3 project in [TurboWarp](https://turbowarp.org/).  
- Inspect the engine structure, runtime behavior, and sprite subsystems.  
- Do **not** expect to create playable games yet. This version is for experimentation only.

---

## Roadmap

- Complete runtime and memory management.  
- Expand type system and method/parameter handling.  
- Implement full scene and GameObject lifecycle.  
- Improve rendering and camera systems.  
- Prepare for eventual vanilla Scratch 1.0 port.

---

## Documentation

See the root README for overall project context and links to planned architecture docs.  
Future detailed architecture and subsystem documentation will be added in the `/docs` folder.

---
