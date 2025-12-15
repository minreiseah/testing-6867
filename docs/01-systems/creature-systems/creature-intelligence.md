# Creature Intelligence

**Layer**: 1 (Systems Architecture)
**Reading time**: 20 minutes

## Purpose

This document explains how creature AI works in Rain World's dual LOD system. Creatures have two forms of intelligence: abstract AI for strategic decisions when far away, and realized AI for tactical behavior when near the player.

## The Dual AI System

Rain World creatures think differently depending on their distance from the player:

| AI Mode | Distance | Update Rate | Purpose | Example Decisions |
|---------|----------|-------------|---------|-------------------|
| **Abstract AI** | Far from player (unrealized rooms) | Every 3-5 frames (~8-13 TPS) | Strategic ecosystem simulation | "Migrate to adjacent room", "Hunt for food", "Flee from threat" |
| **Realized AI** | Near player (realized rooms) | Every frame (40 TPS) | Tactical gameplay interactions | "Path to this tile", "Attack player now", "Take cover behind object" |

**Key insight**: The same creature seamlessly transitions between two AI systems as the player moves through the world.

## Abstract AI: Strategic Layer

### What It Does

When a creature is in an unloaded room (far from player):
- Exists as `AbstractCreature` with `AbstractCreatureAI`
- Makes high-level decisions about movement and survival
- Interacts with other abstract creatures via simplified models
- Maintains ecosystem state (hunger, relationships, territory)

### Decision Types

**Migration**:
```
AbstractCreatureAI evaluates:
- Hunger level → need to find food?
- Threat perception → danger in current room?
- Territory → is this my home?
- Time of day → denning behavior?

→ Probability-based decision to move to adjacent room
```

**Abstract Combat**:
```
Two abstract creatures in same room:
- Compare threat levels
- Dice roll modified by:
  - Creature types (lizard vs scavenger)
  - Health states
  - Hunger levels

→ One flees, one wins, or both ignore
```

**Resource Seeking**:
```
AbstractCreatureAI tracks:
- Hunger increasing over time
- Known food sources (other creatures, plants)
- Path to food via room graph

→ Migrate toward food source
```

### Performance Characteristics

**Update frequency**: Every 3-5 frames
- Specific delay per creature: `3 + (entityID % 2)` (either 3 or 4 frames)
- Spreads CPU load evenly across frames
- ~8-13 decisions per second per creature

**CPU cost**: ~1× (baseline)
- No pathfinding (just room-to-room graph)
- No physics simulation
- Simple probability calculations
- Minimal memory access

**Scalability**: Can simulate 200-400 creatures
- Most creatures are abstract most of the time
- Entire ecosystem runs in abstract mode

### What Gets Saved

When a creature transitions from realized → abstract:
- **Position**: Current room (not exact coordinates within room)
- **State**: Health, hunger, relationships
- **Inventory**: Held items
- **Memory**: Threat tracking, den locations

What **doesn't** get saved:
- Exact position within room
- Current animation state
- Velocity/momentum
- Transient behaviors (mid-jump, mid-attack)
- Short-term memory (saw player 1 second ago)

**Tradeoff**: Data loss enables memory efficiency. Most lost data isn't needed for ecosystem simulation.

## Realized AI: Tactical Layer

### What It Does

When a creature is in a loaded room (near player):
- Exists as `Creature` (PhysicalObject subclass)
- Has `ArtificialIntelligence` component for detailed behavior
- Runs pathfinding, behavior state machines, sensory processing
- Fully simulated physics and collision

### Decision Types

**Pathfinding**:
```
ArtificialIntelligence:
- Sensory input: See player, hear sound
- Goal selection: Chase, flee, or investigate
- Pathfinding: A* on tile grid
- Movement: Apply forces to BodyChunks

→ Frame-by-frame navigation
```

**Behavior States**:
```
State machine examples:
- Idle → Spotted Player → Chase
- Chase → Lost Sight → Search
- Search → Timeout → Idle
- Idle → Hungry → Hunt Food
```

**Combat Decisions**:
```
Every frame evaluation:
- Distance to player
- Player facing direction
- Own health/stamina
- Nearby threats

→ Attack, dodge, retreat, or reposition
```

### AI Components

Different creature types have specialized AI classes:

**Lizard AI**:
- Track focus (prey, threat, or curiosity)
- Evaluate climbing paths
- Coordination with pack (if applicable)
- Camouflage behavior (green lizards)
- Rage state (red lizards)

**Scavenger AI**:
- Complex social behavior
- Item usage (throw spears, use weapons)
- Communication with other scavengers
- Reputation tracking with player
- Tactical positioning

**Vulture AI**:
- Flight pathfinding (3D space)
- Dive attack timing
- Mask protection behavior
- Terrain awareness (avoid obstacles while flying)

### Performance Characteristics

**Update frequency**: Every frame (40 TPS)

**CPU cost**: ~50× abstract AI cost
- Full pathfinding: A* search every few frames
- Behavior state evaluation every frame
- Physics simulation: BodyChunks, constraints, collision
- Sensory processing: Raycasts for vision, sound propagation

**Scalability**: Limited to ~10-20 creatures
- Only creatures in/near player's room
- Performance scales with active creature count

## The Transition: Realize and Abstractize

### Realization (Abstract → Realized)

Triggered when player approaches a room:

```
1. Room begins realization (RoomPreparer)
2. AbstractCreature in room receives Realize() call
3. Creates Creature (PhysicalObject)
   - Initializes BodyChunks at abstract position
   - Creates ArtificialIntelligence
   - Loads state from AbstractCreature
4. Links: abstractCreature.realizedCreature = creature
5. GraphicsModule created when camera sees it
6. Begins updating at 40 TPS
```

**Data transfer**:
- Position → Spawn at room entrance or den
- Health, hunger → Direct copy
- Relationships → Threat levels, pack bonds
- Inventory → Realized items

**Data reconstruction**:
- Physics: BodyChunks spawn at rest
- Animation: Starts from idle state
- Exact position: Placed at abstract coordinates (room entrance, den, etc.)

### Abstraction (Realized → Abstract)

Triggered when player moves far away:

```
1. Player leaves room
2. After delay (room stays realized briefly)
3. Creature.Abstractize() called
4. Saves critical state to AbstractCreature:
   - Current room
   - Health, hunger
   - Held items
5. Destroys Creature (PhysicalObject)
6. AbstractCreature continues in abstract mode
```

**Data loss** (intentional):
- Exact position within room
- Current behavior state (chasing, fleeing)
- Momentum/velocity
- Animation frame
- Short-term memory

**Why allow data loss?**
- Reduces memory footprint dramatically
- Player can't observe the loss (they're far away)
- Ecosystem simulation doesn't need this precision

## AI State Persistence

### What Persists Across Transitions

**Always persists**:
- Health and hunger
- Current room location
- Inventory (held items)
- Long-term memory (den location, threat levels)
- Relationships with player and other creatures

**Sometimes persists**:
- Pursuit behavior: If chasing player and player leaves room, abstract AI may migrate to follow
- Fear state: High threat level can cause fleeing behavior in abstract mode
- Pack behavior: Abstract creatures remember pack affiliations

**Never persists**:
- Exact coordinates
- Current path
- Mid-action states (mid-jump, mid-grab)
- Visual memory of player (if not in same room)

### Example: Lizard Chase Scenario

```
Frame 1000: Player in Room A
- Green Lizard (realized) spots player
- Enters Chase state
- Pathfinds toward player

Frame 1500: Player exits Room A to Room B
- Lizard still in Chase state
- Continues pursuit toward room exit

Frame 1600: Room A begins abstraction
- Lizard.Abstractize() called
- Chase state LOST
- AbstractLizard retains: high threat level toward player

Frame 1650: AbstractLizard in Room A (abstract)
- Evaluates migration
- High threat level → probability to flee OR pursue
- May migrate toward Room B (if very aggressive type)
- Or may flee to Room C (if timid type)

Frame 2000: Player returns to Room A
- Room A realizes
- Lizard realizes at den or room entrance
- Does NOT remember chase (data was lost)
- Sees player again → new Chase state initiated
```

**Result**: Lizard behavior feels continuous but isn't perfectly persistent. Player perception: "The lizard I fled from might still be around."

## Creature State and Memory

### Abstract Creature State

`AbstractCreature` tracks:
```csharp
- ID: EntityID (unique identifier)
- pos: WorldCoordinate (which room, which node)
- state: CreatureState
  - health: float
  - dead: bool
  - meatLeft: int (if dead)
- personality: CreaturePersonality (aggression, bravery, energy)
- abstractAI: AbstractCreatureAI
  - hunger: float
  - denPosition: WorldCoordinate
  - migration: float (willingness to move)
```

### Realized Creature State

`Creature` (PhysicalObject) has:
```csharp
- abstractCreature: AbstractCreature (link back)
- bodyChunks: BodyChunk[] (physics)
- AI: ArtificialIntelligence
  - behavior: AIModule (pathfinding, states)
  - tracker: CreatureTracker (sensory memory)
  - threatTracker: ThreatTracker (danger assessment)
  - preyTracker: PreyTracker (food targets)
- animation: float (state, counters)
- graphicsModule: CreatureGraphics (if visible)
```

### Memory Systems

**Abstract Memory** (persistent):
- Den location (home)
- General threat levels (dangerous rooms)
- Pack affiliations
- Long-term hunger trends

**Realized Memory** (transient):
- Last seen player position
- Sound source locations
- Recent path history
- Combat target

**Why separate?** Abstract memory must be compact (hundreds of creatures). Realized memory can be detailed (only ~15 creatures).

## Performance Budget

### Abstract AI Budget

Per creature, per update (~every 4 frames):
- **Migration check**: 0.01ms
- **Hunger update**: 0.005ms
- **Interaction check**: 0.02ms (if other creatures in room)

**Total**: ~0.03ms per creature every 4 frames = 0.0075ms per frame

With 200 abstract creatures: ~1.5ms per frame

### Realized AI Budget

Per creature, per frame:
- **Sensory processing**: 0.2ms (raycasts, sound)
- **Behavior state evaluation**: 0.1ms
- **Pathfinding**: 0.3ms (amortized, only every few frames)
- **Decision making**: 0.1ms

**Total**: ~0.7ms per creature per frame

With 15 realized creatures: ~10.5ms per frame

**Frame budget at 40 TPS**: 25ms total
- Abstract AI: ~1.5ms (6%)
- Realized AI: ~10.5ms (42%)
- Physics, graphics, other: ~13ms (52%)

## Design Tradeoffs

| Decision | Benefit | Cost |
|----------|---------|------|
| **Dual AI system** | Simulate entire ecosystem | Complexity, state loss on transitions |
| **Staggered abstract updates** | Spread CPU load, handle 200+ creatures | Less responsive (but player can't see) |
| **Simplified abstract combat** | Fast, scalable | Less detailed than realized combat |
| **State loss on abstraction** | Memory efficient | Can break immersion if noticeable |
| **Every-frame realized AI** | Responsive, tactical gameplay | Limited to ~15 creatures |

## What This Enables

**Persistent ecosystem**:
- Lizard you flee from continues existing
- May be in same room when you return
- Or may have migrated hunting for food

**Emergent narratives**:
- Find scavenger corpses (abstract combat occurred)
- Encounter creatures mid-migration
- Discover territorial changes

**Fair gameplay**:
- Creatures don't spawn just for you
- They exist and behave whether you're watching or not
- Deterministic: same seed = same ecosystem

**Performance**:
- 200+ creatures across region
- Only 10-20 fully simulated at once
- 70+ rooms with living population

## Related Documentation

**Same layer**:
- [Dual LOD System](../core-architecture/dual-lod-system.md) - How abstract/realized works for all entities
- [Update Loop](../core-architecture/update-loop.md) - When AI updates execute
- [Abstract AI](abstract-ai.md) - Deep dive on abstract decision-making
- [Realized AI](realized-ai.md) - Deep dive on tactical behavior
- [Creature State](creature-state.md) - State persistence and memory

**Go deeper**:
- [Layer 2: AI State Machines](../../02-implementation/algorithms/ai-state-machines.md) - Behavior state implementation
- [Layer 2: Pathfinding](../../02-implementation/algorithms/pathfinding.md) - A* and room-graph pathfinding
- [Layer 2: Creature Implementation](../../02-implementation/systems-implementation/creature-impl.md) - Creature class internals

**Related topics**:
- [Entity Lifecycle](../entity-systems/entity-lifecycle.md) - Spawn, update, destroy cycle
- [LOD Transitions](../entity-systems/lod-transitions.md) - Realization and abstraction details

## Key Takeaways

1. **Dual AI**: Creatures use abstract AI (strategic) when far, realized AI (tactical) when near
2. **Performance ratio**: Abstract AI is ~50-70× cheaper than realized AI
3. **State loss**: Data not explicitly saved is lost on abstraction (intentional tradeoff)
4. **Staggered updates**: Abstract AI updates every 3-5 frames to spread CPU load
5. **Ecosystem simulation**: Abstract AI enables persistent world with hundreds of creatures
6. **Seamless transitions**: Player perceives continuous behavior despite AI mode switches
