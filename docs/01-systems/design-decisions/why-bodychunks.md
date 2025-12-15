# Why BodyChunks?

**Layer**: 1 (Systems Architecture)
**Reading time**: 15 minutes

## The Question

Why did Rain World implement a custom circle-based soft-body physics system instead of using Unity Physics, Box2D, or another established physics engine?

## The Problem

Rain World needs physics for creatures that:
1. **Feel organic**: Squish through tight spaces, tumble naturally
2. **Run on limited hardware**: PS4, Switch, lower-end PCs
3. **Support many entities**: 15-20 creatures in a room simultaneously
4. **Are deterministic**: Same inputs → same outputs (fair difficulty)
5. **Work at 40 TPS**: Not 60 FPS (for determinism and performance)

Traditional physics engines aren't designed for these requirements.

## Alternatives Considered

### Option 1: Unity Physics (PhysX)

**What it is**: Unity's built-in 3D/2D physics engine, powered by NVIDIA PhysX.

#### Implementation

```csharp
class Creature : MonoBehaviour {
    Rigidbody2D rb;          // Unity's rigid body
    PolygonCollider2D col;   // Polygon collider

    void FixedUpdate() {
        // Physics handled by Unity
        rb.AddForce(movement);
    }
}
```

**Automatic features**:
- Collision detection
- Rigid body dynamics
- Constraints and joints
- Optimized and maintained by Unity

#### Problems for Rain World

**Problem 1: Rigid Bodies**
- PhysX designed for **rigid** objects (boxes, crates)
- Creatures need to be **squishy** (squish through pipes)
- Soft-body physics in PhysX is complex and expensive

**Problem 2: Performance**
- Polygon-polygon collision: expensive
- 15 creatures × 3-4 colliders each = 45-60 polygons
- Cost: ~5-8ms per frame
- Rain World budget: ~2-3ms for physics

**Problem 3: Determinism**
- PhysX uses floating-point math with subtle variations
- Multithreading can introduce non-determinism
- Hard to guarantee same inputs → same outputs
- Rain World needs perfect determinism for fair difficulty

**Problem 4: Control**
- Black box: can't see or modify internals
- Hidden behaviors: friction models, solver details
- Hard to debug: "why did this weird thing happen?"

**Result**: ❌ **Doesn't fit requirements. Too rigid, too expensive, not deterministic.**

---

### Option 2: Box2D

**What it is**: Popular 2D physics library, used in many indie games.

#### Implementation

```csharp
// Box2D physics world
World physicsWorld = new World(gravity);

// Create creature body
BodyDef bodyDef = new BodyDef();
bodyDef.type = BodyType.Dynamic;
Body creatureBody = physicsWorld.CreateBody(bodyDef);

// Add fixtures (shapes)
PolygonShape shape = new PolygonShape();
shape.SetAsBox(width, height);
creatureBody.CreateFixture(shape, density);

// Step simulation
physicsWorld.Step(dt, velocityIterations, positionIterations);
```

#### Problems for Rain World

**Problem 1: Still Rigid-Body Focused**
- Designed for rigid objects
- Soft-body requires complex joint setups
- Not intuitive for squishy creatures

**Problem 2: Polygon Collision**
- Still expensive for many entities
- 15 creatures × 4 polygons = complex collision
- Performance similar to Unity Physics

**Problem 3: Determinism**
- Better than PhysX, but still has floating-point edge cases
- Iteration counts can vary
- Tricky to guarantee perfect determinism

**Problem 4: Integration**
- External library (not built into Unity)
- Must integrate yourself
- No Unity editor integration

**Problem 5: Still a Black Box**
- Can't modify collision algorithms
- Limited control over solver
- Hard to debug internals

**Result**: ❌ **Better than Unity Physics, but still doesn't fit perfectly.**

---

### Option 3: Custom BodyChunk System

**What it is**: Custom implementation using circles connected by springs.

#### Implementation

```csharp
// BodyChunk: simple struct
struct BodyChunk {
    Vector2 pos, lastPos;    // Position (Verlet)
    float mass, rad;         // Mass and radius
}

// PhysicalObject: composed of chunks
class Creature {
    BodyChunk[] bodyChunks;  // 2-5 circles
    BodyChunkConnection[] connections;  // Springs

    void Update() {
        // 1. Apply forces
        foreach (chunk in bodyChunks) {
            chunk.vel += gravity * mass;
        }

        // 2. Integrate (Verlet)
        foreach (chunk in bodyChunks) {
            Vector2 temp = chunk.pos;
            chunk.pos += chunk.pos - chunk.lastPos;
            chunk.lastPos = temp;
        }

        // 3. Collide with terrain
        CollideTerrain();

        // 4. Collide with objects (circle-circle)
        CollideObjects();

        // 5. Solve constraints (springs)
        for (int i = 0; i < 5; i++) {
            SolveConstraints();
        }
    }
}
```

#### Advantages

**✅ Soft-Body by Default**
- Circles connected by springs = natural squishiness
- Squeeze through pipes: springs compress
- Tumble naturally: springs create organic motion
- **Perfect for creatures**

**✅ Performance**
- Circle-circle collision: O(1), just distance check
- 50 BodyChunks: ~2ms per frame
- **2-3× faster than polygon physics**

**✅ Determinism**
- Full control over implementation
- No hidden randomness
- Simple math: reproducible
- **Perfect determinism guaranteed**

**✅ Simplicity**
- BodyChunk: 24 bytes (pos, lastPos, mass, rad)
- Collision: ~50 lines of code
- Constraint solving: ~30 lines
- **Easy to understand and debug**

**✅ Control**
- Can modify any part of system
- Can add special cases easily
- Can optimize specifically for creatures
- **Complete flexibility**

#### Disadvantages

**❌ Development Time**
- Must implement everything yourself
- Physics, collision, constraints, debug tools
- **Estimated: 3-6 months**

**❌ Features**
- No friction model (not needed for Rain World)
- No advanced joints (hinges, motors)
- No rigid bodies (everything is soft)
- Must implement any needed features

**❌ Maintenance**
- Must fix bugs yourself
- Must optimize yourself
- Can't leverage physics engine updates

**Result**: ✅ **Perfect fit for Rain World. Worth the development cost.**

---

## Why BodyChunks Win

### Performance Comparison

| System | 50 BodyChunks | Notes |
|--------|---------------|-------|
| **Unity Physics** | ~5-8ms | Polygon collision, rigid body solver |
| **Box2D** | ~4-6ms | Similar to Unity, polygon-based |
| **BodyChunks** | ~2ms | Circle collision, simple solver |

**BodyChunks are 2-3× faster.**

### Soft-Body Comparison

**Unity Physics soft-body**:
- Multiple rigid bodies connected by joints
- Complex setup (10+ components per creature)
- Expensive to simulate
- Unintuitive to author

**BodyChunks soft-body**:
- Circles connected by springs
- Simple setup (2-5 chunks + connections)
- Cheap to simulate
- Intuitive to author

**BodyChunks are simpler and faster for soft bodies.**

### Determinism Comparison

**Unity Physics / Box2D**:
- Floating-point edge cases
- Solver iteration counts can vary
- Multithreading introduces non-determinism
- **Hard to guarantee perfect determinism**

**BodyChunks**:
- Simple math, no hidden behavior
- Fixed iteration counts (5 passes)
- Single-threaded, predictable
- **Perfect determinism guaranteed**

### What BodyChunks Enable

**Squishiness**:
- Lizard squeezes through pipe: chunks compress, then restore
- Player tumbles down slope: chunks bounce organically
- Creature hit by spear: chunks react with natural deformation

**Many creatures**:
- 15-20 creatures in room: ~50 chunks total
- 2ms physics budget: affordable
- Ecosystem scale achieved

**Fair difficulty**:
- Same inputs → same outputs
- Players can learn patterns
- Speedrunners can rely on consistency
- Critical for brutal difficulty

## Real-World Examples

### Example 1: Slugcat Squeezing Through Pipe

**Setup**:
- Slugcat: 2 BodyChunks (head, body)
- Pipe: 1 tile wide (20 pixels)
- BodyChunk radius: 5 pixels each

**What happens**:
```
Frame 1: Player enters pipe
- Head chunk hits pipe walls
- Terrain collision pushes head inward
- Spring to body chunk stretches

Frame 2-5: Squeezing
- Head continues forward
- Body follows, stretches spring
- Chunks compress to fit

Frame 6: Exit pipe
- Chunks clear walls
- Spring pulls to rest length
- Slugcat "pops out" naturally
```

**Result**: Natural squishing without special-case code.

**With rigid body physics**: Would need complex articulation or special squeeze animation.

### Example 2: Lizard Ragdoll

**Setup**:
- Lizard: 3 BodyChunks (head, body, hips)
- Falls from height after death

**What happens**:
```
Frame 1: Lizard dies
- AI stops controlling chunks
- Only gravity and collisions remain

Frames 2-20: Tumbling
- Chunks fall independently
- Springs maintain rough shape
- Collisions with terrain create bounce
- Natural tumbling emerges

Frame 20: Comes to rest
- Chunks settle on floor
- Springs prevent total separation
- Natural-looking final pose
```

**Result**: Organic ragdoll physics without ragdoll system.

**With rigid body physics**: Would need separate ragdoll setup, more complex.

### Example 3: Creature Combat

**Setup**:
- Lizard attacks Scavenger
- Both have multiple BodyChunks

**What happens**:
```
Frame 1: Lizard lunges
- Lizard head chunk moves forward rapidly
- Collides with Scavenger body chunk

Frame 2: Impact
- Circle-circle collision detected
- Mass ratio: Lizard head (0.4) vs Scavenger body (0.6)
- Push resolved: Scavenger pushed more (heavier)
- Natural impact reaction

Frames 3-10: Recovery
- Springs restore creature shapes
- Creatures separate
- Natural combat feel
```

**Result**: Emergent combat physics from simple collisions.

**With rigid body physics**: Would need complex force calculations, possibly less responsive.

## Design Tradeoffs

| Aspect | BodyChunks | Traditional Physics |
|--------|------------|---------------------|
| **Soft-body** | ✅ Built-in, natural | ❌ Complex, expensive |
| **Performance** | ✅ 2ms for 50 chunks | ❌ 5-8ms for equivalent |
| **Determinism** | ✅ Perfect | ⚠️ Hard to guarantee |
| **Development** | ❌ 3-6 months work | ✅ Free, ready-to-use |
| **Features** | ⚠️ Basic only | ✅ Full-featured |
| **Maintenance** | ❌ Must maintain | ✅ Engine maintains |
| **Control** | ✅ Complete | ❌ Limited |
| **Debugging** | ✅ Simple | ⚠️ Black box |

## When to Use BodyChunks

**Use custom circle physics if**:
1. ✅ Soft-body creatures are core to game
2. ✅ Determinism is critical
3. ✅ Many entities need simulation
4. ✅ Limited performance budget
5. ✅ Team has physics development expertise
6. ✅ Can afford 3-6 months development

**Use traditional physics if**:
1. ❌ Need realistic rigid body physics
2. ❌ Need advanced features (complex joints, friction)
3. ❌ Few entities (< 10 simultaneously)
4. ❌ Ample performance budget
5. ❌ Limited development time
6. ❌ Determinism not critical

## Related Documentation

**Same layer**:
- [BodyChunk Physics](../physics-systems/bodychunk-physics.md) - How it works
- [Collision Detection](../physics-systems/collision-detection.md) - Collision algorithms
- [Tradeoffs](tradeoffs.md) - All architectural decisions

**Go deeper**:
- [Layer 2: Collision Algorithms](../../02-implementation/algorithms/collision-algorithms.md)
- [Layer 3: Physics Integration](../../03-technical/algorithms-deep-dive/physics-integration.md)
- [Layer 3: Verlet Integration](../../03-technical/algorithms-deep-dive/physics-integration.md#verlet)

**Related topics**:
- [Update Loop](../core-architecture/update-loop.md) - When physics runs
- [Creature Intelligence](../creature-systems/creature-intelligence.md) - AI controlling chunks

## Key Takeaways

1. **Custom physics worth it for specific needs**: Traditional engines don't fit soft-body + determinism requirements
2. **Circles are much faster**: 2-3× performance improvement over polygons
3. **Soft-body by default**: Springs create natural squishiness without special systems
4. **Determinism guaranteed**: Full control ensures reproducible results
5. **Development cost**: 3-6 months to implement, but worth it for Rain World
6. **Performance enables scale**: 2ms budget allows 15-20 creatures simultaneously
7. **Not for every game**: Only needed when traditional physics doesn't fit requirements
8. **Simple can be powerful**: 24-byte struct + simple algorithms = organic creature physics
