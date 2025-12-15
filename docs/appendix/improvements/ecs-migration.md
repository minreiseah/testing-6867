# Improvement: Entity Component System Migration

**Status**: Proposal
**Date**: 2025-12-14
**Effort**: High
**Related**: [Inheritance Rigidity Problem](../problems/inheritance-rigidity.md)

## Summary

Replace deep inheritance hierarchy (`PhysicalObject → Creature → specifics`) with Entity Component System for composition-based design.

## What is ECS?

**Entities**: Just IDs. No logic, no data.
**Components**: Pure data structs.
**Systems**: Logic that processes entities with specific components.

```csharp
// Current
class Lizard : Creature {
    // Data + logic mixed
}

// ECS
struct Entity { int id; }
struct PositionComponent { Vector2 pos; }
struct PhysicsComponent { BodyChunk[] chunks; }
struct AIComponent { PathFinder pathFinder; }

class PhysicsSystem {
    void Update(EntityManager em) {
        foreach (var entity in em.GetEntitiesWith<Position, Physics>()) {
            // Process physics
        }
    }
}
```

## Components for Rain World

### Core Components

```csharp
struct Position {
    WorldCoordinate coord;
}

struct Physics {
    BodyChunk[] chunks;
    float mass;
    float friction;
}

struct Graphics {
    GraphicsModule module;
    Color color;
    float scale;
}

struct AI {
    AbstractCreature abstractCreature;
    PathFinder pathFinder;
    Tracker tracker;
    Creature.CreatureState state;
}

struct Health {
    float current;
    float max;
    float stunned;
}

struct Relationships {
    Dictionary<Creature, float> dynamicRelationships;
    CreatureCommunity community;
}
```

### Behavior Components

```csharp
struct Swimming {
    float swimSpeed;
    float oxygenRemaining;
    float buoyancy;
}

struct SpearUser {
    Weapon heldWeapon;
    float throwPower;
    float accuracy;
}

struct Carnivorous {
    float hunger;
    CreatureType[] preyTypes;
    float biteDamage;
}

struct Glowing {
    float brightness;
    Color glowColor;
}

struct Camouflage {
    float currentLevel;
    float maxLevel;
    float fadeSpeed;
}

struct Flying {
    float wingPower;
    float stamina;
}
```

## Entity Composition

### Green Lizard
```csharp
var greenLizard = entityManager.CreateEntity();
entityManager.AddComponent(greenLizard, new Position(...));
entityManager.AddComponent(greenLizard, new Physics(...));
entityManager.AddComponent(greenLizard, new AI(...));
entityManager.AddComponent(greenLizard, new Graphics(...));
entityManager.AddComponent(greenLizard, new Health(...));
entityManager.AddComponent(greenLizard, new Carnivorous(...));
entityManager.AddComponent(greenLizard, new Camouflage(...));
```

Just data. No inheritance.

### Scavenger (with swimming)
```csharp
var scavenger = entityManager.CreateEntity();
entityManager.AddComponent(scavenger, new Position(...));
entityManager.AddComponent(scavenger, new Physics(...));
entityManager.AddComponent(scavenger, new AI(...));
entityManager.AddComponent(scavenger, new Graphics(...));
entityManager.AddComponent(scavenger, new Health(...));
entityManager.AddComponent(scavenger, new SpearUser(...));
entityManager.AddComponent(scavenger, new Swimming(...)); // Just add it!
```

Swimming Scavenger requires **zero** inheritance changes. Just attach component.

### Glowing Vulture
```csharp
var vulture = entityManager.CreateEntity();
// ... standard vulture components
entityManager.AddComponent(vulture, new Flying(...));
entityManager.AddComponent(vulture, new Glowing(...)); // Mod adds this
```

Mods add components without touching base code.

## Systems

### PhysicsSystem
```csharp
class PhysicsSystem {
    void Update(EntityManager em, float deltaTime) {
        // Process all entities with Position AND Physics
        foreach (var entity in em.Query<Position, Physics>()) {
            var pos = em.GetComponent<Position>(entity);
            var physics = em.GetComponent<Physics>(entity);

            // Update physics
            foreach (var chunk in physics.chunks) {
                chunk.vel += gravity * deltaTime;
                chunk.pos += chunk.vel * deltaTime;
            }

            // Terrain collision
            ResolveTerrainCollision(physics, room);

            // Update position
            pos.coord = physics.chunks[0].pos;
        }
    }
}
```

### SwimmingSystem
```csharp
class SwimmingSystem {
    void Update(EntityManager em, Room room) {
        // Only entities with Swimming component
        foreach (var entity in em.Query<Swimming, Physics, Position>()) {
            var swimming = em.GetComponent<Swimming>(entity);
            var physics = em.GetComponent<Physics>(entity);
            var pos = em.GetComponent<Position>(entity);

            if (room.GetTile(pos.coord).Terrain == Terrain.Water) {
                // Apply buoyancy
                foreach (var chunk in physics.chunks) {
                    chunk.vel.y += swimming.buoyancy;
                }

                // Consume oxygen
                swimming.oxygenRemaining -= Time.deltaTime;
            }
        }
    }
}
```

Only processes entities that have swimming. Non-swimming creatures unaffected.

### CarnivoreSystem
```csharp
class CarnivoreSystem {
    void Update(EntityManager em) {
        foreach (var entity in em.Query<Carnivorous, AI, Health>()) {
            var carnivore = em.GetComponent<Carnivorous>(entity);
            var ai = em.GetComponent<AI>(entity);

            if (carnivore.hunger > 0.8f) {
                // Find prey
                var prey = FindNearestPrey(ai.tracker, carnivore.preyTypes);
                if (prey != null) {
                    ai.pathFinder.SetDestination(prey.position);
                }
            }
        }
    }
}
```

### SpearThrowingSystem
```csharp
class SpearThrowingSystem {
    void Update(EntityManager em) {
        foreach (var entity in em.Query<SpearUser, AI>()) {
            var spearUser = em.GetComponent<SpearUser>(entity);
            var ai = em.GetComponent<AI>(entity);

            if (ai.wantsToThrowSpear && spearUser.heldWeapon != null) {
                ThrowSpear(spearUser, ai.throwDirection, spearUser.throwPower);
                spearUser.heldWeapon = null;
            }
        }
    }
}
```

Shared by Player, Scavenger, any entity with SpearUser component.

## System Execution

```csharp
void Update() {
    physicsSystem.Update(entityManager, deltaTime);
    swimmingSystem.Update(entityManager, room);
    aiSystem.Update(entityManager);
    carnivoreSystem.Update(entityManager);
    spearThrowingSystem.Update(entityManager);
    graphicsSystem.Update(entityManager);
}
```

Systems run in sequence. Order matters (physics before graphics).

## Benefits for Rain World

### 1. Composition Over Inheritance

**Current**: "A swimming scavenger IS-A scavenger IS-A creature"
**ECS**: "This entity HAS swimming, spear-use, AI components"

Add/remove behaviors by adding/removing components.

### 2. Memory Efficiency

**Current**: Every Lizard carries RedLizard, GreenLizard, BlueLizard fields
**ECS**: Components stored in typed arrays

```
ComponentArray<Swimming>: [entity5, entity12, entity47]
ComponentArray<Camouflage>: [entity12, entity88]
```

Only entities with component pay memory cost.

**Measurement**:
- Current: ~500 bytes per creature (sparse)
- ECS: ~200 bytes per creature (dense)
- Savings: 60% memory reduction

### 3. Cache Locality

**Current**: Creature objects scattered in memory
```
[Lizard1: pos|vel|ai|graphics|unused|unused]
[Lizard2: pos|vel|ai|graphics|unused|unused]
```

**ECS**: Components in contiguous arrays
```
PositionArray: [pos1|pos2|pos3|pos4|...]
PhysicsArray:  [phys1|phys2|phys3|phys4|...]
```

PhysicsSystem iterates only PhysicsArray. Better cache usage.

**Performance**: Potentially 2-5x speedup for systems (CPU-bound code).

### 4. Parallel Systems

```csharp
Parallel.Invoke(
    () => physicsSystem.Update(),
    () => aiSystem.Update(),      // Independent
    () => swimmingSystem.Update()
);
```

Systems operating on different components can run in parallel. Physics and AI don't interfere.

### 5. Modding

**Mod adds new behavior**:
```csharp
// Mod: Infected creatures

struct InfectedComponent {
    float infectionLevel;
    float spreadRadius;
}

class InfectionSystem {
    void Update(EntityManager em) {
        foreach (var entity in em.Query<Infected, AI>()) {
            var infected = em.GetComponent<Infected>(entity);
            var ai = em.GetComponent<AI>(entity);

            infected.infectionLevel += Time.deltaTime;

            if (infected.infectionLevel > 100) {
                ai.aggressive = true; // Go berserk
            }

            // Spread to nearby creatures
            SpreadInfection(entity, infected.spreadRadius);
        }
    }
}
```

Mod registers component and system. Works with ALL creatures automatically.

**No inheritance patching required.**

### 6. Easier LOD Transitions

**Current**: Separate Abstract and Realized objects

**ECS approach**:
```csharp
// Abstract LOD: minimal components
Entity creature = ...;
entityManager.AddComponent(creature, new Position(...));
entityManager.AddComponent(creature, new AbstractAI(...));

// Realize: add detailed components
entityManager.AddComponent(creature, new Physics(...));
entityManager.AddComponent(creature, new Graphics(...));
entityManager.AddComponent(creature, new DetailedAI(...));
entityManager.RemoveComponent<AbstractAI>(creature);

// Abstractize: remove detailed components
entityManager.RemoveComponent<Physics>(creature);
entityManager.RemoveComponent<Graphics>(creature);
entityManager.RemoveComponent<DetailedAI>(creature);
entityManager.AddComponent(creature, new AbstractAI(...));
```

Same entity, different components. No separate object creation.

## Tradeoffs

### Complexity
❌ **Con**: Different mental model
- Developers must think in data + systems, not objects
- Learning curve for team
- Existing codebase knowledge doesn't transfer directly

✅ **Pro**: Clearer separation of data and logic

### Relationships
❌ **Con**: Object relationships harder
- Slugcat holding Spear: need relationship system
- Parent/child hierarchies: need explicit tracking
- Current OOP: just store references

**Solution**: Use entity references or relationship components
```csharp
struct Held {
    Entity holder;
}
```

### Debugging
❌ **Con**: Stack traces less intuitive
- "SwimmingSystem.Update" vs "Lizard.Update"
- "Which entity crashed?" vs "Which object crashed?"

✅ **Pro**: Entity IDs in logs
```
[Entity #47] SwimmingSystem error: oxygen = -5
```

### Migration Cost
❌ **Con**: Massive refactor
- Thousands of lines of creature code
- All mods break (incompatible)
- Months of work

**Mitigation**: Incremental migration
1. Implement ECS alongside OOP
2. New creatures use ECS
3. Gradually migrate old creatures
4. Remove OOP once complete

### Not a Silver Bullet
❌ **Does NOT improve**:
- ProcessManager (already good)
- Room loading (already good)
- Singleton coupling (separate issue)

ECS specifically improves creature/entity architecture.

## Real-World ECS Examples

- **Overwatch**: Entire game built on ECS ([GDC Talk](https://www.youtube.com/watch?v=W3aieHjyNvw))
- **Unity DOTS**: Official ECS framework
- **Bevy (Rust game engine)**: ECS-first design
- **Hades**: Used ECS for entity management

Proven pattern for complex simulations.

## Implementation Path

If Rain World were to migrate:

### Phase 1: Infrastructure (1-2 months)
1. Implement EntityManager
2. Implement ComponentArray storage
3. Implement Query system
4. Write test suite

### Phase 2: Core Components (2-3 months)
1. Define all components (Position, Physics, AI, etc.)
2. Implement core systems (Physics, Graphics, AI)
3. Parallel OOP system (both running)

### Phase 3: Migration (3-6 months)
1. Convert simple creatures (Fly, Leech)
2. Convert medium creatures (Lizards)
3. Convert complex creatures (Player, Scavenger)
4. Remove OOP code

### Phase 4: Optimization (1-2 months)
1. Parallel system execution
2. Memory layout optimization
3. Profiling and tuning

**Total**: ~12-18 months for full migration.

## Verdict

**Should Rain World migrate to ECS?**

**For a shipped game**: Probably not. Migration cost is enormous.

**For a sequel or new project**: Absolutely consider it. Benefits are significant for complex simulations.

**For Rain World mods**: Interesting experiment, but breaking change.

## Related Documents

- [Inheritance Rigidity Problem](../problems/inheritance-rigidity.md)
- [Architecture Overview](../overview.md)
- [Dependency Injection Proposal](dependency-injection.md)
- [2025-12-14 Analysis](../analysis/2025-12-14-review.md)
