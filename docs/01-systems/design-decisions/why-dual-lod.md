# Why Dual LOD?

**Layer**: 1 (Systems Architecture)
**Reading time**: 12 minutes

## The Question

Why does Rain World use two levels of detail (Abstract and Realized) for entities, when most games just have one?

## The Problem

Rain World's design goals create a technical challenge:

**Goal**: Simulate a living ecosystem
- 70+ interconnected rooms per region
- 200-400 creatures that hunt, migrate, and interact
- Creatures persist when off-screen ("the world continues")
- Player explores smoothly without loading screens

**Hardware**: PS4, Switch, lower-end PCs
- Limited RAM: ~3-4GB for game
- Limited CPU: ~40% of power for AI/physics
- Must maintain 40 FPS minimum

**The math doesn't work**:
- 300 creatures × full AI/physics each = **performance collapse**

## Alternatives Considered

### Option 1: Simulate Everything Always

**Implementation**:
```
Every creature:
- Full AI every frame
- Full physics (BodyChunks, collision)
- Graphics rendering
- 40 TPS update rate
```

**Cost per creature**: ~0.7ms per frame
**300 creatures**: 210ms per frame
**Frame budget**: 25ms @ 40 FPS

**Result**: ❌ **13× over budget. Impossible.**

---

### Option 2: Only Simulate Visible

**Implementation**:
```
Visible creatures (5-10):
- Full AI, physics, graphics

Off-screen creatures:
- Don't exist at all
- Spawn when player enters room
```

**Cost**: ~5ms per frame (only visible creatures)

**Result**: ✅ **Performance: Works**
**But**: ❌ **World feels fake**

**Problems**:
- Creatures spawn just for you (feels scripted)
- Return to room: creatures are gone
- No persistence, no ecosystem
- Immersion broken

This is how many games work, but doesn't fit Rain World's vision.

---

### Option 3: Traditional Graphics LOD Only

**Implementation**:
```
Near player (realized):
- Full AI, physics, high-poly graphics

Far from player (LOD 1-3):
- Full AI, physics, low-poly graphics
```

**Cost**: AI+physics still full for all creatures
- 300 creatures × 0.6ms (AI+physics only) = 180ms
- Still 7× over budget

**Result**: ❌ **Graphics LOD doesn't solve the problem.**

Traditional LOD reduces rendering cost, not simulation cost. Rain World's bottleneck is AI and physics, not graphics.

---

### Option 4: Dual LOD (AI and Physics)

**Implementation**:
```
Far from player (Abstract):
- Simplified AI: probability models, room-level position
- No physics: just room location
- No graphics
- Update every 3-5 frames
- Cost: ~0.01ms per update

Near player (Realized):
- Full AI: pathfinding, behavior trees
- Full physics: BodyChunks, collision
- Graphics: sprites, animations
- Update every frame
- Cost: ~0.7ms per frame
```

**Cost calculation**:
- 285 abstract creatures × 0.01ms / 4 frames = **0.71ms per frame**
- 15 realized creatures × 0.7ms = **10.5ms per frame**
- **Total: 11.2ms per frame** ✅

**Frame budget**: 25ms @ 40 FPS

**Result**: ✅ **Fits comfortably. 45% of budget.**

---

## Why Dual LOD Wins

### Performance Ratio

**Abstract creature**: ~0.01ms per update (every ~4 frames)
**Realized creature**: ~0.7ms per frame

**Ratio**: Realized is **~70× more expensive**

**With 95% abstract, 5% realized**:
- Can simulate 300 creatures total
- Only 15 are expensive at any time
- Ecosystem scale achieved

### What Abstract Mode Does

**Strategic simulation**:
- Migration: "Should I move to adjacent room?"
- Hunger: Increases over time, drives food-seeking
- Simplified combat: Dice rolls instead of physics
- Relationships: Track threats, prey, pack bonds

**Not doing**:
- Pathfinding within room (just room-to-room)
- Physics simulation (no BodyChunks)
- Detailed decision-making (just high-level)
- Rendering (no graphics at all)

**Result**: Just enough simulation to feel alive, not enough to be expensive.

### What Realized Mode Does

**Tactical simulation**:
- Pathfinding: A* on tile grid within room
- Behavior: State machines (chase, flee, patrol)
- Physics: BodyChunks, collision, constraints
- Combat: Actual physics interactions
- Graphics: Sprites, animations, effects

**Result**: Full detail for gameplay, only for nearby creatures.

### The Transition

**Seamless from player perspective**:
1. Player approaches room with abstract creatures
2. Room realizes (loads geometry, spawns creatures)
3. Abstract creatures realize (create PhysicalObjects)
4. Creatures appear naturally (at dens, room edges)
5. Player enters, never noticed the transition

**Data transfer**:
- Position: Room + node → Spawn location
- State: Health, hunger, relationships
- Not transferred: Exact coordinates, momentum, current behavior

**Cost**: Data loss, but player can't observe it (they weren't watching).

## Real-World Impact

### Performance Metrics

**Typical room** (15 creatures realized, 285 abstract across region):
- Abstract AI: 0.71ms (6.3% of 11.2ms simulation budget)
- Realized AI: 10.5ms (93.7% of 11.2ms simulation budget)
- Physics: 2ms
- Graphics: 3ms
- **Total frame**: ~17ms (68% of 25ms budget)

**Without dual LOD** (all realized):
- 300 creatures × 0.7ms = 210ms
- **Frame budget exceeded by 840%**

### Ecosystem Behavior

**Emergent narratives**:
- Lizard hunts scavenger in Room A (abstract)
- Scavenger flees to Room B
- Player enters Room B
- Finds wounded scavenger (realizes)
- Story told through state persistence

**Persistence**:
- Flee from lizard in Room A
- Explore Rooms B, C, D (10 minutes)
- Return to Room A
- Lizard still there (or migrated hunting food)
- World feels alive, not scripted

### Player Experience

**What player notices**:
- Creatures exist in rooms they haven't visited
- Return to room: creatures are where expected
- Off-screen interactions have consequences
- World feels alive beyond view

**What player doesn't notice**:
- Creatures switching between AI modes
- Data loss on transitions
- Simplified simulation when far away
- 70× performance difference

**Success**: Illusion is complete. World feels real.

## Why Other Games Don't Do This

### Requirement: Persistent World

Most games **don't need** creatures to persist off-screen:
- **Linear games**: Player never returns
- **Arena games**: One small area
- **Scripted games**: Creatures spawn for encounters

Rain World **requires** persistence:
- Metroidvania structure (return to areas)
- Ecosystem simulation (interactions continue)
- Survival gameplay (learn creature patterns)

### Development Cost

Implementing dual LOD is **expensive**:
- Two AI systems to build (abstract + realized)
- Transition logic (realize/abstractize)
- Data persistence carefully designed
- Testing both modes
- **Estimated**: 6-12 months of development

**Worth it if**:
- Ecosystem simulation is core to game
- Large persistent world
- Limited hardware (can't afford full simulation)

**Not worth it if**:
- Small levels
- Few AI entities
- Linear progression
- Ample performance budget

### Alternatives for Other Games

**For large worlds without ecosystem**:
- Spawn creatures when needed (cheaper)
- Respawn on room re-entry (simpler)
- Traditional graphics LOD (if rendering is bottleneck)

**For ecosystem simulation with more performance**:
- Full simulation for all (if you can afford it)
- Fewer creatures (reduce scope)
- Simpler AI (make full simulation cheaper)

## Design Tradeoffs

| Aspect | Gained | Lost |
|--------|--------|------|
| **Performance** | 70× cheaper for distant creatures | Must maintain two systems |
| **Scale** | 300 creatures across 70 rooms | Data loss on transitions |
| **Persistence** | Creatures continue existing off-screen | Exact state not preserved |
| **Immersion** | World feels alive beyond view | Occasional inconsistencies |
| **Ecosystem** | Creatures hunt, migrate, interact | Simplified when abstract |

## When to Use Dual LOD

**Use dual LOD if**:
1. ✅ Large persistent world
2. ✅ Many AI entities (100+)
3. ✅ Ecosystem simulation important
4. ✅ Limited performance budget
5. ✅ Player returns to areas

**Don't use dual LOD if**:
1. ❌ Small contained levels
2. ❌ Few AI entities (<20)
3. ❌ Linear progression (no backtracking)
4. ❌ Ample performance budget
5. ❌ Entities can be spawned on demand

## Related Documentation

**Same layer**:
- [Dual LOD System](../core-architecture/dual-lod-system.md) - How it works
- [Creature Intelligence](../creature-systems/creature-intelligence.md) - AI implementation
- [Tradeoffs](tradeoffs.md) - All architectural decisions

**Go deeper**:
- [Layer 2: Abstract AI](../../02-implementation/algorithms/abstract-room-pathfinding.md)
- [Layer 2: Realized AI](../../02-implementation/algorithms/realized-tile-pathfinding.md)
- [Layer 3: LOD Performance](../../03-technical/performance-optimization/lod-performance.md)

**Related topics**:
- [Entity Lifecycle](../entity-systems/entity-lifecycle.md)
- [Update Loop](../core-architecture/update-loop.md)

## Key Takeaways

1. **Traditional LOD doesn't solve AI/physics cost**: Only reduces rendering
2. **Abstract mode is 70× cheaper**: Strategic simulation instead of tactical
3. **95/5 split**: 95% abstract (cheap), 5% realized (expensive) = scalable
4. **Data loss is acceptable**: Player can't observe it (they're far away)
5. **Enables ecosystem scale**: 300 creatures across 70 rooms
6. **Seamless transitions**: Player never notices mode switches
7. **Development cost high**: 6-12 months to implement, worth it for Rain World's goals
8. **Not for every game**: Only needed for large persistent worlds with many AI entities
