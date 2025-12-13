# Core Architectural Patterns

**Date**: 2025-12-14

## Pattern 1: Dual LOD System

Rain World's most distinctive pattern - entities exist in two forms:

### Abstract LOD (Distant)
```
AbstractPhysicalObject
├─ ID: EntityID
├─ pos: WorldCoordinate
├─ Update(abstractTime) - Variable timestep
└─ Lightweight simulation
```

**When**: Entity in unloaded room or too far from player
**Performance**: Updates every few frames with variable time delta
**Data**: Position, basic state, relationships

### Realized LOD (Nearby)
```
PhysicalObject
├─ abstractPhysicalObject: AbstractPhysicalObject
├─ bodyChunks: BodyChunk[]
├─ Update() - Fixed 40 TPS
└─ Full physics simulation
```

**When**: Entity in loaded room near player
**Performance**: Every frame, full physics
**Data**: BodyChunks, collision, graphics, detailed AI

### Transitions

**Abstract → Realized** (`Realize()`)
1. AbstractPhysicalObject.Realize() called
2. Creates PhysicalObject from abstract state
3. Links: `abstractPhysicalObject.realizedObject = physicalObject`
4. Adds to room's update list

**Realized → Abstract** (`Abstractize()`)
1. PhysicalObject.Abstractize() called
2. Serializes state back to AbstractPhysicalObject
3. **Non-persistent entities discarded** (particles, effects)
4. Removes from room, keeps abstract form

### Tradeoffs

**Benefits**:
- Simulate hundreds of creatures across entire region
- Only pay physics cost for nearby entities
- Ecosystem feels alive beyond player's view

**Costs**:
- Data loss on transitions (momentum, temporary states)
- Synchronization bugs between abstract/realized
- Complexity in managing two representations

---

## Pattern 2: ProcessManager State Machine

Clean game state management:

```
ProcessManager
├─ currentMainLoop: MainLoopProcess
├─ RequestMainProcessSwitch(newProcess)
└─ Update()
```

**States**:
- `MenuProcess` - Main menu, settings
- `RainWorldGame` - Actual gameplay
- `SlideShowMenuProcess` - Cutscenes
- `FastTravelScreen` - Region transitions

**Transitions**:
```
Menu → RainWorldGame  (start campaign)
RainWorldGame → SlideShowMenuProcess (death/sleep)
SlideShowMenuProcess → Menu (game over)
```

**Strength**: Each process is self-contained. No mode-specific `if` statements scattered everywhere.

---

## Pattern 3: Room-based Streaming

World is divided into discrete rooms:

```
World
├─ abstractRooms: AbstractRoom[] (all rooms in region)
└─ activeRooms: Room[] (currently loaded)
```

### Loading Pipeline

1. **Player approaches room**
2. **RoomPreparer spawned** - Background loading thread
3. **Geometry loaded** - Tiles, terrain
4. **AI maps generated** - Pathfinding data
5. **Room.Loaded()** - Marked ready
6. **Entities realized** - AbstractPhysicalObjects → PhysicalObjects

### Progressive Loading

```
Frame N:   Player in Room A
Frame N+1: Player moves toward exit to Room B
Frame N+2: RoomPreparer(Room B) started
...
Frame N+50: Room B loaded, entities realized
Frame N+51: Player enters Room B
```

Rooms load ahead of player movement.

### Memory Management

- Only ~3-5 rooms loaded at once
- Distant rooms unload (entities → abstract)
- Keeps memory footprint constant

**Issue**: Hard room boundaries. Creatures can't see through doors, projectiles stop at exits.

---

## Pattern 4: Graphics/Logic Separation

Physical simulation separate from rendering:

```
PhysicalObject (logic, physics, AI)
  ↓ (one-to-many)
GraphicsModule (rendering, animation)
  ↓
PlayerGraphics, LizardGraphics, etc.
```

### Why Separate?

- **Physics**: Fixed 40 TPS
- **Graphics**: Variable 60 FPS
- **Interpolation**: Graphics interpolate between physics frames

### Example: Creature

```
Creature (PhysicalObject)
├─ bodyChunks[] - Physics positions
├─ AI.Update() - Behavior
└─ graphicsModule: CreatureGraphics
    ├─ DrawSprites() - Render
    └─ Interpolate physics frames
```

**Benefit**: Can render at different framerates than simulation. Graphics mods don't affect physics.

---

## Pattern 5: BodyChunks Physics

Simple, stable physics system:

```
BodyChunk
├─ pos: Vector2
├─ vel: Vector2
├─ mass: float
└─ rad: float (circle radius)
```

**All collision is circle-based**. Complex creatures are multiple circles connected by constraints.

### Example: Lizard

```
Lizard
├─ bodyChunks[0] - Head
├─ bodyChunks[1] - Body
├─ bodyChunks[2] - Hips
└─ bodyChunks[3] - Tail

Constraints:
- bodyChunks[0] ↔ bodyChunks[1]: distance = 15
- bodyChunks[1] ↔ bodyChunks[2]: distance = 20
- etc.
```

### Physics Loop

1. Apply forces (gravity, input)
2. Integrate velocities
3. Check terrain collision (circles vs tiles)
4. Resolve constraints (maintain distances)
5. Iterate until stable

**Tradeoff**: Simple and stable, but less realistic than box collision or proper skeletal animation.

---

## Pattern 6: UpdatableAndDeletable Lifecycle

Base class for all game objects:

```
UpdatableAndDeletable
├─ Update()
├─ Destroy()
├─ slatedForDeletetion: bool
└─ room: Room
```

**Lifecycle**:
1. Created, added to room.updateList
2. Update() called each frame
3. When done: `slatedForDeletetion = true`
4. Room removes from updateList next frame

**Benefit**: Consistent lifecycle. No manual memory management. Objects clean themselves up.

---

## How Patterns Interact

```
ProcessManager
  └─ RainWorldGame
      └─ World (Pattern 3: Room streaming)
          └─ Room
              └─ PhysicalObject (Pattern 6: Lifecycle)
                  ├─ BodyChunks (Pattern 5: Physics)
                  ├─ GraphicsModule (Pattern 4: Graphics separation)
                  └─ AbstractPhysicalObject (Pattern 1: Dual LOD)
```

Each pattern serves a specific purpose. Together they enable Rain World's unique ecosystem simulation.

## Related Documents

- [Architecture Overview](overview.md)
- [Singleton Coupling Problem](problems/singleton-coupling.md) - How global access undermines these patterns
