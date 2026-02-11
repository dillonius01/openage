# openage Architecture Overview

## What is openage?

openage is an open-source reimplementation of the **Genie Engine** — the engine behind Age of Empires 1, Age of Empires 2, and Star Wars: Galactic Battlegrounds. It's not a mod or a patch; it's a ground-up rewrite of the game engine in C++20 and Python 3, designed to be moddable, cross-platform, and architecturally modern.

The project is currently in a **major rewrite phase**. The old prototype had basic gameplay working, but the team scrapped it to build a proper ECS-based architecture. Right now, core infrastructure (rendering, pathfinding, events, time management) is solid, but gameplay simulation is still being assembled.

---

## The Big Picture: Three Threads

The engine runs three independent subsystems, each on its own thread:

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│  Time Loop  │     │   Simulation     │     │  Presenter   │
│             │────▶│                  │     │              │
│  Advances   │     │  Game state,     │     │  Rendering,  │
│  the clock  │     │  entities,       │◀───▶│  input,      │
│             │────▶│  events          │     │  GUI         │
└─────────────┘     └──────────────────┘     └─────────────┘
```

- **Time Loop** — maintains the simulation clock. No fixed tick rate; it just advances time and notifies subscribers.
- **Simulation** — the game world. Entities, components, player commands, combat, movement. Runs an event loop that processes all events up to the current time.
- **Presenter** — everything visual. Rendering pipeline, camera, input handling, Qt GUI. On macOS, this **must** run on the main thread (Cocoa requirement), so the simulation runs in a background thread instead.

These subsystems communicate through **curves** and **render entities** — thread-safe data structures that avoid direct shared state. The simulation never touches the renderer directly; it updates curves, and the renderer reads them independently.

---

## The Event System (No Ticks!)

This is the most unusual architectural choice. Most game engines run a loop like "60 times per second, update everything." openage doesn't do that.

Instead, everything is **event-driven**:

- Events are scheduled at specific simulation times (e.g., "at t=3.5s, this unit arrives at its destination")
- The event loop processes all events up to the current time with `reach_time(t)`
- Events can reschedule themselves or trigger new events
- If nothing is happening, nothing computes

**Why?** It's designed for networked RTS. Instead of syncing entire game states 60 times per second, you only need to sync events. It also enables replay and prediction — you can query "where will this unit be at t=10?" without simulating forward.

**Example flow:**

1. Player clicks to move a unit → creates a `MoveCommand` event
2. Command event fires → pathfinding calculates waypoints → inserts position keyframes into the unit's position curve
3. Unit "moves" by interpolating between keyframes — no per-frame position updates needed
4. When the unit arrives (curve reaches destination), arrival event fires → unit goes idle

---

## The Curve System

Curves are how game state is stored. Instead of a variable like `position = (5, 3)`, you have a **curve** of keyframes:

```
Time:     0      1.0     2.5     4.0
Value:  (0,0)  (2,1)   (5,3)   (5,3)
```

Query the position at any time and get an interpolated result. Three curve types:

- **Discrete** — step function, returns the last value set (health, activity state)
- **Continuous** — linearly interpolates between keyframes (position, progress)
- **Segmented** — like continuous but allows instant jumps (facing angle — a unit can snap from 0° to 180°)

There are also **container curves**: Queue (command queue), UnorderedMap (attributes), Array (terrain tiles). All are time-indexed.

Curves double as event triggers — when a curve value changes, dependent events get re-evaluated. This is how the simulation stays consistent without polling.

---

## Entity-Component System (ECS)

Located in `libopenage/gamestate/`. Every game object (unit, building, tree, projectile) is a `GameEntity` — basically an ID with a bag of components:

**Components** (each stores one aspect of an entity):

| Component | What it stores | Curve type |
|-----------|---------------|------------|
| `Position` | 3D world location + facing angle | Continuous position, Segmented angle |
| `Activity` | Current behavior state machine node | Discrete |
| `CommandQueue` | Player commands waiting to execute | Queue |
| `Ownership` | Which player owns this entity | Discrete |
| `Live` | Health/attribute values | Discrete |
| `Move` | Movement capability | (from nyan data) |
| `Turn` | Can change direction | (from nyan data) |

**Activities** are the behavior state machines. Think of a unit's AI as a flowchart:

```
Idle → Received command? → Move to target → In range? → Attack → Target dead? → Idle
```

Each node is an `Activity` node. The system walks the graph based on conditions and events. This was recently made data-driven (configurable via nyan files rather than hardcoded).

`EntityFactory` creates entities from nyan data definitions. It's what crashes when you run `game` — it tries to create players with nyan database views, but the game simulation path isn't complete yet.

---

## The Rendering Pipeline

Two-level design in `libopenage/renderer/`:

**Level 1 — Graphics API abstraction** (OpenGL today, Vulkan planned):
Provides generic GPU primitives: shaders, textures, geometry, render passes. Game code never calls `glDrawArrays` directly.

**Level 2 — Game-specific render stages**, composited in order:

1. **Skybox** — background
2. **Terrain** — the isometric tile grid (3D mesh with textures)
3. **World** — unit/building sprites (2D animations in 3D space)
4. **HUD** — health bars, selection boxes
5. **GUI** — Qt-based UI overlay
6. **Screen** — final compositing (alpha-blends all layers)

Each stage has **RenderEntities** — thread-safe bridges between game state and rendering. The simulation thread writes to them; the render thread reads from them. No direct game state access from the renderer.

**Camera**: Orthographic projection at the classic AoE2 isometric angle (yaw -135°, pitch -30°). Supports **frustum culling** — objects outside the camera view are skipped entirely, which can eliminate ~95% of draw calls on large maps.

**Targeting**: The renderer draws an invisible "ID texture" where each pixel stores the entity ID of whatever sprite is there. When you click, it reads that pixel to know which unit you clicked on.

---

## The Coordinate System

Multiple coordinate spaces in `libopenage/coord/`:

- **`phys2`/`phys3`** — game world coordinates in "terrain units." Fixed-point integers (16-bit fractional part for sub-tile precision). This is what the simulation uses.
- **`tile`** — integer tile grid (a tile center is at phys `(0.5, 0.5)`)
- **`chunk`** — groups of 16x16 tiles (for terrain rendering optimization)
- **`scene`** — renderer's version of world coords, transformed to Eigen vectors for GPU math
- **`input`/`viewport`** — screen pixel coordinates from mouse/keyboard

The UP axis is divided by sqrt(8) to match AoE2's aspect ratio for the isometric projection.

---

## Pathfinding (Flow Fields)

In `libopenage/pathfinding/`. Uses a two-stage approach:

**Stage 1 — High-level A\* on a portal graph:**
The map is divided into sectors (16x16 tiles). Sectors connect via "portals" (passable edges). A* finds which sectors to traverse.

**Stage 2 — Flow field generation:**
For each sector on the path, compute a flow field — a grid where each cell has a vector pointing toward the cheapest path to the goal. Units follow these vectors.

**Why flow fields?** In AoE, you often send 40 units to the same place. With traditional A\*, that's 40 separate pathfinding queries. With flow fields, you compute the field once and all 40 units reuse it. Recent optimizations brought individual path requests down to 0.3-2ms.

Recent blog updates show they've also added:

- Line-of-sight propagation through portals (units move in straight lines across sector boundaries instead of zigzagging)
- Flow field caching
- Diagonal movement blocking fixes (can't cut through impassable corners)

---

## nyan — The Data Language

openage uses a custom DSL called **nyan** for all game data (units, techs, abilities, effects). Think of it as a typed, inheritance-based configuration language:

```nyan
Knight(Unit):
    hp = 120
    attack = 10
    speed = 1.35
    abilities += {Attack, Move}

EliteKnight(Knight):
    hp += 40
    attack += 4
```

nyan supports **patches** — modifications that can be applied/removed at runtime. This is how techs work: researching "Blast Furnace" applies a patch that adds +2 attack to all infantry. It's also the modding system — mods are just nyan patches.

The **converter** (`openage/convert/`) reads original AoE game files (`.dat`, `.slp` graphics, `.wav` audio) and transforms them into nyan definitions + converted media. This is why the game asks about asset conversion on first run.

---

## The Cython Bridge (C++ <-> Python)

C++ handles performance-critical engine code. Python handles asset conversion, scripting, code generation, and test harness. They talk through Cython:

- **C++ -> Python**: `PyIfFunc` function pointers. C++ code calls a registered Python function without linking to Python directly.
- **Python -> C++**: Cython `.pxd` declarations auto-generated from annotated C++ headers (the `pxd:` comments you'll see in headers).
- **`PyObj`**: Wraps Python objects for storage in C++ data structures.

The `libopenage` library deliberately does **not** link against Python — it uses function pointer registration so Python is optional (headless mode works without it).

---

## Recent Development (from the blog)

The most recent updates (mid-2024 through mid-2025) show active progress:

- **June 2025**: Game entity interactions completed — units can attack, convert, and interact. The activity system (behavior state machines) was made data-driven. Added an ID texture for mouse-click targeting. Curve compression to prevent animation jitter.
- **August 2024**: `ApplyEffect` ability type for batched effects (damage + conversion simultaneously). Working toward entity collision systems.
- **July 2024**: Frustum culling added (major rendering optimization). Release 0.6.0 completed.
- **May-June 2024**: Flow field pathfinding integrated into simulation. Fixed diagonal path blocking and sector accessibility bugs. 2-4x pathfinding performance improvement.

The project is actively developed, with the main developer (heinezen) and community contributors making steady progress. The current frontier is completing entity interactions (combat, gathering, construction) and unit death mechanics.

---

## Summary: What's Working vs. What's Not

| Subsystem | Status |
|-----------|--------|
| Rendering pipeline | Working (demos run) |
| Camera + frustum culling | Working |
| Pathfinding (flow fields) | Working |
| Event system | Working |
| Curve system | Working |
| Input handling | Working |
| Time management | Working |
| Asset conversion (AoE1/2) | Working |
| nyan data loading | Working |
| Entity creation + components | Partial (crashes on player creation) |
| Entity interactions (combat) | In development (recent blog posts) |
| Gameplay loop | Not functional |
| Audio | Stubbed out |

The engine is essentially a car with a great chassis, engine block, and transmission, but the steering wheel and pedals aren't connected yet. The demos let you see the individual systems working; the `game` command tries to wire everything together and hits the unfinished parts.
