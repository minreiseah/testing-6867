# Problem: Inheritance Rigidity

**Status**: Identified
**Date**: 2025-12-14
**Severity**: Medium
**Related**: [ECS Migration Proposal](../improvements/ecs-migration.md)

## The Pattern

Rain World uses deep inheritance hierarchies for game objects:

```
UpdatableAndDeletable
  └─ PhysicalObject
      ├─ Creature
      │   ├─ Player
      │   ├─ Lizard
      │   │   ├─ GreenLizard
      │   │   ├─ PinkLizard
      │   │   ├─ RedLizard
      │   │   └─ ...
      │   ├─ Scavenger
      │   ├─ Vulture
      │   └─ ...
      └─ Item
          ├─ Spear
          ├─ Rock
          └─ ...
```

Each creature type inherits:
- Physics (BodyChunks)
- Graphics (rendering)
- AI (behavior)
- State management

## Concrete Problems

### 1. Rigid Categorization

Objects must fit into single inheritance chain.

**Example**: Scavenger

```csharp
class Scavenger : Creature {
    // Can throw spears
    void ThrowSpear() { ... }

    // Can pick up items
    void GrabItem(Item item) { ... }

    // Has complex AI
    void UpdateAI() { ... }
}
```

**Problem**: Want a creature that's "like a Scavenger but also swims"?

**Options**:
1. ❌ **Inherit from Scavenger and add swimming** - Works, but creates SwimmingScavenger subclass
2. ❌ **Add swimming to Scavenger** - Now ALL scavengers can swim (wrong)
3. ❌ **Create SwimmableCreature intermediate class** - Breaks existing hierarchy
4. ✅ **Composition** - But not how codebase is structured

Multiple inheritance would help, but C# doesn't support it.

### 2. Code Duplication

Both Scavenger and Player can throw spears, but implementation must be duplicated or awkwardly factored.

```csharp
class Player : Creature {
    void ThrowSpear() {
        // Duplicate logic from Scavenger
        var spear = heldObject as Spear;
        if (spear != null) {
            spear.Throw(throwDirection, throwPower);
            heldObject = null;
        }
    }
}

class Scavenger : Creature {
    void ThrowSpear() {
        // Same logic duplicated
        var spear = heldObject as Spear;
        if (spear != null) {
            spear.Throw(throwDirection, throwPower);
            heldObject = null;
        }
    }
}
```

**Attempted solutions**:
- Static helper methods (breaks OOP)
- Interface with default implementation (complex)
- Extract to base class (bloats Creature)

None are clean with deep inheritance.

### 3. Memory Waste

Each Lizard variant carries fields for ALL possible behaviors.

```csharp
class Lizard : Creature {
    // Green lizards don't need these
    float redLizardRageTimer;
    bool redLizardEnraged;

    // Red lizards don't need these
    float greenLizardCamouflageLevel;
    bool greenLizardHiding;

    // Pink lizards don't need these
    float blueLizardElectricCharge;

    // But all Lizards carry all fields
}
```

**Impact**:
- Memory bloat (hundreds of creatures in world)
- Cache pollution (CPU loads unused fields)
- Maintenance burden (more fields to track)

**Alternative approach**: Subclass per variant

```csharp
class GreenLizard : Lizard {
    float camouflageLevel;
}

class RedLizard : Lizard {
    float rageTimer;
}
```

But now shared logic is harder (must be in base Lizard class).

### 4. Cross-cutting Behaviors

Want to add "all creatures that can swim"?

**Current approach**: Add to base Creature class

```csharp
class Creature : PhysicalObject {
    // Not all creatures swim!
    bool canSwim;
    float swimSpeed;

    void UpdateSwimming() {
        if (canSwim) {
            // Swimming logic
        }
    }
}
```

**Problems**:
- Base class bloats with optional features
- Every creature pays memory cost
- Conditional logic everywhere (`if (canSwim)`)

**Better approach**: Composition

```csharp
class Creature {
    SwimmingBehavior swimming; // null if can't swim
}
```

But codebase isn't structured this way.

### 5. Modification Difficulty

**Scenario**: Add new behavior "creatures that glow in dark"

**Current approach**:
1. Add fields to Creature base class
2. Add conditional update logic
3. All creatures now carry glow state (even if unused)

**Or**:
1. Create GlowingCreature intermediate class
2. Reparent specific creatures to inherit from it
3. Breaks existing hierarchy
4. Existing mods that patch those creatures break

**Impact**: Adding features becomes architectural decision affecting entire codebase.

### 6. The "Gorilla/Banana Problem"

> "You wanted a banana but you got a gorilla holding the banana and the entire jungle."

**Example**: Want to reuse Lizard AI for new creature

```csharp
class MyCustomCreature : Lizard {
    // Inherits:
    // - Lizard AI ✓ (wanted)
    // - Lizard graphics ✗ (don't want, must override)
    // - Lizard physics ✗ (don't want, must override)
    // - Lizard state ✗ (don't want, must override)
    // - All Lizard fields ✗ (memory waste)
}
```

Wanted just the AI, got everything.

**Actual mod approach**: Copy-paste Lizard AI into new creature. Duplication.

## Real Impact in Rain World

### Mod Ecosystem

**Common mod pattern**:
1. Create new creature type
2. Inherit from existing similar creature
3. Override specific behaviors
4. Fight inherited baggage

**Example**: Swimming Scavenger mod
- Inherits from Scavenger (gets item pickup, throwing, AI)
- Must override physics to add buoyancy
- Must override graphics to add swimming animation
- Carries all Scavenger fields even if unused

### Performance

**Creature count in typical world**: ~200-400 entities
**Memory overhead**: Each carries fields for ALL possible behaviors
**CPU cache**: Poor locality due to sparse field usage

**Measurement** (estimated):
- Average creature: ~500 bytes
- Used fields: ~200 bytes
- Wasted: ~300 bytes (60%)
- Total waste: 60KB-120KB per world

Small in absolute terms, but impacts cache efficiency.

### Code Complexity

**Lines of code in creature hierarchy**: Likely 10,000+
**Conditional logic**: Hundreds of `if (creatureType == ...)` checks
**Override methods**: Each subclass overrides 10-20 base methods

**Maintenance burden**: Changing base Creature affects all subclasses.

## Why It Happened

**Historical reasons**:
1. **Natural OOP thinking**: "A lizard IS-A creature"
2. **Rapid prototyping**: Inheritance is quick
3. **Single developer initially**: Easier to change
4. **Worked well enough**: Shipped successful game

These are valid pragmatic choices. Inheritance isn't wrong, just has tradeoffs.

## When Inheritance Works

**Good use of inheritance in Rain World**:
- `UpdatableAndDeletable` - Shared lifecycle makes sense
- `MainLoopProcess` - Different game modes truly are variants
- `GraphicsModule` - Rendering variations fit inheritance

**Key difference**: These are shallow hierarchies with true "is-a" relationships.

## When Composition is Better

**Creatures**: Composition of behaviors
```
Entity
├─ PhysicsComponent
├─ AIComponent
├─ GraphicsComponent
└─ SwimmingComponent (optional)
```

Each creature is a **bag of components**, not a taxonomic classification.

## Solutions

See: [ECS Migration Proposal](../improvements/ecs-migration.md)

**Summary**:
- Entity Component System for creatures
- Components for physics, AI, graphics
- Systems process entities with specific components
- Composition over inheritance

## Related Documents

- [Architecture Overview](../overview.md)
- [ECS Migration Proposal](../improvements/ecs-migration.md)
- [Singleton Coupling Problem](singleton-coupling.md)
- [2025-12-14 Analysis](../analysis/2025-12-14-review.md)
