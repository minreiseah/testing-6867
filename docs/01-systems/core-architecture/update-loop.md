# The Update Loop and Frame Timing

**Layer**: 1 (Systems Architecture)
**Reading time**: 15 minutes

## Purpose

This document explains how Rain World's update loop works, including frame timing, the separation between physics and graphics updates, and how abstract vs realized entities are updated at different rates.

## Two-Tier Update Rates

Rain World uses **different update rates for different systems**, a key architectural decision for performance and fairness:

| System | Rate | Why |
|--------|------|-----|
| **Physics & AI** | 40 TPS (ticks per second) | Deterministic simulation for fair gameplay |
| **Graphics Rendering** | 60 FPS (frames per second) | Smooth visual experience |
| **Abstract AI** | Every 3-5 frames (~8-13 TPS) | Strategic decisions don't need high frequency |

**Key insight**: Physics runs slower than graphics. Graphics interpolate between physics frames for smooth motion.

## Unity Update Methods

Rain World leverages three Unity update methods:

### RawUpdate()
- **Called**: Every render frame (capped at 60 FPS)
- **Purpose**: Input processing, immediate UI response
- **Timing**: Variable (depends on frame rate)
- **Used for**: Time-sensitive operations that need to respond immediately

### Update()
- **Called**: Fixed 40 TPS rate
- **Purpose**: Game logic, physics, AI
- **Timing**: Fixed timestep (deterministic)
- **Used for**: All simulation that affects gameplay

### Graphics Updates
- **Called**: During rendering pipeline
- **Purpose**: Sprite positioning, animation
- **Timing**: Interpolated between physics frames
- **Used for**: Visual-only updates

## The Complete Update Hierarchy

Here's what happens each frame:

```
Frame Start (Unity Engine)
│
├─ RawUpdate() @ ~60 FPS
│   └─ Input processing
│   └─ Immediate UI response
│
├─ Update() @ 40 TPS [DETERMINISTIC]
│   │
│   └─ 1. RainWorld.Update()
│       │
│       └─ 2. ProcessManager.Update()
│           │
│           └─ 3. currentMainLoop.Update()
│               │
│               ├─ If Menu: MenuProcess.Update()
│               │   └─ UI logic
│               │
│               └─ If Gameplay: RainWorldGame.Update()
│                   ├─ Update rain cycle timer
│                   ├─ Update global processes (rain, sound)
│                   │
│                   └─ 4. World.Update()
│                       │
│                       ├─ 5. Update AbstractRooms
│                       │   ├─ Update lineages (creature spawning)
│                       │   ├─ Update creature dens
│                       │   ├─ Update abstract creatures [EVERY 3-5 FRAMES]
│                       │   │   ├─ Abstract AI (migration decisions)
│                       │   │   ├─ Abstract physics (room-level movement)
│                       │   │   └─ Abstract interactions (simplified combat)
│                       │   └─ Update spawn events
│                       │
│                       ├─ 6. Update Realized Rooms [EVERY FRAME]
│                       │   │
│                       │   └─ For each Room:
│                       │       ├─ Update all UpdatableAndDeletable entities
│                       │       │
│                       │       ├─ 7. Update PhysicalObjects
│                       │       │   │
│                       │       │   └─ For each PhysicalObject:
│                       │       │       ├─ Update AI (if Creature)
│                       │       │       │   ├─ Sensory processing
│                       │       │       │   ├─ Behavior state machine
│                       │       │       │   └─ Decision making
│                       │       │       │
│                       │       │       ├─ Update physics
│                       │       │       │   ├─ Apply forces (gravity, input)
│                       │       │       │   ├─ Integrate velocities (Verlet)
│                       │       │       │   ├─ Resolve terrain collision
│                       │       │       │   ├─ Resolve object-object collision
│                       │       │       │   └─ Solve constraints (BodyChunkConnections)
│                       │       │       │
│                       │       │       └─ Update special behaviors
│                       │       │           └─ Item usage, combat, etc.
│                       │       │
│                       │       └─ Update room processes
│                       │           ├─ Water dynamics
│                       │           ├─ Background elements
│                       │           └─ Room-specific effects
│                       │
│                       └─ 8. Room Management
│                           ├─ Check for room realization (player approaching)
│                           ├─ Check for room abstraction (player left)
│                           └─ Update RoomPreparers (background loading)
│
└─ Graphics Update @ 60 FPS [VISUAL ONLY]
    │
    └─ For each visible PhysicalObject:
        └─ GraphicsModule.DrawSprites()
            ├─ Interpolate position between physics frames
            ├─ Update sprite state
            ├─ Update shaders
            └─ Submit to renderer
```

## Staggered Update Rates

Different entity types update at different frequencies:

### Abstract Entities: Every 3-5 Frames (~8-13 TPS)
```csharp
// Pseudocode
if (frame % abstractAI.updateDelay == 0) {
    abstractCreature.Update(abstractTime);
}
```

**Why slower?**
- Strategic decisions (migration, hunting targets) don't need high frequency
- Enables simulating hundreds of creatures across entire region
- Still frequent enough for responsive ecosystem

**What updates?**
- Migration between rooms
- Hunger/food seeking
- Simplified combat (dice rolls)
- Relationship tracking

### Realized Entities: Every Frame (40 TPS)
```csharp
// Pseudocode
creature.Update(); // Every physics frame
```

**Why every frame?**
- Tactical gameplay needs responsive controls
- Smooth creature movement and combat
- Frame-perfect platforming

**What updates?**
- Full AI (pathfinding, behavior trees)
- Physics simulation (BodyChunks, constraints)
- Collision detection and response
- Combat and interactions

### Graphics: 60 FPS (Interpolated)
```csharp
// Pseudocode
float t = (Time.time - lastPhysicsTime) / physicsTimestep;
Vector2 interpolatedPos = Vector2.Lerp(prevPos, currentPos, t);
```

**Why 60 FPS?**
- Smooth visual motion
- Responsive camera movement
- Better perceived performance

**What interpolates?**
- BodyChunk positions
- Sprite positions
- Camera movement
- Particle effects

## Determinism and Fairness

### Why 40 TPS for Physics?

Rain World's difficulty is brutal. To make it feel fair, the simulation must be **deterministic**:

- Same starting conditions → same outcomes
- No randomness from frame rate variance
- Precise, repeatable gameplay

**Fixed 40 TPS achieves this**:
- Consistent timestep every frame
- No accumulation of floating-point errors
- Players can learn enemy patterns reliably

### Why Not 60 TPS?

**Pros of 60 TPS**:
- More responsive controls
- Smoother physics

**Cons of 60 TPS**:
- 50% more CPU cost for physics
- Less CPU available for ecosystem simulation
- Tighter memory constraints

**Decision**: 40 TPS physics + 60 FPS graphics via interpolation gives smooth visuals with reasonable CPU cost.

## Performance Characteristics

### CPU Distribution (Estimated)

During typical gameplay:
- **40% - Realized entities** (physics, AI, collision)
- **15% - Abstract entities** (strategic AI, spawning)
- **20% - Graphics** (rendering, shaders)
- **10% - Room management** (streaming, loading)
- **15% - Other** (sound, UI, systems)

### Entity Update Cost

| Entity State | Updates/Second | CPU Cost (Relative) |
|--------------|----------------|---------------------|
| Abstract creature | 8-13 | 1× (baseline) |
| Realized creature (off-camera) | 40 | ~50× |
| Realized creature (on-camera) | 40 + graphics | ~70× |

**Key insight**: Most creatures are abstract most of the time. This enables simulating hundreds of creatures.

## Frame Budget Analysis

At 40 TPS, each frame has **25ms budget**:

| System | Time Budget | Notes |
|--------|-------------|-------|
| Abstract entities | ~6ms | ~200 creatures @ 0.03ms each |
| Realized entities | ~10ms | ~15 creatures @ 0.67ms each |
| Physics collision | ~4ms | 3 collision layers, spatial partitioning |
| Room management | ~2ms | Streaming, realization |
| Other systems | ~3ms | Sound, rain, camera |

**Total**: ~25ms = 40 FPS
**Slack**: Important to maintain 60 FPS graphics

## Optimization Techniques

### 1. Staggered Updates
Not all abstract entities update the same frame:
```csharp
updateDelay = 3 + (entityID % 2); // 3 or 4 frames
```
Spreads CPU load evenly.

### 2. Early Exit
```csharp
if (!hasTarget && !isHungry) {
    return; // Skip expensive pathfinding
}
```
Many AI checks bail out early.

### 3. LOD Within Realized
Even realized creatures optimize:
- **On-camera**: Full graphics, detailed animations
- **Off-camera**: Physics only, no graphics
- **Far from player in same room**: Simplified AI

### 4. Spatial Partitioning
Collision uses three layers:
- Only check collisions within same layer
- O(n/3)² instead of O(n)² complexity
- ~9× faster collision detection

## Update Flow for Different Scenarios

### Scenario 1: Player Standing Still

```
Frame 0:  Update realized entities in player's room (full)
          Update abstract entities across region (1/5)
          Graphics interpolate

Frame 1:  Update realized entities in player's room (full)
          Skip abstract (not their frame)
          Graphics interpolate

...repeat...
```

**CPU**: Moderate (only player's room simulated fully)

### Scenario 2: Player Moving Through Rooms

```
Frame 0:  Update Room A (realized, player here)
          Update Room B (being prepared in background)
          Update abstract entities
          Graphics

Frame 1:  Update Room A
          Room B nearly loaded
          Graphics

Frame 2:  Player crosses boundary
          Update Room A (still realized)
          Update Room B (now player here)
          Graphics

Frame 3:  Room A begins abstraction (player far)
          Update Room B
          Graphics
```

**CPU**: Higher (2 rooms realized temporarily)

### Scenario 3: Creature Realization

```
Frame 0:  Creature exists as AbstractCreature
          Updates every 3-5 frames
          No graphics, no physics

Frame 1:  Player approaches room
          RoomPreparer starts loading

Frame 5:  Room realizes
          AbstractCreature.Realize() called
          Creates Creature (PhysicalObject)
          Initializes BodyChunks
          Starts AI

Frame 6:  Camera sees creature
          GraphicsModule created
          Full update loop (AI + physics + graphics)
```

## Related Documentation

**Same layer**:
- [Overview](overview.md) - Complete architecture
- [Dual LOD System](dual-lod-system.md) - Abstract vs Realized details
- [Three-Tier Hierarchy](three-tier-hierarchy.md) - RainWorld → ProcessManager → Game

**Go deeper**:
- [Layer 2: Update Flow](../../02-implementation/data-flow/update-flow.md) - Frame-by-frame code execution
- [Layer 2: ProcessManager Implementation](../../02-implementation/systems-implementation/processmanager-impl.md) - How state machine routes updates
- [Layer 3: Unity Update Loop](../../03-technical/unity-specifics/unity-update-loop.md) - Update vs FixedUpdate vs LateUpdate

**Related topics**:
- [BodyChunk Physics](../physics-systems/bodychunk-physics.md) - What updates during physics phase
- [Creature Intelligence](../creature-systems/creature-intelligence.md) - What updates during AI phase
- [Graphics Separation](../graphics-systems/graphics-separation.md) - Graphics update lifecycle

## Key Takeaways

1. **Two-tier timing**: 40 TPS physics + 60 FPS graphics enables determinism + smoothness
2. **Staggered updates**: Abstract entities update every 3-5 frames to save CPU
3. **LOD drives performance**: Most entities are abstract most of the time
4. **Determinism matters**: Fixed timestep makes difficulty feel fair
5. **Interpolation hides cost**: Graphics run faster than physics seamlessly
