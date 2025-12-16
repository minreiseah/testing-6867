# Collision Detection

**Layer**: 1 (Systems Architecture)
**Reading time**: 15 minutes

## Purpose

This document explains Rain World's collision detection system: how circle-based BodyChunks collide with each other and with terrain, and why the three-layer system enables efficient collision at scale.

## The Challenge

With 15+ creatures in a room, each with 2-5 BodyChunks, you have **~50 chunks** that need collision detection.

**Naive approach**: Check every chunk against every other chunk
- Complexity: O(n²) = 50² = 2,500 checks per frame
- At 40 FPS: 100,000 checks per second
- Unacceptable performance

**Rain World's approach**: Collision layers + circle-circle optimization
- Complexity: ~O(n²/3) per layer + O(n) terrain
- 50 chunks across 3 layers: ~850 checks per frame
- At 40 FPS: 34,000 checks per second
- **3× faster** than naive approach

## Collision Types

Rain World has two distinct collision systems:

### 1. Terrain Collision (Chunk vs Tiles)
- **What**: BodyChunk against tile grid
- **When**: Every frame for every chunk
- **Cost**: O(n) where n = number of chunks
- **Purpose**: Keep creatures out of walls, on floors

### 2. Object Collision (Chunk vs Chunk)
- **What**: BodyChunk against other BodyChunks
- **When**: Every frame, only within same layer
- **Cost**: O(n²) per layer, but split across 3 layers
- **Purpose**: Creatures push each other, combat, physics

## Terrain Collision

### The Tile Grid

Rain World's levels are built from a **20×20 pixel tile grid**:
- Each tile is solid, air, slope, or special
- Tiles stored as 2D array: `room.Tiles[x, y]`
- Simple and fast to query

### Circle-Tile Collision

For each BodyChunk:

```csharp
// 1. Find nearby tiles
int tileX = (int)(chunk.pos.x / 20);
int tileY = (int)(chunk.pos.y / 20);

// 2. Check 3×3 grid around chunk
for (int x = tileX - 1; x <= tileX + 1; x++) {
    for (int y = tileY - 1; y <= tileY + 1; y++) {
        if (room.GetTile(x, y).Solid) {
            // 3. Check if circle overlaps tile
            if (CircleOverlapsTile(chunk, x, y)) {
                // 4. Push chunk out
                PushChunkOutOfTile(chunk, x, y);
            }
        }
    }
}
```

### Circle-Tile Overlap Test

```csharp
bool CircleOverlapsTile(BodyChunk chunk, int tileX, int tileY) {
    // Find closest point on tile to circle center
    Vector2 tilePos = new Vector2(tileX * 20, tileY * 20);
    Vector2 closest = new Vector2(
        Mathf.Clamp(chunk.pos.x, tilePos.x, tilePos.x + 20),
        Mathf.Clamp(chunk.pos.y, tilePos.y, tilePos.y + 20)
    );

    // Distance from circle center to closest point
    float distance = Vector2.Distance(chunk.pos, closest);

    // Overlap if distance < radius
    return distance < chunk.rad;
}
```

**Optimization**: Only check 9 tiles (3×3 grid) instead of all tiles in room.

### Push Resolution

When a chunk overlaps a tile:

```csharp
void PushChunkOutOfTile(BodyChunk chunk, int tileX, int tileY) {
    // Find push direction
    Vector2 tileCenter = new Vector2(tileX * 20 + 10, tileY * 20 + 10);
    Vector2 pushDir = (chunk.pos - tileCenter).normalized;

    // Find overlap depth
    float closestDist = DistanceToTile(chunk.pos, tileX, tileY);
    float overlap = chunk.rad - closestDist;

    // Push out
    chunk.pos += pushDir * overlap;
}
```

**Result**: Chunk positioned just outside the tile.

### Slopes and Surfaces

**Slopes** (45° angled tiles):
- Special tile types: `Slope_NorthEast`, `Slope_NorthWest`, etc.
- Push direction follows slope angle
- Creatures can climb slopes naturally

**Surfaces** (poles, floors):
- Horizontal surfaces: Push up vertically
- Vertical surfaces: Push horizontally
- Special handling for creature grabs/climbs

## Object-Object Collision

### The Three-Layer System

BodyChunks are divided into **three collision layers**:

| Layer | Contains | Purpose |
|-------|----------|---------|
| **0** | Most creatures, player | Main creature interactions |
| **1** | Small items, some small creatures | Item physics |
| **2** | Special cases (thrown spears, etc.) | Projectile physics |

**Key rule**: Chunks only collide with others in the **same layer**.

### Why Layers?

**Without layers** (all in one group):
- 50 chunks: 50 × 49 / 2 = **1,225 checks**

**With 3 layers** (distributed evenly):
- Layer 0: 17 chunks: 17 × 16 / 2 = **136 checks**
- Layer 1: 17 chunks: 17 × 16 / 2 = **136 checks**
- Layer 2: 16 chunks: 16 × 15 / 2 = **120 checks**
- **Total: 392 checks** (~3× faster)

### Circle-Circle Collision

The simplest collision algorithm:

```csharp
foreach (BodyChunk chunk1 in layer) {
    foreach (BodyChunk chunk2 in layer) {
        if (chunk1 == chunk2) continue;

        // Check overlap
        float distance = Vector2.Distance(chunk1.pos, chunk2.pos);
        float minDist = chunk1.rad + chunk2.rad;

        if (distance < minDist) {
            // Collision! Resolve
            ResolveCollision(chunk1, chunk2, distance, minDist);
        }
    }
}
```

**Optimization**: Distance check is fast (just `sqrt((x2-x1)² + (y2-y1)²)`).

### Collision Resolution

When two chunks overlap:

```csharp
void ResolveCollision(BodyChunk c1, BodyChunk c2,
                      float dist, float minDist) {
    // Calculate overlap
    float overlap = minDist - dist;

    // Push direction (from c1 to c2)
    Vector2 pushDir = (c2.pos - c1.pos).normalized;

    // Mass ratio (heavier objects push less)
    float totalMass = c1.mass + c2.mass;
    float mass1Ratio = c2.mass / totalMass;  // c1 pushes based on c2's mass
    float mass2Ratio = c1.mass / totalMass;  // c2 pushes based on c1's mass

    // Apply push (split by mass)
    c1.pos -= pushDir * overlap * mass1Ratio;
    c2.pos += pushDir * overlap * mass2Ratio;
}
```

**Result**: Heavier objects push lighter objects more. Equal masses push equally.

### Special Cases

**Static objects** (terrain):
- Mass = infinity
- Don't get pushed by collisions
- Only push other objects

**One-way collisions** (spears hitting creatures):
- Spear deals damage
- Spear bounces off
- Creature takes knockback

**Grabbing** (player grabbing creature):
- Disable normal collision
- Special grab physics
- Manual positioning

## Broad Phase Optimization

Rain World doesn't implement complex spatial partitioning (quadtrees, etc.) because:
1. Collision layers already provide good separation
2. Rooms are relatively small (~1-4 screens)
3. Simple nested loops are cache-friendly

**However**, there's implicit spatial partitioning:
- Only check chunks in **realized rooms**
- Chunks in abstract rooms: 0 collision cost
- Room-based separation prevents cross-room checks

### Room-Based Partitioning

```
Room A (realized):
- 25 chunks → collision checks

Room B (abstract):
- 0 chunks realized → 0 collision checks

Room C (realized):
- 20 chunks → collision checks

Rooms don't check against each other
```

**Result**: Collision cost scales with **chunks in realized rooms**, not total chunks in region.

## Collision Order

Each frame @ 40 TPS:

```
1. Update AI (decide movement)
   ↓
2. Apply forces (gravity, input, AI)
   ↓
3. Integrate positions (Verlet)
   ↓
4. Terrain collision (push out of walls)
   ← FIRST, because terrain is immovable
   ↓
5. Object-object collision (layer by layer)
   ← SECOND, after terrain resolved
   ↓
6. Constraint solving (springs, 5 iterations)
   ← LAST, to maintain creature shape
```

**Why this order?**
- Terrain collision first: Prevents chunks inside walls
- Object collision second: Creatures push each other
- Constraints last: Restore creature integrity

## Performance Analysis

### Cost Breakdown (50 chunks in room)

**Terrain collision**:
- 50 chunks × 9 tiles per chunk × 0.001ms = **0.45ms**

**Object collision** (split across 3 layers):
- 392 checks × 0.001ms = **0.39ms**

**Total collision cost**: **~0.84ms per frame**

At 40 TPS (25ms per frame): Collision uses **3.4% of frame budget**.

### Scaling

| Chunks | Terrain (ms) | Object (ms) | Total (ms) | % of 25ms |
|--------|--------------|-------------|------------|-----------|
| 25 | 0.23 | 0.10 | 0.33 | 1.3% |
| 50 | 0.45 | 0.39 | 0.84 | 3.4% |
| 75 | 0.68 | 0.88 | 1.56 | 6.2% |
| 100 | 0.90 | 1.57 | 2.47 | 9.9% |

**Takeaway**: Collision scales linearly with terrain, quadratically with objects (but mitigated by layers).

## Edge Cases and Special Handling

### 1. High-Speed Collisions

**Problem**: Fast-moving chunk might tunnel through thin walls.

**Solution**: Not solved perfectly, but mitigated:
- BodyChunks have reasonable max speed
- Multiple constraint iterations help
- Most walls are thick enough

**Tradeoff**: Occasional tunneling accepted for performance.

### 2. Stacking and Compression

**Problem**: Many chunks in small space compress infinitely.

**Solution**: Iterative constraint solving (5 passes) converges to stable state.

**Result**: Stable piles of creatures/items.

### 3. Creature Inside Creature

**Problem**: If creature spawns inside another, they're stuck.

**Solution**:
- Spawn creatures at safe locations (dens, room edges)
- Initial spawn checks for overlaps
- Extreme overlap: forcefully separate

### 4. Water and Special Materials

**Water**:
- Not handled by collision system
- Separate water physics (buoyancy, drag)
- Applies forces to chunks in water

**Slopes and Poles**:
- Special terrain types
- Modify push direction
- Enable climbing mechanics

## Comparison to Other Approaches

### Polygon Collision (Box2D, Unity Physics)

**Algorithm**: SAT (Separating Axis Theorem), GJK
**Cost**: O(edges²) per pair
**Pros**: Accurate for complex shapes
**Cons**: Expensive, complex, harder to make deterministic

### Rain World (Circle Collision)

**Algorithm**: Distance check
**Cost**: O(1) per pair
**Pros**: Fast, simple, deterministic
**Cons**: Less accurate for non-circular shapes

**Verdict**: For Rain World's soft-body creatures, circles are perfect.

## Design Tradeoffs

| Decision | Benefit | Cost |
|----------|---------|------|
| **Circle-only collision** | O(1) per pair, ~10× faster than polygons | Can't represent complex rigid shapes |
| **Three collision layers** | ~3× fewer checks | Must categorize objects carefully |
| **Tile-based terrain** | Fast lookup, simple collision | Fixed grid size, visible in level design |
| **No spatial partitioning** | Simple, cache-friendly | Scales O(n²) without layers |
| **Iterative resolution** | Stable under compression | Some jitter possible |
| **No tunneling prevention** | Simpler, faster | Occasional tunneling at high speed |

## Related Documentation

**Same layer**:
- [BodyChunk Physics](bodychunk-physics.md) - Physics system overview
- [Terrain Collision](terrain-collision.md) - Tile collision deep dive
- [Physics Constraints](physics-constraints.md) - Spring connections

**Go deeper**:
- [Layer 2: Collision Algorithms](../../02-implementation/algorithms/collision-algorithms.md)
- [Layer 3: Collision Complexity](../../03-technical/algorithms-deep-dive/collision-detection-complexity.md)

**Related topics**:
- [Update Loop](../core-architecture/update-loop.md) - When collision runs
- [Design Decisions: Why BodyChunks?](../design-decisions/why-bodychunks.md)

## Key Takeaways

1. **Two collision types**: Terrain (chunk vs tiles) and object (chunk vs chunk)
2. **Three collision layers**: ~3× speedup by avoiding unnecessary checks
3. **Circle-circle collision**: O(1) per pair, extremely fast
4. **Tile-based terrain**: Only check nearby tiles (3×3 grid)
5. **Mass-based resolution**: Heavier objects push lighter ones more
6. **Collision order**: Terrain → objects → constraints
7. **Performance**: ~0.84ms for 50 chunks (3.4% of frame budget)
8. **Scaling**: Linear with terrain, quadratic with objects (mitigated by layers)
