# Architectural Tradeoffs

**Layer**: 1 (Systems Architecture)
**Reading time**: 20 minutes

## Purpose

This document catalogs every major architectural decision in Rain World, explaining what was gained, what was sacrificed, and why the tradeoff was worthwhile for Rain World's specific goals.

## The Core Goal

Rain World's technical requirements:
1. **Large persistent world**: 70+ interconnected rooms per region
2. **Living ecosystem**: 200-400 creatures that exist and interact
3. **Limited hardware**: Must run on PS4, Switch, lower-end PCs
4. **Smooth gameplay**: Responsive platforming, 60 FPS visuals
5. **Fair difficulty**: Deterministic simulation for brutal-but-fair challenge

Every architectural decision serves these goals.

## Tradeoff Summary Table

| Decision | Gained | Lost | Verdict |
|----------|--------|------|---------|
| **Dual LOD System** | Simulate entire ecosystem, 70× cheaper abstract AI | Data loss on transitions, complexity | ✅ Essential for ecosystem |
| **Custom BodyChunk Physics** | Performance, squishiness, determinism | Realism, complexity of implementing everything | ✅ Perfect for creatures |
| **Circle-Only Collision** | 10× faster than polygons, simple | Can't represent complex rigid shapes well | ✅ Worth it for soft bodies |
| **Three Collision Layers** | 3× fewer collision checks | Must categorize objects, occasional bugs | ✅ Essential for performance |
| **Room-Based Streaming** | Predictable memory, clean boundaries | Hard room boundaries, no cross-room vision | ✅ Necessary for scale |
| **Graphics/Logic Separation** | 20MB saved per room, moddability | Synchronization complexity | ✅ Significant benefits |
| **40 TPS Physics, 60 FPS Graphics** | Determinism + smooth visuals | Interpolation required | ✅ Best of both worlds |
| **RainWorld Singleton** | Convenient global access | Tight coupling, hard to test | ⚠️ Convenient but problematic |
| **Deep Inheritance** | Natural OOP, code reuse | Rigid categorization, memory waste | ⚠️ Works but has issues |
| **No Unity Physics** | Control, performance, determinism | Must implement everything yourself | ✅ Worth full control |

## Detailed Analysis

### 1. Dual LOD System

**Decision**: Entities exist in two forms: Abstract (lightweight) and Realized (full detail).

#### What Was Gained

**Ecosystem simulation at scale**:
- 200-400 creatures across entire region
- Abstract AI: ~0.03ms per creature per update
- Can afford to simulate hundreds simultaneously

**Persistent world**:
- Creatures continue existing off-screen
- Return to a room: creatures are where they should be
- Emergent narratives from off-screen interactions

**Performance**:
- Abstract creatures: ~70× cheaper than realized
- Most creatures are abstract most of the time (95%)
- Only pay full cost for ~5% of population

#### What Was Lost

**State continuity**:
- Exact position lost on abstraction
- Current behavior state (mid-chase, mid-flee) forgotten
- Momentum doesn't carry over
- Short-term memory erased

**Complexity**:
- Two AI systems to maintain
- Synchronization between abstract/realized
- Transition logic can have bugs
- Data persistence carefully managed

**Edge cases**:
- Wolf spider revive mechanic lost when abstracted
- Creature "forgets" player if abstracted mid-chase
- Some behaviors don't translate well to abstract

#### Why It's Worth It

**Without dual LOD**:
- Option A: Simulate everything fully → Performance collapse
- Option B: Only simulate visible → World feels fake
- Option C: Traditional graphics LOD only → Doesn't solve AI/physics cost

**With dual LOD**:
- Ecosystem feels alive across entire region
- Performance enables 70+ rooms with population
- Player rarely notices data loss (they're not watching)

**Verdict**: ✅ **Essential**. The defining architectural innovation.

---

### 2. Custom BodyChunk Physics

**Decision**: Implement custom circle-based soft-body physics instead of using Unity Physics or Box2D.

#### What Was Gained

**Performance**:
- Circle-circle collision: O(1), extremely fast
- 50 BodyChunks: ~2ms per frame
- Unity Physics equivalent: ~5-8ms

**Soft-body behavior**:
- Creatures naturally squish through tight spaces
- Organic-feeling ragdoll without special systems
- Natural deformation on impact

**Determinism**:
- Full control over implementation
- No hidden randomness
- Perfectly reproducible results
- Critical for fair difficulty

**Simplicity**:
- Easy to debug (circles are intuitive)
- Easy to visualize (draw circles in editor)
- Easy to mod (simple data structures)

#### What Was Lost

**Realism**:
- Less accurate than polygon physics
- Can't perfectly represent rigid boxes
- Some situations feel "blobby"

**Features**:
- No friction model
- No proper rigid body dynamics
- No advanced constraints (motors, prismatic joints)
- Must implement everything yourself

**Development time**:
- Months of work to implement physics system
- Must maintain custom physics code
- Can't leverage existing physics engines

#### Why It's Worth It

**Unity Physics alternative**:
- Realistic but rigid
- Expensive for many creatures
- Hard to make deterministic
- Overkill for platformer

**BodyChunk system**:
- Perfect for soft-body creatures
- Performance enables ecosystem scale
- Deterministic for fair challenge
- Squishiness enhances feel

**Verdict**: ✅ **Perfect fit**. Worth the development cost.

---

### 3. Circle-Only Collision

**Decision**: All collision is circle-based. No polygons, no complex shapes.

#### What Was Gained

**Speed**:
- Circle-circle: Just distance check
- ~10× faster than polygon-polygon
- Enables dozens of creatures simultaneously

**Simplicity**:
- Collision code is ~500 lines
- Easy to understand and debug
- No edge cases from complex shapes

**Cache-friendly**:
- BodyChunk data is compact (24 bytes)
- Array of chunks fits in L2 cache
- Fast iteration

#### What Was Lost

**Shape accuracy**:
- Can't represent thin spears realistically
- Can't represent flat surfaces well
- Everything is "round"

**Some gameplay**:
- Can't have perfectly flat platforms
- Can't have thin barriers
- Some unrealistic squeezing

#### Why It's Worth It

**Polygon alternative**:
- More accurate shapes
- 10× slower collision
- Can't afford ecosystem scale

**Circle system**:
- Fast enough for ecosystem
- "Round" feel enhances creature organicness
- Spears use 1 circle: good enough

**Verdict**: ✅ **Excellent tradeoff**. Speed enables scale.

---

### 4. Three Collision Layers

**Decision**: Divide BodyChunks into 3 collision layers. Chunks only collide within same layer.

#### What Was Gained

**Performance**:
- ~3× fewer collision checks
- 50 chunks: 392 checks instead of 1,225
- Critical for hitting performance target

**Separation of concerns**:
- Creatures don't collide with every item
- Items don't collide with each other unnecessarily
- Cleaner physics behavior

#### What Was Lost

**Flexibility**:
- Must categorize every object into a layer
- Can't easily change layer at runtime
- Some objects don't fit cleanly

**Bugs from miscategorization**:
- Put object in wrong layer → doesn't collide
- Hard to debug ("why isn't this colliding?")
- Requires careful design

**Cross-layer interactions**:
- Items on different layers don't interact
- Can lead to weird situations
- Must use special-case code

#### Why It's Worth It

**Without layers**:
- 1,225 checks for 50 chunks
- Can't hit performance target
- Must reduce creature count

**With layers**:
- 392 checks for 50 chunks
- Hits performance target
- Ecosystem scale achievable

**Verdict**: ✅ **Essential optimization**. 3× speedup critical.

---

### 5. Room-Based Streaming

**Decision**: Divide world into discrete rooms. Stream rooms in/out as player explores.

#### What Was Gained

**Predictable memory**:
- 1-3 rooms loaded at once
- Memory usage is constant
- No spikes, no surprises

**Clean loading**:
- RoomPreparer loads in background
- Seamless transitions (no pop-in)
- Progressive loading while player approaches

**Authoring simplicity**:
- Rooms are manageable chunks (1-4 screens)
- Easy to test individual rooms
- Clear boundaries for level design

#### What Was Lost

**Hard boundaries**:
- Creatures can't see through doors
- Projectiles stop at room exits
- Sound doesn't travel between rooms

**Visible seams**:
- Room transitions are noticeable
- Can't have truly seamless world
- Level design constrained by rooms

**No long-range interactions**:
- Can't shoot from one room to another
- Can't see into adjacent room
- Chase sequences break at boundaries

#### Why It's Worth It

**Open world alternative**:
- Seamless, no boundaries
- Extremely complex streaming
- Memory management nightmare
- Can't afford on target hardware

**Room-based system**:
- Simple, predictable
- Fits in memory on all platforms
- Clean transitions
- Enables 70+ room regions

**Verdict**: ✅ **Necessary compromise**. Scale requires boundaries.

---

### 6. Graphics/Logic Separation

**Decision**: PhysicalObject (logic) and GraphicsModule (rendering) are separate objects.

#### What Was Gained

**Memory savings**:
- ~20MB per room (10 off-camera creatures)
- ~100MB per region (5 realized rooms)
- Critical for console memory limits

**CPU savings**:
- ~0.4ms per frame per room
- Off-camera creatures: 0ms graphics cost
- More CPU for ecosystem simulation

**Moddability**:
- Replace graphics without touching logic
- Graphics mods very popular
- Logic mods don't break graphics

**Frame rate independence**:
- Physics: 40 TPS (deterministic)
- Graphics: 60 FPS (smooth)
- Interpolation connects them

#### What Was Lost

**Synchronization complexity**:
- Graphics must read from logic every frame
- Out-of-sync bugs possible
- Must maintain consistency

**Boilerplate**:
- InitiateGraphicsModule() on every PhysicalObject
- DisposeGraphicsModule() cleanup
- More classes to maintain

**Not intuitive**:
- Separate objects for one entity
- New developers confused
- Debugging: which object has the bug?

#### Why It's Worth It

**Monolithic alternative**:
- Simple: one object does both
- Wasted memory on off-camera entities
- Wasted CPU on off-camera updates
- Can't afford ecosystem scale

**Separated system**:
- Memory enables more realized rooms
- CPU enables more creatures
- Moddability is huge win
- Interpolation enables smooth visuals

**Verdict**: ✅ **Significant benefits**. Complexity worth it.

---

### 7. Dual Update Rates (40 TPS / 60 FPS)

**Decision**: Physics/AI at 40 TPS, graphics at 60 FPS with interpolation.

#### What Was Gained

**Determinism**:
- Fixed timestep: same inputs → same outputs
- Critical for fair challenge
- Reproducible for speedrunners

**Performance**:
- 40 TPS: 33% less physics work than 60
- Saves CPU for ecosystem
- Enables more creatures

**Smooth visuals**:
- 60 FPS graphics via interpolation
- Smooth character movement
- Responsive camera

#### What Was Lost

**Interpolation complexity**:
- Graphics must interpolate between frames
- timeStacker parameter everywhere
- Potential visual artifacts

**Input latency**:
- Input processed at 60 FPS (RawUpdate)
- But physics at 40 TPS
- ~8ms latency max

**Not industry standard**:
- Most games: same rate for all
- Unusual architecture
- New developers confused

#### Why It's Worth It

**60 TPS alternative**:
- More responsive
- 50% more CPU for physics
- Can't afford ecosystem scale

**40 TPS alternative (no interpolation)**:
- Save CPU
- Choppy visuals (not 60 FPS)
- Feels bad for platformer

**40 TPS + 60 FPS interpolation**:
- Deterministic physics
- Smooth visuals
- Best of both worlds

**Verdict**: ✅ **Clever solution**. Optimal for requirements.

---

### 8. RainWorld Singleton

**Decision**: RainWorld class is globally accessible singleton.

#### What Was Gained

**Convenience**:
- Access from anywhere: `game.rainWorld.options`
- No dependency passing
- Rapid development

**Unity integration**:
- Natural fit for MonoBehaviour pattern
- Single persistent GameObject
- Simple to understand initially

#### What Was Lost

**Tight coupling**:
- Every class coupled to RainWorld
- Hard to change RainWorld API
- Modifications ripple everywhere

**Testing difficulty**:
- Must initialize entire engine to test anything
- Unit tests nearly impossible
- Integration tests only

**Mod conflicts**:
- Mods mutate shared global state
- Race conditions
- Compatibility nightmares

#### Why It's a Problem

**Dependency Injection alternative**:
- Explicit dependencies
- Testable
- More initial work
- Cleaner architecture long-term

**Singleton pattern**:
- Convenient short-term
- Technical debt long-term
- Works for small projects
- Problematic at scale

**Verdict**: ⚠️ **Convenient but problematic**. See [Singleton Coupling Problem](../../appendix/problems/singleton-coupling.md).

---

### 9. Deep Inheritance Hierarchies

**Decision**: Creatures organized in deep inheritance trees.

#### What Was Gained

**Natural OOP**:
- "Lizard IS-A Creature"
- Intuitive structure
- Matches mental model

**Code reuse**:
- Base Creature class: shared physics, rendering
- Specialized subclasses: unique behaviors
- Don't repeat yourself (DRY)

#### What Was Lost

**Rigidity**:
- Can't make "creature that's half lizard, half scavenger"
- Must fit into single inheritance chain
- Hard to add cross-cutting features

**Memory waste**:
- All lizards carry fields for all lizard variants
- ~60% waste estimated
- Cache pollution

**Modification difficulty**:
- Change base class → affects all subclasses
- Hard to add features
- Brittle

#### Why It's Problematic

**Component-based alternative (ECS)**:
- Flexible composition
- Memory efficient
- More complex initially
- Better long-term

**Deep inheritance**:
- Works for initial development
- Becomes rigid over time
- Hard to refactor later
- Technical debt accumulates

**Verdict**: ⚠️ **Works but has issues**. See [Inheritance Rigidity Problem](../../appendix/problems/inheritance-rigidity.md).

---

### 10. No Unity Physics

**Decision**: Don't use Unity's built-in physics engine.

#### What Was Gained

**Full control**:
- Implement exactly what's needed
- No hidden behavior
- No black box

**Performance**:
- Custom system: ~2ms for 50 chunks
- Unity Physics: ~5-8ms equivalent
- 2-3× faster

**Determinism**:
- Guaranteed reproducible
- No hidden randomness
- Perfect for fair difficulty

**Soft-body focus**:
- Built specifically for squishy creatures
- Unity Physics: designed for rigid bodies
- Better fit for requirements

#### What Was Lost

**Development time**:
- Months to implement physics system
- Must maintain custom code
- Can't leverage Unity updates

**Features**:
- No advanced constraints
- No friction models
- No articulated physics
- Must implement everything

**Ecosystem tools**:
- Can't use Unity physics debugger
- Can't use third-party physics tools
- Custom debugging required

#### Why It's Worth It

**Unity Physics alternative**:
- Free, maintained by Unity
- Feature-rich
- But: rigid body focused, hard to make deterministic, expensive

**Custom BodyChunks**:
- Perfect for Rain World's creatures
- Deterministic by design
- Performance enables scale
- Soft-body behavior built-in

**Verdict**: ✅ **Worth full control**. Enables core gameplay.

---

## Tradeoff Themes

### Theme 1: Performance vs Features

Multiple decisions sacrifice features for performance:
- Circles instead of polygons
- Three layers instead of all-collide-all
- 40 TPS instead of 60 TPS

**Why**: Target hardware (PS4, Switch) can't afford full-featured systems.

### Theme 2: Scale vs Detail

Dual LOD is the ultimate scale/detail tradeoff:
- Can't afford to simulate everything in full detail
- Can afford to simulate everything in simple detail
- Player gets both: detail nearby, scale overall

**Why**: Ecosystem simulation requires hundreds of entities.

### Theme 3: Convenience vs Architecture

Singleton and inheritance prioritize rapid development:
- Convenient initially
- Technical debt later
- Hard to refactor

**Why**: Small initial team, tight deadlines.

### Theme 4: Control vs Leverage

Custom physics instead of Unity Physics:
- More initial work
- But: perfect fit for needs
- Determinism essential

**Why**: Specific requirements (soft bodies, determinism) not met by generic solution.

## Lessons for Other Games

### When to Use These Patterns

**Dual LOD**: Games with large worlds, many AI entities, limited hardware
**BodyChunk physics**: Soft-body focused, determinism important, performance critical
**Graphics separation**: Many entities, memory constrained, moddability valued
**Room-based streaming**: Large worlds, memory limited, clear boundaries acceptable

### When to Avoid

**Dual LOD**: Small worlds, few entities, ample performance
**BodyChunk physics**: Need realistic rigid bodies, have CPU to spare
**Graphics separation**: Few entities, not memory constrained
**Room-based streaming**: Seamless world essential, open-world game

## Related Documentation

**Same layer**:
- [Why Dual LOD?](why-dual-lod.md) - Deep dive on LOD decision
- [Why BodyChunks?](why-bodychunks.md) - Deep dive on physics decision
- [Why Room-Based?](why-room-based.md) - Deep dive on streaming decision

**Problems**:
- [Singleton Coupling](../../appendix/problems/singleton-coupling.md)
- [Inheritance Rigidity](../../appendix/problems/inheritance-rigidity.md)

**Improvements**:
- [ECS Migration](../../appendix/improvements/ecs-migration.md)
- [Dependency Injection](../../appendix/improvements/dependency-injection.md)

## Key Takeaways

1. **Every decision trades something**: No perfect choices, only tradeoffs
2. **Context matters**: Good decisions for Rain World might be bad for other games
3. **Scale drives architecture**: Ecosystem simulation shaped every major decision
4. **Performance is paramount**: Target hardware limitations influenced everything
5. **Determinism is critical**: Fair difficulty requires reproducible simulation
6. **Some tradeoffs age better**: Dual LOD ages well, singleton ages poorly
7. **Convenience vs correctness**: Early convenience can become later technical debt
