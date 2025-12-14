# The Dual LOD System

**Reading time: 8-10 minutes**

Rain World's most distinctive technical feature is its dual Level-of-Detail (LOD) system. Unlike traditional LOD that only affects graphics quality, Rain World's LOD system fundamentally changes how entities *exist and think*.

This document explains how the abstract and realized layers work, why they're necessary, and what happens during transitions.

## The Problem: Simulating an Ecosystem

Imagine you're in Industrial Complex, a region with ~70 interconnected rooms. At any moment, dozens of creatures roam this space: lizards hunting, scavengers patrolling, vultures circling, squidcadas fleeing. The player occupies just one or two rooms.

**Challenge**: How do you simulate all these creatures without melting the CPU?

**Naive solution**: Only simulate what's visible.
- **Fails because**: The world feels fake. Return to a room and creatures have respawned in the exact same spots. No sense of persistence or ecology.

**Brute force solution**: Simulate everything in full detail all the time.
- **Fails because**: Performance collapse. 70 rooms × 5 creatures each = 350 creatures running pathfinding, collision detection, and detailed AI every frame. Impossible on the target hardware.

**Rain World's solution**: Creatures exist in **two forms simultaneously**—cheap when far (abstract), expensive when near (realized)—and seamlessly transition between them.

## The Two Layers

### Abstract Layer: The Strategic Simulation

When a creature is far from the player (not in the current room or adjacent rooms), it exists only as an **AbstractCreature**.

**What AbstractCreature contains**:
- `Room` reference (which room it's currently in)
- `creatureTemplate` (species type: pink lizard, green lizard, scavenger, etc.)
- `state` (CreatureState: hunger level, relationships, personality traits)
- `abstractAI` (AbstractCreatureAI: simplified decision-making)
- `pos` (position within the abstract room, not pixel-precise)
- `abstractPhysicalObject` data (for persistence)

**What AbstractCreatureAI does**:
- Decides whether to **migrate** to another room (based on hunger, threats, pathing behavior)
- Tracks **relationships** with other abstract creatures
- Resolves abstract **interactions** (hunting, fleeing, combat) via probabilistic models
- Updates **every few frames** instead of every frame

Think of abstract creatures as playing a **board game**. They move between rooms (board squares) based on dice rolls weighted by hunger, danger, and personality. When two abstract creatures meet in the same room, they interact via simplified combat resolution ("this creature has 80% chance to eat that creature").

**Example**:
A pink lizard in abstract form:
- Checks hunger level
- Decides to migrate toward sound of prey (abstract pathfinding through room network)
- Moves from room LF_A04 to LF_A05
- Encounters abstract scavenger in LF_A05
- Combat resolution: lizard's threat level (7) vs scavenger's with spear (6)
- Outcome: lizard flees to adjacent room

This all happens with minimal CPU cost because there's no detailed pathfinding, no pixel-level collision, no animation. Just state machines and probability tables.

### Realized Layer: The Tactical Simulation

When a room becomes active (player enters or is about to enter), all abstract entities in that room **realize**—they spawn full-detail versions.

**What happens during realization**:

1. **AbstractCreature** creates a **Creature** instance (the realized form)
2. The Creature instance:
   - Spawns `BodyChunks` (physics circles for collision)
   - Creates `ArtificialIntelligence` (detailed behavior AI)
   - Initializes position from the abstract's room position
   - Reads state from AbstractCreature.state (hunger, relationships, etc.)

3. If the creature is visible on camera:
   - `InitiateGraphicsModule()` creates a GraphicsModule
   - The GraphicsModule handles rendering, animations, visual effects

Now the creature is **fully simulated**:
- AI runs every frame (pathfinding, behavior state machine, sensory processing)
- Physics updates every frame (velocity integration, collision with terrain and other creatures)
- Graphics module renders animations based on AI state

**Example**:
The same pink lizard, now realized:
- Full pathfinding using Dijkstra through terrain tiles
- Sensory system: sees player, hears sounds, smells prey
- Behavior state machine: HUNT → CHASE → ATTACK states
- Physics: 5 body chunks (head, 3 body segments, tail) with spring connections
- Graphics: smooth animation blending between walk/run/lunge sprites

This is computationally expensive, which is why only 1-3 rooms are realized at once.

## The Transition: Abstract ↔ Realized

### Realization (Abstract → Realized)

**When**: Player approaches a room boundary, triggering progressive loading.

**Process**:

1. **RoomPreparer** starts loading the next room in background
2. **Room realization**:
   - Load room geometry (tiles, water, objects)
   - Initialize physics grid
   - Set up collision data structures

3. **Entity realization** for each AbstractPhysicalObject in the room:
   ```csharp
   // Simplified pseudocode
   AbstractCreature absCreature = /* from room's entity list */;
   Creature realCreature = absCreature.Realize();
   realCreature.PlaceInRoom(room);
   ```

4. **State transfer** (abstract → realized):
   - Position: abstract room position → spawn coordinates in realized room
   - Health: abstract health → creature health value
   - Relationships: abstract memory → creature's AI memory
   - State: abstract hunger/fear → creature behavior state

5. **Graphics initialization** (if visible):
   - Create GraphicsModule
   - Initialize sprites and shaders

**What gets lost**: Transient state that wasn't saved to the abstract form. For instance:
- Exact animation frame
- Precise subpixel position within the room
- Current velocity/momentum (resets to zero)
- Temporary behavior modifiers

**Why the loss is acceptable**: These details don't matter for ecosystem persistence. What matters is that the lizard is still in that room, still hunting, still remembers you threw a spear at it.

### Abstraction (Realized → Abstract)

**When**: Player moves far enough away that the room is no longer needed.

**Process**:

1. **State transfer** (realized → abstract):
   ```csharp
   creature.abstractCreature.state = creature.SaveState();
   ```

   What gets saved:
   - Current room position
   - Health/injuries
   - Hunger level
   - Relationships and grudges
   - Important state flags (e.g., "this creature saw the player")

2. **Cleanup**:
   - Destroy GraphicsModule (if exists)
   - Destroy BodyChunks
   - Destroy ArtificialIntelligence
   - Destroy Creature instance

3. **AbstractCreature persists**:
   - Continues existing in the abstract layer
   - Continues updating (migration, abstract interactions)
   - Can realize again if player returns

**What gets lost**: Everything not explicitly saved—animation state, exact physics state, active timers. Again, this is acceptable because the abstract simulation captures what matters.

## Why This Works: The Performance Math

Let's calculate the savings:

**Realized creature cost** (per frame):
- AI pathfinding: ~500 instructions
- Behavior state machine: ~200 instructions
- Physics (5 body chunks): ~1000 instructions (collision, integration, constraints)
- Graphics: ~800 instructions (animation blending, sprite updates)
- **Total**: ~2500 instructions per creature per frame

**Abstract creature cost** (per update cycle, every ~5 frames):
- Abstract AI: ~100 instructions (simple state machine)
- Migration check: ~50 instructions
- Relationship update: ~30 instructions
- **Total**: ~180 instructions per creature per ~5 frames = ~36 instructions/frame

**Savings ratio**: 2500 / 36 ≈ **70x cheaper** for abstract creatures.

**In practice**:
- Industrial Complex: ~350 creatures total
- 3 realized rooms with ~15 creatures realized = 15 × 2500 = 37,500 instructions/frame
- 335 abstract creatures = 335 × 36 = 12,060 instructions/frame
- **Total**: ~50,000 instructions/frame

If all creatures were realized: 350 × 2500 = 875,000 instructions/frame → **17x worse performance**.

This is why the ecosystem simulation is even possible.

## Concrete Gameplay Examples

### Example 1: The Persistent Lizard

**Scenario**: You enter a room, see a blue lizard, throw a rock at it, and flee to another room.

**What happens**:
1. **Realized phase**:
   - Blue lizard chases you (full AI, pathfinding)
   - You damage it with the rock (physics collision, health reduction)
   - Lizard's AI updates relationship: "hostile to player"

2. **You leave the room**:
   - Blue lizard abstracts
   - State saved: "hostile to player, 70% health, in room LF_C12"

3. **10 minutes later** (many game cycles):
   - Blue lizard's abstract AI migrates to adjacent rooms hunting
   - Now in room LF_C14 (two rooms away)

4. **You return to original room LF_C12**:
   - Room realizes, but blue lizard isn't there (it migrated)
   - Persistence achieved: creature continues existing independently

5. **You enter LF_C14**:
   - Blue lizard realizes from abstract form
   - Still hostile, still at 70% health
   - Remembers you and immediately attacks

This creates the feeling of a living world. The lizard didn't disappear—it continued existing and moving.

### Example 2: The Scavenger War

**Scenario**: A vulture attacks a scavenger patrol in an abstract room while you're three rooms away.

**What happens** (all in abstract layer):
1. Vulture's abstract AI: high hunger, initiates combat
2. Combat resolution:
   - Vulture threat: 10
   - 4 scavengers with spears threat: 4 × 4 = 16
   - Outcome: Vulture retreats, but kills 1 scavenger first

3. Abstract state updates:
   - 1 scavenger AbstractCreature marked dead
   - Vulture health reduced to 60%
   - Scavenger faction relationship: "hostile to vultures ++"

4. **You enter the room 2 minutes later**:
   - Room realizes
   - 3 living scavengers realize (hostile to vultures)
   - 1 dead scavenger body realizes on the floor
   - Vulture is gone (migrated while wounded)

You discover the aftermath of a battle you never saw. This is emergent narrative from abstract simulation.

## Limitations and Tradeoffs

### Data Loss on Transition

**Problem**: State not saved to abstract form is lost.

**Example**: A lizard mid-lunge when the room abstracts. When it realizes again later, it's in a neutral stance, not lunging.

**Impact**: Minor visual discontinuity if player leaves and immediately returns. Mitigated by keeping rooms realized briefly after leaving.

### Abstract AI Simplification

**Problem**: Abstract AI is less sophisticated than realized AI.

**Example**: A realized lizard can pathfind around complex terrain, through tunnels, around obstacles. An abstract lizard just moves between rooms via connections—it doesn't understand detailed geometry.

**Impact**: Creatures sometimes take unexpected routes in abstract form. A lizard that would navigate a complex room carefully while realized might "teleport" through it when abstract.

**Mitigation**: Abstract pathfinding uses room connectivity graphs, ensuring logical (if not geometrically perfect) routes.

### Memory Overhead

**Problem**: Every creature exists twice—both abstract and (when realized) realized instances.

**Example**: A realized lizard has both its Creature instance (with BodyChunks, AI, graphics) and its AbstractCreature (with abstract state).

**Impact**: ~2x memory overhead for realized creatures compared to a purely realized system.

**Justification**: This is cheaper than the alternative (no abstract layer → must realize everything → impossible).

## Implementation Details

### Key Classes

**Abstract layer**:
- `AbstractWorldEntity`: Base class for all abstract entities
- `AbstractPhysicalObject`: Subclass for objects that can realize
- `AbstractCreature`: Subclass for creatures
- `AbstractCreatureAI`: AI for abstract creatures
- `CreatureState`: Stores persistent creature state

**Realized layer**:
- `UpdatableAndDeletable`: Base class for all realized entities
- `PhysicalObject`: Realizes AbstractPhysicalObject
- `Creature`: Realizes AbstractCreature
- `ArtificialIntelligence`: AI for realized creatures

**Bridging**:
- `AbstractPhysicalObject.realizedObject`: Reference to realized form (null when abstract-only)
- `PhysicalObject.abstractPhysicalObject`: Reference back to abstract form

### Synchronization

Realized and abstract forms stay synchronized:

```csharp
// Realized creature updates its abstract position each frame
creature.abstractCreature.pos = creature.room.GetAbstractPosition(creature.mainBodyChunk.pos);

// Abstract form always knows where the realized form is
if (abstractCreature.realizedObject != null) {
    abstractCreature.pos = abstractCreature.realizedObject.room.GetAbstractPosition(...);
}
```

When the creature abstracts, final state sync happens:
```csharp
abstractCreature.state = creature.SaveState(); // Transfer health, hunger, relationships, etc.
```

## Comparison to Traditional LOD

**Traditional LOD** (e.g., in 3D games):
- High-poly model when close
- Low-poly model when far
- Affects only **graphics**
- Logic/AI unchanged

**Rain World's LOD**:
- Full simulation (AI + physics + graphics) when close
- Abstract simulation (simplified AI, no physics/graphics) when far
- Affects **AI, physics, and graphics**
- Fundamentally different execution models

This is why Rain World's approach is novel—it's LOD for *game logic*, not just rendering.

## Summary

The dual LOD system enables Rain World's persistent ecosystem by running two simulations simultaneously:

- **Abstract layer**: Cheap, always-on, strategic simulation of the entire region
- **Realized layer**: Expensive, on-demand, tactical simulation of nearby areas

Transitions between the two are seamless (from the player's perspective), though some transient state is lost. This tradeoff—accepting minor discontinuities for massive performance gains—is what makes the ecosystem simulation feasible.

The result: a world that feels alive. Creatures persist, migrate, hunt, and die whether you're watching or not. The code architecture directly enables this core experience.

---

**Next**: [World & Room Management](world-rooms.md) explains how the world streams and manages these realized/abstract transitions.
