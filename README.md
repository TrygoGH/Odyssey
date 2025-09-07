# Odyssey Engine

Odyssey is a Scratch-based, higher-level programming environment designed to bridge the gap between Scratch and traditional programming languages like JavaScript. Inspired by the concept of a journey or adventure (hence the name "Odyssey"), it guides users through progressively enhancing their programming skills as they navigate learning and problem-solving.

The engine introduces abstractions and systems that enable management of more complex programming concepts within Scratch's environment, while maintaining a strong focus on performance and ease of use. Odyssey empowers users to implement advanced programming constructs and practices inside Scratch, emphasizing smooth skill progression, reliability, and optimized performance.

Odyssey 2.0 is currently under active development and not yet usable for making games. This repository tracks progress and experiments with TurboWarp-specific features.

---

## Project Versions

This repository includes two versions of Odyssey:

- **Odyssey 2.0** (coming first)  
  Built for TurboWarp. Uses TurboWarp-specific features and compiler optimizations. Easier to develop and runs with higher performance.  

- **Odyssey 1.0** (coming later)  
  A base version that works in vanilla Scratch without TurboWarp. This version aims for maximum compatibility, but will not be as feature-rich or fast as Odyssey 2.0.  

---

## Status

**Odyssey 2.0 is not ready for game development.**  
The repository is intended to track development progress, test subsystems, and experiment with TurboWarp-specific features. Users should not expect to create playable games yet.

---

## Features (Odyssey 2.0)

- Runtime and memory management  
  Heap, stack, handle-based reference counting, and safe/unsafe functions.

- Deterministic initialization  
  Solves Scratch's "double green flag" problem by ensuring reproducible startup and consistent object initialization.

- Type system  
  Modular, static types with structs and objects, including inheritance.

- Renderer and camera  
  Full rendering pipeline (world -> camera -> screen) with layered UI support.

- Input, time and sound systems  
  Player input handling, delta time management, and tick-based sound playback.

- Save/load support  
  Serialize game state to lists and restore via generated save strings.

Note: Some of these features are only partially implemented and features may change as development continues.

---

## Getting Started (Odyssey 2.0)

1. Open the Odyssey 2.0 .sb3 project in [TurboWarp](https://turbowarp.org/).  
2. Run the engine with the green flag.  
3. Explore the engine's structure, runtime behavior, and subsystems.


Note: Odyssey 1.0 (vanilla Scratch) will be released later.

---

## Documentation

- Work in progress...

---

## Project Context

Odyssey can be thought of as a VM inside a VM inside a VM:  
- Scratch runs on JavaScript  
- Scratch simulates its own runtime  
- Odyssey simulates memory and higher-level systems on top  

This layering allows Scratch projects to explore advanced computer science concepts like memory management, type systems, and component-based game architecture.

---
