# Class Hierarchy Reference

Visual map of Rain World's class hierarchies and inheritance relationships.

## Game Lifecycle Hierarchy

```
RainWorld (MonoBehaviour, singleton)
  └─ ProcessManager
      └─ MainLoopProcess (abstract base)
          ├─ Menu.Menu (base for menu screens)
          │   ├─ SlugcatSelectMenu
          │   ├─ SaveGameMenu
          │   ├─ OptionsMenu
          │   └─ ... (many menu types)
          │
          ├─ RainWorldGame (gameplay session)
          │   ├─ GameSession
          │   │   ├─ StoryGameSession
          │   │   └─ ArenaGameSession
          │   │
          │   └─ World (current region)
          │       ├─ AbstractRoom[] (all rooms)
          │       └─ Room[] (realized rooms)
          │
          ├─ Menu.MusicPlayer (persistent music)
          ├─ RainWorldSteamManager (achievements)
          └─ MenuMicrophone (menu sound effects)
```

## Abstract Layer Hierarchy

Everything far from the player exists in abstract form:

```
AbstractWorldEntity (base class for all abstract entities)
  │
  └─ AbstractPhysicalObject (can realize to PhysicalObject)
      │
      ├─ AbstractCreature (creatures in abstract form)
      │   │  • Has: AbstractCreatureAI
      │   │  • Has: CreatureState (personality, hunger, health)
      │   │  • Realizes to: Creature
      │   │
      │   ├─ AbstractCreature (player slugcats)
      │   ├─ AbstractCreature (lizards)
      │   ├─ AbstractCreature (scavengers)
      │   └─ AbstractCreature (all creature types)
      │
      └─ AbstractPhysicalObject (items in abstract form)
          │  • Realizes to: PhysicalObject subclasses
          │
          ├─ AbstractSpear → Spear
          ├─ AbstractRock → Rock
          ├─ AbstractBatFly → Batfly
          └─ ... (all item types)
```

**Key distinction**:
- `AbstractWorldEntity`: Never realizes (pure abstract)
- `AbstractPhysicalObject`: Can realize to PhysicalObject when room becomes active

## Realized Layer Hierarchy

Everything near the player exists in realized form:

```
UpdatableAndDeletable (base class for all realized entities)
  │
  ├─ PhysicalObject (has physics/collision)
  │   │  • Has: BodyChunk[] (physics circles)
  │   │  • Has: GraphicsModule (optional, for rendering)
  │   │  • Attached to: AbstractPhysicalObject
  │   │
  │   ├─ Creature (living entities)
  │   │   │  • Has: ArtificialIntelligence (detailed AI)
  │   │   │  • Attached to: AbstractCreature
  │   │   │
  │   │   ├─ Player (slugcat)
  │   │   │   • Graphics: PlayerGraphics
  │   │   │
  │   │   ├─ Lizard
  │   │   │   • AI: LizardAI
  │   │   │   • Graphics: LizardGraphics
  │   │   │   • Subtypes: PinkLizard, GreenLizard, BlueLizard, etc.
  │   │   │
  │   │   ├─ Scavenger
  │   │   │   • AI: ScavengerAI
  │   │   │   • Graphics: ScavengerGraphics
  │   │   │
  │   │   ├─ Vulture
  │   │   │   • AI: VultureAI
  │   │   │   • Graphics: VultureGraphics
  │   │   │
  │   │   └─ ... (dozens of creature types)
  │   │
  │   ├─ Items (throwable/carriable objects)
  │   │   ├─ Weapon
  │   │   │   ├─ Spear
  │   │   │   ├─ Rock
  │   │   │   ├─ ScavengerBomb
  │   │   │   └─ ... (weapon types)
  │   │   │
  │   │   ├─ PlayerCarryableItem
  │   │   │   ├─ Batfly
  │   │   │   ├─ SlimeMold (food)
  │   │   │   ├─ DataPearl
  │   │   │   └─ ... (carriable items)
  │   │   │
  │   │   └─ ...
  │   │
  │   └─ Environmental objects
  │       ├─ SporePlant
  │       ├─ PoleMimic
  │       ├─ TubeWorm
  │       └─ ...
  │
  └─ Non-physical entities (no collision)
      ├─ Water (water simulation)
      ├─ WaterDrip (dripping water effects)
      ├─ Smoke (smoke particles)
      ├─ LightSource (dynamic lights)
      └─ ... (visual/audio effects)
```

## AI Hierarchy

### Abstract AI (for distant creatures)

```
AbstractCreatureAI (simple state machine)
  │
  ├─ AbstractCreatureAI (generic implementation)
  ├─ ScavengerAbstractAI (scavenger-specific)
  ├─ VultureAbstractAI (vulture-specific)
  └─ ... (specialized abstract AI per creature type)
```

**What Abstract AI handles**:
- Room-to-room migration
- Abstract combat/predation
- Hunger and fear tracking

### Realized AI (for nearby creatures)

```
ArtificialIntelligence (detailed behavior)
  │
  ├─ LizardAI
  │   • Pathfinding
  │   • Behavior states (hunt, chase, attack, flee, idle)
  │   • Sensory processing (sight, hearing, smell)
  │
  ├─ ScavengerAI
  │   • Social behavior (squad tactics)
  │   • Trading mechanics
  │   • Patrol routes
  │
  ├─ VultureAI
  │   • Flying pathfinding
  │   • Dive attacks
  │   • Circling behavior
  │
  └─ ... (one AI class per intelligent creature type)
```

**What Realized AI handles**:
- Frame-by-frame decision making
- Pathfinding through terrain
- Combat tactics
- Animation triggers

## Graphics Hierarchy

```
GraphicsModule (rendering logic, separate from game logic)
  │
  ├─ PlayerGraphics
  │   • Slugcat rendering
  │   • Animation blending (walk, crawl, roll, etc.)
  │   • Tail simulation
  │
  ├─ LizardGraphics
  │   • Body segment rendering
  │   • Color variations
  │   • Tongue animation
  │
  ├─ ScavengerGraphics
  │   • Masked rendering
  │   • Gear/items rendering
  │   • Squad color markings
  │
  └─ ... (one graphics module per creature/object type with visuals)
```

**Important**: Not all PhysicalObjects have GraphicsModules. Simple objects (rocks, spears) render directly without a separate graphics module.

## World/Room Hierarchy

```
World (current region)
  │
  ├─ AbstractRoom (lightweight, always exists)
  │   • connections[] (which rooms connect)
  │   • entities[] (AbstractWorldEntity list)
  │   • realizedRoom (null when not active)
  │
  ├─ Room (realized geometry)
  │   • Tiles[][] (terrain grid)
  │   • aimap (pathfinding data)
  │   • physicalObjects[][] (realized entities)
  │   • roomSettings (lighting, effects)
  │
  ├─ RoomCamera (rendering camera)
  │   • Follows player
  │   • Screen shake
  │   • Camera effects
  │
  └─ RoomPreparer (background loader)
      • Loads next room while player is in current room
```

## Save System Hierarchy

```
RainWorld
  └─ progression (PlayerProgression)
      │
      ├─ currentSaveState (SaveState)
      │   • Player position
      │   • Karma level
      │   • Unlocked regions
      │   • Passage completion
      │
      ├─ miscProgressionData (MiscProgressionData)
      │   • Achievements
      │   • Discoveries
      │   • Story flags
      │
      └─ rainWorld.options (Options)
          • User settings
          • Input configuration
```

**Persistence**:
- `SaveState`: Per-save-slot data (player progress)
- `RegionState`: Per-region, per-cycle data (creature spawns, deaths, migrations)
- `Options`: Global user preferences

## Helper Systems

### BodyChunk (Physics)

```
PhysicalObject
  └─ BodyChunk[] (array of physics circles)
      • pos (position)
      • vel (velocity)
      • mass (mass)
      • rad (radius)
      • collideWithTerrain (bool)
      • collideWithObjects (bool)
```

Connected via:
```
BodyChunkConnection[] (spring connections between chunks)
  • type (normal, pull, push, rope)
  • distance (rest length)
  • elasticity (spring constant)
```

### Collision Layers

PhysicalObjects exist on one of three collision layers:

- **Layer 0**: Most creatures and objects
- **Layer 1**: Small items, some creatures
- **Layer 2**: Special cases (background objects)

Objects **only collide** with others on the same layer, drastically reducing collision checks.

## Common Modding Entry Points

### Creating a New Creature

1. **Subclass AbstractCreature**:
   ```csharp
   public class AbstractMyCreature : AbstractCreature { ... }
   ```

2. **Subclass Creature**:
   ```csharp
   public class MyCreature : Creature { ... }
   ```

3. **Implement ArtificialIntelligence**:
   ```csharp
   public class MyCreatureAI : ArtificialIntelligence { ... }
   ```

4. **Implement GraphicsModule** (optional):
   ```csharp
   public class MyCreatureGraphics : GraphicsModule { ... }
   ```

5. **Hook into spawn system** (via modding API)

### Creating a New Item

1. **Subclass AbstractPhysicalObject**:
   ```csharp
   public class AbstractMyItem : AbstractPhysicalObject { ... }
   ```

2. **Subclass PhysicalObject** (or PlayerCarryableItem, Weapon, etc.):
   ```csharp
   public class MyItem : PlayerCarryableItem { ... }
   ```

3. **Hook into spawn system**

### Hooking Into Existing Classes

Rain World's modding framework (BepInEx) allows hooks:

```csharp
On.Player.Update += (orig, self) => {
    orig(self); // Call original
    // Your custom logic
};
```

Common hook points:
- `Player.Update()`: Player behavior modifications
- `Creature.Update()`: Creature behavior
- `Room.Update()`: Room-level logic
- `RainWorldGame.Update()`: Global game logic

## Quick Class Lookup

**"I want to modify..."** | **Hook into...**
--- | ---
Player movement | `Player.Update()`, `Player.MovementUpdate()`
Creature AI | `ArtificialIntelligence.Update()` or specific AI class
Graphics/rendering | `GraphicsModule.DrawSprites()`
Room geometry | `Room.Loaded()` (when room first loads)
World/ecosystem | `World.Update()`, `AbstractRoom.Update()`
Save data | `SaveState`, `RegionState`
Physics | `PhysicalObject.Update()`, `BodyChunk`

## Key Interfaces

```
IPlayerEdible
  • Can be eaten by player

IDrawable
  • Has custom rendering logic

IProvideWarmth
  • Provides warmth (fire, lantern, etc.)

IOwnARelationshipTracker
  • Tracks relationships with other creatures

IHaveAppendages
  • Has appendages (tentacles, wings, etc.)
```

## Related Documentation

- [Architecture Overview](overview.md)
- [Dual LOD System](dual-lod.md)
- [World & Room Management](world-rooms.md)
