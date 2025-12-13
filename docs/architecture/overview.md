# Current Architecture Overview

**Date**: 2025-12-14
**Status**: Analysis

## High-Level Structure

Rain World's architecture centers around several key components:

```
RainWorld (singleton)
  └─ ProcessManager (state machine)
      └─ MainLoopProcess (abstract base)
          ├─ MenuProcesses (UI states)
          └─ RainWorldGame (gameplay)
              └─ World (region)
                  ├─ AbstractRoom[] (all rooms)
                  ├─ Room[] (loaded/realized)
                  └─ AbstractPhysicalObject[] (entities)
```

## Core Systems

### 1. RainWorld Singleton
- Central game instance
- Manages resources, options, sound
- **Issue**: Global access creates coupling

### 2. ProcessManager
- State machine for game modes
- Manages transitions between menu, gameplay, sleep screens
- Clean separation of concerns
- **Strength**: Well-designed pattern

### 3. Dual LOD (Level of Detail)
- **Abstract LOD**: Lightweight simulation for distant areas
- **Realized LOD**: Full physics/graphics for nearby areas
- **Innovation**: Enables large ecosystem simulation
- **Issue**: Data loss during transitions

### 4. Room-based World
- World divided into rooms
- Progressive loading as player moves
- Creatures/items persist in abstract form
- **Strength**: Memory efficient streaming
- **Issue**: Hard room boundaries affect gameplay

### 5. Graphics/Logic Separation
- PhysicalObject (logic) separate from GraphicsModule (rendering)
- One-to-many relationship possible
- **Strength**: Clean separation, moddable

## Entity Hierarchy

```
UpdatableAndDeletable (lifecycle)
  └─ PhysicalObject (physics)
      ├─ Creature
      │   ├─ Player
      │   ├─ Lizard variants
      │   ├─ Scavenger
      │   └─ ...
      └─ Items (Spears, Rocks, etc.)
```

**Issue**: Deep inheritance makes adding cross-cutting behaviors difficult.

## Update Loop

```
ProcessManager.Update()
  └─ currentMainLoop.Update()
      └─ RainWorldGame.Update()
          ├─ World.Update(abstract entities)
          └─ Room.Update(realized entities)
              └─ PhysicalObject.Update()
                  └─ AI.Update()
```

**60 FPS rendering, 40 TPS physics** - Decoupled update rates.

## Key Design Decisions

| Decision | Rationale | Tradeoff |
|----------|-----------|----------|
| Dual LOD | Simulate large ecosystem | Data loss on transitions |
| Room-based | Memory efficiency | Hard boundaries |
| Singleton | Convenience | Tight coupling |
| Deep inheritance | Natural OOP | Rigidity |
| BodyChunks physics | Simple, stable | Less realistic |

## Related Documents

- [Core Patterns](core-patterns.md) - Deep dive on key patterns
- [Singleton Coupling Problem](problems/singleton-coupling.md)
- [Inheritance Rigidity Problem](problems/inheritance-rigidity.md)
