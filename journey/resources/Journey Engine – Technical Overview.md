# Journey Engine – Technical Overview (Updated)

*October 2026 · Companion to the Odyssey engine · Items marked (planned) are not built yet*

Journey is a lightweight, Scratch-based engine that gives projects structured sprite lifecycles, scenes and serialization while keeping normal Scratch blocks and workflows working as expected.

## Project Context

Journey is the stripped-down version of Odyssey. Journey targets vanilla Scratch with decent performance; Odyssey keeps vanilla-Scratch-style code but targets TurboWarp (a future Odyssey 2.0 may be fully TurboWarp-based, using extensions).

**Name and identity:** Journey began as the first step on a journey toward more complex development in Scratch, while Odyssey is meant to go to the stars beyond. The Journal system is Journey's signature feature.

**Core goal:** Journey should never perform worse than base Scratch. Code that runs fine in plain Scratch should run about the same in Journey.

Journey uses:

- A runtime sprite that drives broadcast-based lifecycles
- Real Scratch sprites and clones for rendering and sensing
- Cached (virtual) sprite data for dirty checking
- JAML, a YAML-inspired serialization format

## Design Principles

- Every feature needs a clear purpose and use cases, or works quietly in the background with optional settings.
- Never drop standard Scratch functionality. Normal blocks (change x by, if touching, and so on) work inside lifecycle methods.
- The engine observes instead of simulating: it records what happened at the end of a frame and does not own movement or physics.
- The Sprite is the group and the tag. No hierarchies, no prefabs (use clones of clones with lifecycles disabled on the template), no separate tag or group system.
- Editor tooling and documentation aim to make the Scratch editor necessary only for writing code.
- Prefer private over public. The order of preference is private list, then private variable, then public list, then public variable. Private lists are hidden from other sprites, private variables are read-only from outside, and public lists are preferred over public variables because one list holds more data with less clutter.
- Never use infinite loops in lifecycle or tick code. The only infinite loops are the one that calls the runtime and the sound agent clones.

## Core Subsystems

### Runtime System

- A runtime sprite sends the lifecycle broadcasts every frame.
- Plain broadcasts are used for anything visual. There is no broadcast-and-wait (BAW) in the runtime loop, because profiling showed BAWs cost time even when no code ran. A few BAWs cannot be removed. The SceneManager runs code that spans multiple frames but must finish within one; run without screen refresh does not guarantee that, it only avoids forcing a screen update unless something yields (for example by taking too long). The InputManager also needs one to send several event broadcasts (MousePressed, MouseReleased, KeyReleased and so on) before the lifecycles run. None should ever unintentionally trigger a screen refresh at runtime.
- Every lifecycle method has an internal call and a user-facing call. Including the render hooks, a sprite receives about 9 calls per frame.
- Normal Scratch code works inside lifecycle methods, except timing blocks (wait, wait until, and broadcasts that depend on lifecycle timing). The replacement pattern is to store an end time in a variable and check it each update.
- Regular broadcasts sent at the end of the frame (outside the lifecycle) are used for sounds and anything that does not need precise timing or needs to fire more than once per frame.

### Lifecycle Management

- **Awake** — once, when the object is initialized.
- **Start** — once after Awake, kept as its own step for consistency.
- **PreUpdate** — per-frame preparation, including world-space data.
- **Update** — every frame.
- **FixedUpdate** — a true fixed timestep at 60 ticks per second using accumulated time, capped at 5 steps per frame (five separate broadcasts). Visual smoothing by interpolation is intentionally not used.
- **LateUpdate** — end-of-frame logic.
- **Render hooks** — pre-processing, render and post-processing (for effects such as shadows or last-moment visual changes).
- Inactive GameObjects are skipped before any lifecycle logic runs.
- Clones receive the same broadcasts but call their own clone custom blocks (for example CloneUpdate), so clone logic can differ from the original sprite's.

### Time System

- A Time sprite provides delta time, FPS, a timescale and unscaled time.
- Timescale 0 pauses the game.
- A small timer API turns a wait time into an end time and provides functions for total duration, time elapsed, time remaining, has ended and a ratio. Every function has an unscaled version, and callers always pass in the start and/or end time themselves, so no timer state is stored.

### Sprite & Data System

- Each GameObject stores its data in variables: x, y, layer, direction, size, costume, visual effects (brightness, color, ghost, pixelate, mosaic, fisheye, whirl), saying and thinking text, visibility, active and enabled flags, UI flag, target camera, update mode and render mode.
- Sprites can be controlled with ordinary Scratch blocks.
- Each sprite keeps virtual (cached) copies of properties such as position, layer and effects. Dirty checks compare against them, so unchanged sprites run only if-checks and skip rendering code.
- Update mode and render mode decide whether the sprite or its virtual data is the source of truth. The default is that the sprite sets its own virtual properties, which behaves like normal Scratch.
- A global sprite list holds the virtualized information for every sprite and clone.
- Full virtual objects are not built into the engine (planned: possibly later); projects can build their own when they need objects that do not use sensing.

### Clone Management

- Clones are real Scratch clones, so sensing works normally.
- Each clone is tracked by its instance ID in the global sprite list and by sprite-level clone tracking.
- **ID model:** the UUID identifies the GameObject (shared by every instance of it); the instance UUID identifies one specific runtime instance. References inside the engine are UUIDs stored in variables, because Scratch cannot store object references.
- Clones are serialized with their engine properties. They have no custom Sprite Data block yet, and clone properties are not stored anywhere else.
- Clones can be created at editor time and selected in the editor and by sprites.
- Template clones (the prefab replacement) have lifecycles disabled; only real instances run them.

### Rendering System

- Sprites render themselves. Layering uses Scratch's own go-to-front followed by go-back-N-layers (layer 0 = front). Because sprites are real Scratch layers, the order stays stable until a layer changes.
- UI layers are offset from non-UI sprites.
- Only properties that changed are applied.
- World-space data is cached after each non-rendering lifecycle method. The main remaining cost is calculating and caching world-space and screen-space data (about two thirds of a sprite's final render time).
- **Cameras:** a world camera (also the editor camera, with pan and zoom) and a separate UI camera. Screen shake can affect either.
- **ScreenManager:** holds screen bounds, midpoints, ratio and resolution, caches them, and raises an event when the screen size changes (used for responsive UI).
- **RenderObject (experimental):** a simulated sprite without behavior that currently exists only as a stamped sprite and must be rendered manually (automation is planned). RenderObjects support custom fields such as velocity, are never saved and are purely visual. They live in one dynamically allocated list that only grows (deleting wipes values instead of removing items), use the same accessors as the Sprite class, and are selected by index so name and start index can be cached. A sprite can assume a RenderObject, becoming it until it is released or the lifecycle ends. The API is very early and not fully supported yet.

### Serialization System (JAML)

- A YAML-inspired format stored as one string, so it can be copied and pasted directly.
- Entries are separated by an invisible character (U+3164, Hangul filler) because Scratch strips newline characters.
- Hierarchy is represented by the number of tabs; a scalar value sits one level deeper than its key; list items are marked with a dash.
- Structure: scene id, version, name, then Sprites keyed by UUID, each with Sprite Data (custom variables), Main Object (engine properties) and a Clones list.
- Parallel lists track keys, values, depths and (optionally) parents. Decoding rebuilds the parallel lists, not full objects, and all decoding lives in one sprite.
- Serializes about 99% of data (anchors are the exception). It stores no object references.
- Values are text; Scratch casts types implicitly, so types are not encoded.
- Player saves are handled separately with a dictionary.

### UUID System

- Format: four 4-character hex segments, for example abcd-ef12-3456-789a.
- The first segment comes from days-since-2000 (wrapped with mod); the rest is random, which is unique enough for a Scratch project.
- UUIDs persist for saved objects.

### Scenes & Editor

- Scene save, load and switch are fully working.
- The editor loads and saves scenes exactly the way playmode does. A temporary scene is saved before playmode, and a separate saved version is written on manual save.
- Duplicating a scene can keep its data under a new name.
- A file integrity check runs by broadcast when the runtime first loads and sometimes when the editor is entered.
- Editor features: pan and zoom camera, visible grid (editor aid only), hide/show, active/inactive, clone creation, and documentation on the scene sprites. Functions meant to be clicked in the editor are named in square brackets, like \[LikeThis\], so they look like buttons.
- Multi-select and a basic property inspector are planned. Custom properties are set in Awake or in code and saved manually. Undo and redo are postponed.
- A game state manager (StateManager) handles game states, and a GameManager for game rules is in progress.

### Input System

- Keyboard and mouse are supported, with input states, input order, mapped actions, remapping and helper functions, lists and constants.
- A settings menu for remapping exists (subject to change).
- A default analog joystick works but is not customizable yet; touch input is possible but not fully implemented.

### Audio System

- A simple API that can be added to any sprite to play sounds.
- Pitch, panning and other effects; song queueing; sounds have owners, so songs queued by the same owner do not override each other; queue items have lifetimes.
- Multiple copies of a sound play polyphonically (working around Scratch's one-instance limit) using a pool of clones, so each sound keeps its own settings.
- Stores per-sound information: duration, name, index, copies, queue type and more.
- No built-in 2D or 3D audio; callers can queue sounds with their own computed parameters. Sounds can also be played normally from any sprite, but you cannot query how long a sound has left, sounds from a scene cannot be reliably cleared, and overrides cannot be made.

### Math, Camera & Profiling Tools

- Math utilities and constants are included.
- **Profiler:** each tracked system is a profile hooked into a broadcast. It shows each profile's share of total time (averaging over a time slice is planned). It cannot give exact per-frame durations, because starting a profile has overhead and a function's end is only known when the next profile starts.

### Animation System

- Animations are defined in code; a full animation editor is not planned.
- A clip has a name, a frames-per-second value and any number of keyframes. Each keyframe is a costume name plus a frame count.
- Keyframes are expanded into one continuous list of frames holding costume numbers, so each frame is a single lookup.
- Clip names have a parallel lookup list holding the index of each clip's first frame.
- Starting an animation copies its start index and fps into the animation player list. Each sprite then updates its own animation every frame (the frame is computed from scaled time) until it ends, then clears its current animation.
- Only PlayAnimation exists for now. It overrides whatever is playing, even the same animation.
- Everything uses private variables and lists, so it works the same for every sprite and clone with nothing to free.
- Planned: looping with loop counts, ping-pong, animating other visual properties such as effects and size, a UI version, and smoothing functions.

### Raycast System

- A bounded loop checks coarse steps along the ray, with a minimum distance (hits before it are ignored), a maximum distance and a step size.
- Ray properties can be set before casting. A defaults function sets sensible values (step size 10, max distance 480), and a shorthand takes only a target and a direction, applies the defaults and the given values, and casts immediately.
- Results are exposed as the private variables Hit?, hitX and hitY. All values, including the original position and size to restore afterwards, are also stored in the private raycastInfo list.
- Planned: layer masks (by checking stored gameobject names), color casting, color-to-color checks and more variations.

### Build Mode

- A build mode flag exists, with broadcasts for building and unbuilding the project.
- Its main purpose is to store as little data in lists as possible, keeping list lengths small enough for Scratch.

### Journal

- The Journal is Journey's signature feature: a logging system.
- Sprites contain no Journal logic. They only push data to a queue.
- The queue is processed only while the console is open. When the Journal is deactivated or deleted nothing is pushed, leaving only early-return checks, so the cost should be minimal.
- Everything is themed like a journal: function names are themed around journaling, and the engine gets hand-drawn icons for all its art, with full polish coming later.

* Log levels: normal, warning and error. All are built from the same base log, which appends or replaces information in the queue.
* Entries carry the time and days since 2000, plus optional sender and miscellaneous information.
* An error log stops execution immediately after logging. The program can be resumed safely afterwards, and Journey detects that the project stopped and can do any needed saving. Clone data is the one thing it cannot save after a crash.
* Log calls return early if a value inside the Journal sprite is not true, so a Journal deleted for a full build does nothing.
* MVP scope: open the Journal and clear it. The queue is public and the Journal itself is private, so advanced features can be added later without breaking changes.

## Performance & Optimization

**Measured so far**

- A ScreenManager bug (screen info re-checked every frame) cost half the frame budget; fixing it took 100 clones from about 7 to about 15 fps.
- Broadcast overhead is only about 5–10% of per-frame cost. Most of the rest is lifecycle logic, mainly rendering and world-space and screen-space calculation.
- 200 clones that only receive broadcasts run fine. In one test, 200 clones went from about 140 ms to 90 ms per frame without the final screen-space render step.
- Targets: 100 clones at 30 fps, and 300 minimal clones at 15 fps.

**Techniques in use**

- Dirty-property checks and virtual properties
- World-space data cached after non-rendering lifecycles
- No broadcast-and-wait in the loop (one exception)
- Profiler-driven optimization

## Known Limitations

- Timing blocks (wait, wait until) do not work inside lifecycle methods.
- Clone properties are not stored outside their runtime data.
- No hierarchies, groups, tags or prefabs, by design.
- No 2D or 3D audio; sound time remaining, reliable per-scene clearing and overrides are not possible; joystick customization and touch input are incomplete.
- A few broadcast-and-waits (SceneManager, InputManager) cannot be removed, though none should trigger a screen refresh at runtime.
- Seeded random numbers are hard because sprites cannot share a global script (idea: pregenerated numbers taken in order).
- Large numbers of clones are still performance-limited, and raycasts can skip targets thinner than their step size.

* World space is only stable with the camera at the origin and 100% zoom; fixing the camera and world-space system is the first item of the feature lock.
* RenderObjects are experimental, are rendered manually, and as stamps always draw behind real sprites.

## Planned Features

- Grid and tile system (deferred, not part of the MVP): snap-to-grid for normal games and a proper tile system for tileset games
- RenderObject automation and a fuller API (not soon)
- Faster world-space and screen-space calculation and caching
- Trimming redundant calls in the lifecycle
- Raycast extras: layer masks, color casting and color-to-color checks
- Profiler averaging
- Multi-select and a basic inspector in the editor; more editor quality-of-life features
- Animation extras: looping with loop counts, ping-pong, animating other visual properties and more API functions
- Touch and joystick customization
- Documentation and a starter project, once the engine is less volatile

**Current phase:** the project is in a feature lock. Existing features are being checked and optimized, starting with the camera and world-space system. The Journal stays minimal for the MVP (open and clear). Documentation everywhere and finishing the demo projects (a cookie clicker and a zombie wave survival game) come after.

## Journey Engine – FAQ

**Q: Can I use normal Scratch blocks?** A: Yes, inside lifecycle methods too, except timing blocks like wait and wait until.

**Q: How is Journey different from Odyssey?** A: Journey is the lightweight version for vanilla Scratch. Odyssey keeps vanilla-style code but targets TurboWarp.

**Q: Why is there no broadcast-and-wait in the loop?** A: Profiling showed it costs time even when no code is running, and a redraw request ends the frame's stepping, so waits that include visual code stall a whole frame.

**Q: How is FixedUpdate handled?** A: 60 ticks per second from accumulated time, at most 5 steps per frame. There is no interpolation.

**Q: How do clones work?** A: They are real clones that receive all lifecycle broadcasts and call their own clone methods. Their properties are transient and are tracked by instance ID.

**Q: How do I pause the game?** A: Set the timescale to 0.

**Q: How do I make something wait?** A: Use the timer functions: turn a wait time into an end time, then check has-ended each update. Every function has an unscaled version.

**Q: How does saving work?** A: Scenes serialize to a single JAML string; the editor and playmode use the same path. Player saves use a dictionary.

**Q: Is there a hierarchy or prefab system?** A: No. The Sprite is the group and tag, and clones of clones act as templates.

**Q: How do animations work?** A: Clips are defined in code as keyframes (a costume plus a frame count), expanded into a flat list of frames and played with PlayAnimation. Each sprite updates its own animation from scaled time.

**Q: How do raycasts work?** A: A bounded loop steps along the ray in coarse steps (default 10, up to a default max distance of 480) and reports Hit?, hitX and hitY.

**Q: What is the Journal?** A: Journey's signature logging system. Sprites only push data to a queue that is processed while the console is open, so a deactivated Journal costs almost nothing.

## Engine Status

| Feature | Status |
| --- | --- |
| Sprite data model | Implemented |
| Lifecycle methods | Implemented (optimization ongoing) |
| FixedUpdate (60 Hz, 5-step cap) | Implemented |
| Time system and timer API | Implemented |
| Clone system | Working; performance-limited |
| Rendering and cameras | Implemented; world space only stable at the origin and 100% zoom (fix in progress) |
| Serialization (JAML) | Implemented (about 99%) |
| Scenes | Implemented |
| UUID system | Implemented |
| Input system | Mostly implemented |
| Audio system | Implemented (no 2D or 3D) |
| Animation system | Basic (play only; looping planned) |
| Raycast system | Basic (layer masks and color casts planned) |
| Profiler | Working, basic |
| Build mode | Implemented |
| Editor | In progress |
| GameManager | In progress |
| Journal (logging) | In progress (log levels and queue exist; console minimal for the MVP) |
| Grid and tile system | Deferred (editor grid visual only) |
| RenderObject | Experimental (manual stamping) |
| Full virtualization | Not built in; manual per project |

Journey is usable and has supported a working cookie clicker demo, with a zombie wave survival demo in progress. The MVP targets 2D platformers and logic-heavy games that benefit from saves and scenes. The main remaining areas are performance, the editor, the Journal system, and locking down, documenting and demoing the existing features.
