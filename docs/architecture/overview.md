# Rain World Architecture Overview

**Reading time: 10-15 minutes**

This document explains Rain World's complete architecture from first principles. By the end, you'll understand why the codebase is structured the way it is and how all the pieces fit together.

## The Core Problem

Rain World's design goal creates an unusual technical challenge: simulate a living ecosystem where creatures hunt, migrate, and interact across dozens of interconnected rooms, while the player smoothly explores this world on limited hardware.

Traditional approaches fail here:

- **Simulate everything always**: Impractical. With 50+ rooms containing dozens of creatures each, simulating every lizard's detailed movement and AI would crush performance.
- **Only simulate what's visible**: Breaks immersion. If creatures only exist when you see them, the world feels fake. A lizard you saw hunting in one room should still be there (or have moved somewhere logical) when you return.
- **Traditional LOD**: Designed for graphics (lower-res models at distance), not AI and physics. Rain World needs creatures to *think* and *interact* even when far away, just less precisely.

Rain World's solution: **a dual-world architecture** where entities exist in two forms simultaneously—detailed when nearby, abstract when far away—with seamless transitions between the two.

## The Three-Tier Hierarchy

Rain World's code is organized as a three-level hierarchy:

```
RainWorld (singleton, lifecycle manager)
  └─ ProcessManager (state machine)
      └─ MainLoopProcess (current game state)
          ├─ Menu (various menu screens)
          └─ RainWorldGame (actual gameplay)
              └─ World (current region)
                  └─ Rooms (streaming level geometry)
                      └─ Entities (creatures, items, etc.)
```

Let's examine each level.

### Level 1: RainWorld Singleton

The `RainWorld` class is a Unity MonoBehaviour that persists for the entire game session. It's the root of everything—menus, gameplay, options, shaders, sound system, modding hooks. This is Rain World's "kernel."

**Why a singleton?** Convenience. From anywhere in the codebase, you can access `rainWorld.options`, `rainWorld.processManager`, or `rainWorld.soundSystem`. The downside: tight coupling. Every class that touches the singleton becomes harder to test and modify. (See [Singleton Coupling Problem](problems/singleton-coupling.md) for analysis.)

**What it manages:**
- The ProcessManager (next level down)
- Global resources (shaders, options, setup values)
- Modding lifecycle hooks (OnModsInit, PreModsEnabled, etc.)
- Resource loading/unloading

The RainWorld instance does minimal work itself. It's primarily a container and lifecycle coordinator, delegating actual game logic to ProcessManager.

### Level 2: ProcessManager (State Machine)

The ProcessManager implements a **state machine** for high-level game modes. At any moment, Rain World is in exactly one state:

- Main menu
- Story mode gameplay
- Arena mode gameplay
- Sleep screen cutscene
- Death screen
- Save file selection
- Credits
- Music player (special persistent state)

This is one of Rain World's cleanest architectural patterns. ProcessManager handles state transitions with fade effects, manages subprocess lifecycle, and routes update calls.

**Key insight**: Menus and gameplay are just different `MainLoopProcess` subclasses. The ProcessManager doesn't care whether it's running a menu or a game—it just calls `currentMainLoop.Update()` every frame.

**Update cycle**:
- The ProcessManager maintains a target FPS (usually 40) for game logic
- `RawUpdate()` is called every render frame (capped at 60 FPS)
- `Update()` is called at the stable 40 FPS rate for deterministic simulation
- Physics runs at 40 TPS (ticks per second), graphics at 60 FPS

This separation allows smooth rendering even if game logic runs at a lower rate, saving CPU for ecosystem simulation.

### Level 3: MainLoopProcess (Game State)

Each game mode is a `MainLoopProcess` subclass. The most important is `RainWorldGame`, which handles actual gameplay.

**RainWorldGame** manages:
- The World (current region being played)
- RoomCamera (what the player sees)
- GameSession (story mode vs arena mode logic)
- Global rain timer (the countdown to deadly rain)
- Player entities

When you sleep in a shelter, the game doesn't just reload—it tears down the entire RainWorldGame process and creates a new one for the next cycle. This ensures clean state between cycles.

## The Dual-World System: Abstract vs Realized

This is Rain World's most distinctive architectural feature.

Every entity (creature, item, etc.) exists in **two forms**:

### 1. Abstract Form (Always Active)

The abstract layer is a lightweight simulation that runs for the **entire region**, all the time. When a lizard is five rooms away:

- It exists as an `AbstractCreature` with an `AbstractCreatureAI`
- It moves between rooms based on probability models ("should I migrate?")
- It tracks basic state (hunger level, position, which room it's in)
- It can interact with other abstract entities (eating, fighting) via simplified dice-roll logic
- It updates every few frames instead of every frame

Think of the abstract layer as the game's "strategic view." Creatures are chess pieces moving across a board of rooms.

### 2. Realized Form (Only When Near Player)

When an entity is in or near the player's current room, it "realizes"—it spawns a full-detail version:

- The `AbstractCreature` creates a `Creature` instance (the realized form)
- This Creature has a full `ArtificialIntelligence` with pathfinding, behavior state machines, and detailed decision-making
- It has `BodyChunks` (physics circles) for collision and movement
- It has a `GraphicsModule` for rendering
- It updates every frame with full physics and AI

Think of the realized layer as the game's "tactical view." This is where the actual platforming, combat, and detailed creature behavior happens.

### Seamless Transitions

The magic is in the transition:

**Realization** (Abstract → Realized):
When the player enters a room with abstract entities:
1. The Room realizes (loads geometry, initializes physics)
2. Each AbstractPhysicalObject in that room realizes (spawns its Creature/Item)
3. The realized object reads initial state from its abstract counterpart
4. Graphics modules initialize for on-screen entities

**Abstraction** (Realized → Abstract):
When the player leaves a room:
1. Each realized entity saves relevant state back to its abstract form
2. Realized objects are destroyed (freeing memory)
3. The Room de-realizes (unloads geometry)
4. Abstract entities continue existing and updating in the background

**Critical detail**: Data not explicitly saved to the abstract form is lost. For example, a lizard's current animation state or exact position within a room isn't preserved—only its room location and essential state (health, hunger, etc.). This is a tradeoff: persistent ecosystem simulation costs memory.

## Room-Based World Streaming

The World is divided into **Rooms** (usually 1-4 screen sizes each). The game streams rooms in and out as needed.

### How It Works

**At any moment**:
- All rooms exist in abstract form (as `AbstractRoom` objects)
- 1-3 rooms are realized (the player's room + adjacent rooms being pre-loaded)
- The rest are dormant (just data, no active simulation)

**Progressive Loading**:
When you move toward a room boundary:
1. The game starts a `RoomPreparer` to realize the next room in the background
2. While you're still in the current room, the next room loads geometry and realizes entities
3. When you cross the boundary, the transition is seamless
4. The room you just left stays realized briefly (in case you go back)
5. If you move further away, it abstracts to free memory

**Why room-based?** Alternatives were worse:

- **Tile-based**: Too granular, too much overhead managing thousands of tiles
- **One huge level**: Can't fit entire regions in memory on target hardware
- **Portal/sector system**: More complex, harder to author

Rooms are large enough to feel expansive but small enough to stream efficiently. The compromise: visible room boundaries in level design.

## BodyChunk Physics System

Rain World doesn't use Unity's built-in physics (or any traditional physics engine). Instead, it implements a custom **BodyChunk** system.

### What Are BodyChunks?

Every `PhysicalObject` is composed of one or more `BodyChunk` instances. Each BodyChunk is:
- A **circle** (not a polygon) with a radius and mass
- A point with position, velocity, and acceleration
- Part of a **collision layer** (0, 1, or 2)

Examples:
- **Slugcat**: 2 body chunks (head and torso)
- **Lizard**: 3+ body chunks (head, body segments, tail)
- **Spear**: 1 body chunk
- **Vulture**: 10+ body chunks forming the large body

Chunks connect via `BodyChunkConnection` objects that act like springs, creating soft-body physics.

### Why Not Standard Physics?

Traditional physics engines (Box2D, Havok, Unity Physics) use:
- Polygon colliders (complex shapes)
- Rigid body dynamics (objects don't deform easily)
- Constraint solvers (for joints and connections)

This works great for realistic physics, but Rain World needed:
- **Squishiness**: Creatures should deform when squeezed through tight spaces
- **Performance**: Simulating dozens of creatures with complex shapes is expensive
- **Determinism**: Rain World's cycles must be perfectly repeatable for fair gameplay
- **Simplicity**: Circle-circle collision is much faster than polygon-polygon

**The tradeoff**: Less realistic physics, but creatures feel organic and weighty. A lizard squeezing through a pipe deforms naturally. A slugcat tumbling down a slope bounces and rolls believably. The BodyChunk system achieves this with minimal CPU cost.

### Collision Layers

The three collision layers prevent performance collapse:
- **Layer 0**: Most creatures
- **Layer 1**: Some items and small creatures
- **Layer 2**: Special cases

Objects only collide with others on the **same layer**. This cuts collision checks dramatically—instead of testing every object against every other object (O(n²)), you test within each layer.

Why this works: Creatures rarely need to collide with *every* other creature. A lizard doesn't need to collide with distant scavengers, only nearby threats and prey.

## Graphics Separation

Rain World separates **logic** from **rendering** via the `GraphicsModule` system.

### How It Works

- `PhysicalObject` handles physics, AI, and game logic
- `GraphicsModule` (optional) handles rendering, animations, and visual effects
- They're separate objects with a one-to-one relationship

**Lifecycle**:
1. A Creature is created (logic exists)
2. When the room becomes visible on camera, `InitiateGraphicsModule()` is called
3. The Creature creates its GraphicsModule (e.g., `LizardGraphics`)
4. The GraphicsModule reads the Creature's state and renders it
5. When the camera moves away, `DisposeGraphicsModule()` destroys the graphics
6. The Creature continues existing (logic) without rendering overhead

**Why separate?**

- **Memory**: Graphics modules use significant RAM (sprite states, shaders, rendering data). Creatures off-screen don't need them.
- **Moddability**: You can replace a creature's graphics without touching its logic
- **Performance**: Skip rendering entirely for off-screen entities

**Tradeoff**: Some complexity. The graphics module must continuously sync with the logic object. If they get out of sync, visual bugs appear (e.g., a creature's sprite at the wrong position).

## The Complete Update Loop

Here's what happens each frame:

```
1. Unity calls RainWorld.Update()
   └─ 2. RainWorld calls ProcessManager.Update()
       └─ 3. ProcessManager calls currentMainLoop.Update()
           └─ 4. RainWorldGame.Update() executes:
               ├─ Update rain cycle timer
               ├─ Update global processes (rain, sound)
               └─ 5. World.Update() executes:
                   ├─ Update all AbstractRooms (abstract creatures, lineages, spawners)
                   ├─ 6. For each realized Room:
                   │   ├─ Update all UpdatableAndDeletable entities
                   │   ├─ 7. For each PhysicalObject:
                   │   │   ├─ Update AI (if creature)
                   │   │   ├─ Update physics (integrate velocity, apply gravity)
                   │   │   ├─ Resolve collisions (terrain and other objects)
                   │   │   └─ Update graphics module (if visible)
                   │   └─ Update room processes (water, background elements)
                   └─ Check for room realization/abstraction
```

Abstract creatures update less frequently (every ~5 frames), while realized entities update every frame. This is why you can simulate an entire region—most of it runs in the cheap abstract mode.

## Why This Architecture?

Every choice trades something:

| Decision | Benefit | Cost |
|----------|---------|------|
| Singleton RainWorld | Convenient global access | Tight coupling, hard to test |
| ProcessManager state machine | Clean mode separation | Teardown/setup overhead between states |
| Dual LOD (Abstract/Realized) | Simulate entire ecosystem | Data loss on transitions, complexity |
| Room-based streaming | Memory efficient, smooth loading | Visible room boundaries in design |
| BodyChunk physics | Performance, squishiness, determinism | Less realistic physics |
| Graphics separation | Memory savings, moddability | Sync complexity |

These tradeoffs are **specific to Rain World's goals**. A different game (e.g., a racing game or puzzle game) would make different choices. Rain World's architecture exists to solve one problem: simulating a living ecosystem in a memory-constrained platformer.

## What This Enables

With this architecture, Rain World achieves:

- **Persistent ecosystem**: A lizard you encounter, evade, and flee from actually continues existing. If you return later, it might still be there—or it might have migrated to another room hunting for food.

- **Emergent narratives**: Because creatures interact abstractly even off-screen, you stumble onto aftermath scenes. A room full of scavenger corpses tells a story: a predator was here.

- **Fair challenge**: The rain timer is deterministic. The ecosystem simulation is deterministic (same cycle, same initial conditions = same outcomes). This makes the brutal difficulty feel fair.

- **Large interconnected world**: Regions like Industrial Complex have 70+ rooms. With streaming and abstract simulation, it all fits in memory and runs smoothly.

## Next Steps

Now that you understand the architecture, you can dive deeper:

- **Want to understand the game loop in detail?** → [The Game Loop](game-loop.md)
- **Curious about Abstract vs Realized specifics?** → [Dual LOD System](dual-lod.md)
- **Need to mod creature behavior?** → [Creature Intelligence](creature-ai.md)
- **Interested in the design patterns and problems?** → [Design Patterns](design-patterns.md)

Or jump to [Class Hierarchy Reference](class-hierarchy.md) if you want class diagrams and implementation details.

---

**This is the core knowledge.** Everything else in the documentation is elaboration on these fundamentals.
