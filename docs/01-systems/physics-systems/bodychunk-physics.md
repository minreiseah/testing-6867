# BodyChunk Physics System

**Layer**: 1 (Systems Architecture)
**Reading time**: 18 minutes

## Purpose

This document explains Rain World's custom circle-based soft-body physics system using BodyChunks, and why it was chosen over traditional polygon physics engines.

## The Core Concept

Instead of using Unity Physics or a traditional physics engine, Rain World implements a **custom BodyChunk system** where:

- Every PhysicalObject is composed of **circles** (BodyChunks)
- Circles connect via **springs** (BodyChunkConnections)
- All collision is **circle-based** (circle-circle, circle-terrain)
- Results in **soft-body physics** with natural deformation

## What Is a BodyChunk?

A BodyChunk is fundamentally a **circle with physics properties**:

```csharp
BodyChunk {
    pos: Vector2           // Position in world
    lastPos: Vector2       // Previous position (for Verlet integration)
    vel: Vector2           // Velocity
    mass: float            // Mass (affects physics)
    rad: float             // Radius (collision size)
    collideWithTerrain: bool
    collideWithObjects: bool
    goThroughFloors: bool
}
```

**That's it**. No complex polygon shapes, no rigid body dynamics, just circles.

## Why Circles Instead of Polygons?

### Traditional Physics Engines (Box2D, Unity Physics)

**Approach**:
- Polygon colliders (rectangles, arbitrary shapes)
- Rigid body dynamics (objects don't deform)
- Constraint solvers for joints

**Good for**:
- Realistic physics simulations
- Rigid objects (crates, vehicles)
- Complex mechanical systems

**Problems for Rain World**:
- **Expensive**: Testing polygon-polygon collision is O(n²) in edges
- **Rigid**: Creatures don't squish through tight spaces naturally
- **Complex**: Hard to debug, hard to make deterministic
- **Overkill**: Don't need realistic physics, need responsive gameplay

### Rain World's Approach

**Approach**:
- Circle colliders only
- Soft-body physics (circles connected by springs)
- Simple constraint solver (5 iterations)

**Benefits**:
- **Fast**: Circle-circle collision is trivial (distance < r1+r2)
- **Squishy**: Creatures deform naturally through tight spaces
- **Simple**: Easy to debug, easy to make deterministic
- **Performant**: Can simulate dozens of creatures simultaneously

**Tradeoff**:
- Less realistic than polygon physics
- Can't represent complex rigid shapes accurately
- Creatures are "blobby" rather than solid

## BodyChunk Examples

### Slugcat (Player): 2 BodyChunks

```
    O  ← Head (bodyChunks[0])
    |     mass: 0.4, rad: 4
    O  ← Body (bodyChunks[1])
          mass: 0.6, rad: 5

Connected by spring (distance = 8)
```

**Behavior**:
- Squishes when squeezed
- Head and body move semi-independently
- Natural-looking ragdoll when falling

### Lizard: 3+ BodyChunks

```
 O—O—O  ← Head, Body, Hips
        mass varies, rad varies
    └─O  ← Tail
```

**Behavior**:
- Serpentine movement
- Body follows head naturally
- Tail drags behind
- Squeezes through pipes

### Vulture: 10+ BodyChunks

```
     O          ← Head
    /|\         Main body chunks
   O O O        (large, heavy)
  O  O  O       Wings (lighter)
```

**Behavior**:
- Large soft body
- Wings flap semi-realistically
- Deforms on impact

## How BodyChunks Work

### 1. Integration (Apply Forces)

Every frame @ 40 TPS:

```csharp
// Apply gravity
bodyChunk.vel.y -= gravity * mass;

// Apply user input (for player)
bodyChunk.vel += inputForce;

// Verlet integration (no explicit velocity)
Vector2 temp = bodyChunk.pos;
bodyChunk.pos += (bodyChunk.pos - bodyChunk.lastPos) + acceleration;
bodyChunk.lastPos = temp;
```

**Verlet integration**:
- Position-based (more stable than velocity-based)
- Implicit velocity: `vel = pos - lastPos`
- Deterministic and easy to debug

### 2. Terrain Collision

Check each BodyChunk against tile grid:

```csharp
// For each tile near bodyChunk.pos:
if (tile.solid && circleOverlapsTile(bodyChunk, tile)) {
    // Push bodyChunk out of tile
    Vector2 pushDir = closestPointOnTile - bodyChunk.pos;
    bodyChunk.pos -= pushDir.normalized * overlapDistance;
}
```

**Tile collision**:
- Simple ray tests
- Push circles out of solid tiles
- Handle slopes (angled tiles push at angle)

### 3. Object-Object Collision

Only within same collision layer:

```csharp
foreach (other in sameLayer) {
    float dist = Vector2.Distance(chunk1.pos, chunk2.pos);
    if (dist < chunk1.rad + chunk2.rad) {
        // Collision! Push apart
        Vector2 pushDir = (chunk2.pos - chunk1.pos).normalized;
        float overlap = (chunk1.rad + chunk2.rad) - dist;

        chunk1.pos -= pushDir * overlap * 0.5;
        chunk2.pos += pushDir * overlap * 0.5;
    }
}
```

**Circle-circle collision**:
- O(1) per pair (just distance check)
- Split push force based on mass
- Happens after terrain collision

### 4. Constraint Solving

**BodyChunkConnections** act as springs:

```csharp
BodyChunkConnection {
    chunk1, chunk2: BodyChunk
    distance: float        // Rest length
    elastic: float         // Stiffness (0-1)
}

// Each frame, pull chunks toward rest distance
Vector2 dir = chunk2.pos - chunk1.pos;
float currentDist = dir.magnitude;
float error = currentDist - connection.distance;

chunk1.pos += dir * error * 0.5 * elastic;
chunk2.pos -= dir * error * 0.5 * elastic;
```

**Iterative solving**:
- Run 5 iterations per frame
- Each iteration pulls chunks closer to rest distance
- Converges to stable configuration

### 5. Final Position

After all constraints:
```csharp
bodyChunk.vel = bodyChunk.pos - bodyChunk.lastPos;
```

Velocity is implicit (derived from position change).

## Collision Layers

To avoid O(n²) collision checks, BodyChunks are divided into **three layers**:

| Layer | Contains | Collides With |
|-------|----------|---------------|
| **0** | Most creatures, large items | Layer 0 only |
| **1** | Small items, some creatures | Layer 1 only |
| **2** | Special cases | Layer 2 only |

**Optimization**:
- Instead of checking all chunks vs all chunks: O(n²)
- Check layer 0 vs layer 0, layer 1 vs layer 1, layer 2 vs layer 2
- Approximately O(n²/3) = 3× faster

**Why this works**:
- Large creatures don't need to collide with every small item
- Small items don't need to collide with every large creature
- Special cases (spears mid-flight) get their own layer

## Soft-Body Behavior

The magic of BodyChunks is **soft-body deformation**:

### Squeezing Through Tight Spaces

```
Before:      After:
  |O|          | O |
  |O|    →     |  O|
              Chunks compress, then restore
```

**How it works**:
1. BodyChunks push against walls (terrain collision)
2. BodyChunkConnections try to maintain distance (spring constraint)
3. If space is tight, chunks compress
4. Once through, springs pull back to rest length
5. Result: Creature squishes through naturally

### Ragdoll Physics

When a creature dies or falls:
- No AI to control BodyChunks
- Gravity and collisions take over
- Chunks tumble independently
- Connections create natural-looking flailing
- Result: Organic ragdoll without special ragdoll system

### Impact Deformation

When creature hits wall at speed:
- Leading chunks collide first
- Trailing chunks catch up (momentum)
- Connections compress
- Then bounce back
- Result: Natural impact squish

## Performance Characteristics

### Cost per BodyChunk per Frame

- **Gravity**: 0.001ms
- **Integration**: 0.002ms
- **Terrain collision**: 0.02ms (depends on nearby tiles)
- **Object collision**: 0.01ms × chunks in layer
- **Constraint solving**: 0.005ms × connections

**Total**: ~0.04ms per chunk per frame

### Typical Scene

- **15 creatures** × 3 chunks avg = 45 chunks
- **5 items** × 1 chunk = 5 chunks
- **Total**: 50 chunks

**Physics cost**: 50 × 0.04ms = **2ms per frame**

At 40 TPS (25ms per frame): Physics uses ~8% of frame budget.

### Scaling

**Good scaling** (linear in chunk count):
- Small creatures (2-3 chunks): Very cheap
- Medium creatures (5-8 chunks): Moderate
- Large creatures (10-15 chunks): Still reasonable

**Bad scaling** (quadratic in creature count if all in one layer):
- 10 creatures in layer 0: 100 collision checks
- 20 creatures in layer 0: 400 collision checks
- Solution: Spread across layers, spatial partitioning

## Why This Works for Rain World

### Requirements Rain World Has

1. **Many creatures**: Need to simulate 10-20 at once
2. **Soft bodies**: Creatures squish through pipes, feel organic
3. **Determinism**: Same inputs → same outputs (for fair difficulty)
4. **Performance**: Must run on consoles, lower-end PCs
5. **Responsiveness**: Tight platforming controls

### How BodyChunks Deliver

1. **Many creatures**: Circle collision is fast enough for 15+ creatures
2. **Soft bodies**: Spring connections create natural squishiness
3. **Determinism**: Simple math, no randomness, reproducible
4. **Performance**: 2ms for 50 chunks is excellent
5. **Responsiveness**: Direct force application, no complex constraints

### What BodyChunks Sacrifice

1. **Realism**: Less realistic than polygon physics
2. **Rigid objects**: Can't represent solid boxes well
3. **Complex shapes**: Everything is circles
4. **Advanced physics**: No friction models, air resistance, etc.

**Conclusion**: The tradeoffs are worth it for Rain World's needs.

## Comparison to Traditional Physics

| Feature | Unity Physics | BodyChunk Physics |
|---------|---------------|-------------------|
| **Collider types** | Box, circle, polygon, mesh | Circle only |
| **Collision algorithm** | SAT, GJK | Distance check |
| **Rigid bodies** | Yes | No (all soft-body) |
| **Joints** | Many types | Spring only |
| **Performance (50 objects)** | ~5-8ms | ~2ms |
| **Soft-body support** | Via complex constraints | Built-in |
| **Determinism** | Hard to guarantee | Easy |
| **Learning curve** | Moderate | Low |

## Design Tradeoffs

| Decision | Benefit | Cost |
|----------|---------|------|
| **Circle-only collision** | Fast, simple, deterministic | Less realistic shapes |
| **Verlet integration** | Stable, implicit velocity | Less intuitive than Euler |
| **Soft-body by default** | Natural squishiness | No true rigid bodies |
| **Collision layers** | 3× faster collision | Must categorize objects carefully |
| **5 constraint iterations** | Good convergence | Some jitter possible |
| **Custom physics (not Unity)** | Control, performance, determinism | Must implement everything yourself |

## Related Documentation

**Same layer**:
- [Collision Detection](collision-detection.md) - Detailed collision algorithms
- [Terrain Collision](terrain-collision.md) - Tile-based collision
- [Physics Constraints](physics-constraints.md) - Spring connections

**Go deeper**:
- [Layer 2: Collision Algorithms](../../02-implementation/algorithms/collision-algorithms.md)
- [Layer 3: Physics Integration](../../03-technical/algorithms-deep-dive/physics-integration.md)
- [Layer 3: Struct vs Class](../../03-technical/csharp-specifics/structs-vs-classes.md) - Why BodyChunk is a struct

**Related topics**:
- [Update Loop](../core-architecture/update-loop.md) - When physics updates
- [Design Decisions: Why BodyChunks?](../design-decisions/why-bodychunks.md)

## Key Takeaways

1. **Circles only**: All collision is circle-based for performance
2. **Soft-body default**: Spring connections create natural squishiness
3. **Fast**: Circle collision is O(1), enables many creatures
4. **Deterministic**: Simple math, reproducible results
5. **Verlet integration**: Position-based, stable simulation
6. **Collision layers**: 3 layers avoid O(n²) checks
7. **Tradeoff**: Less realistic physics, but perfect for Rain World's needs
