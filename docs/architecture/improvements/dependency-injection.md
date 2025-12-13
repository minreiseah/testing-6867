# Improvement: Dependency Injection

**Status**: Proposal
**Date**: 2025-12-14
**Effort**: Medium-High
**Related**: [Singleton Coupling Problem](../problems/singleton-coupling.md)

## Summary

Replace singleton pattern with explicit dependency passing. Classes receive dependencies through constructors instead of reaching out to global state.

## What is Dependency Injection?

**Current (Singleton)**:
```csharp
class Lizard {
    void Update() {
        // Reaches out to global state
        var volume = game.rainWorld.options.soundVolume;
        var threats = game.rainWorld.threatTracker.GetThreats();
    }
}
```

**With DI**:
```csharp
class Lizard {
    private Options options;
    private ThreatTracker threatTracker;

    // Dependencies provided at construction
    Lizard(Options options, ThreatTracker threatTracker) {
        this.options = options;
        this.threatTracker = threatTracker;
    }

    void Update() {
        // Uses injected dependencies
        var volume = options.soundVolume;
        var threats = threatTracker.GetThreats();
    }
}
```

**Key difference**: Dependencies are **explicit** (visible in constructor) vs **implicit** (hidden in implementation).

## Patterns for Rain World

### 1. Constructor Injection

**For most classes**:
```csharp
class Creature {
    private World world;
    private Options options;
    private SoundLoader soundLoader;

    Creature(World world, Options options, SoundLoader soundLoader) {
        this.world = world;
        this.options = options;
        this.soundLoader = soundLoader;
    }
}
```

**Benefits**:
- Dependencies visible in signature
- Immutable (set once)
- Easy to test (pass mocks)

**Drawback**: Constructor parameters can get long.

### 2. Service Locator (Lighter Alternative)

**For complex dependency graphs**:
```csharp
class Services {
    public World world;
    public Options options;
    public SoundLoader soundLoader;
    public ThreatTracker threatTracker;
    public CreatureCommunities communities;
    // ... etc
}

class Creature {
    private Services services;

    Creature(Services services) {
        this.services = services;
    }

    void Update() {
        var volume = services.options.soundVolume;
    }
}
```

**Benefits**:
- Single parameter
- Still explicit (Services in constructor)
- Easy to add new services

**Drawback**: Less explicit about which services used.

**Verdict**: Service Locator is good compromise for Rain World's complexity.

### 3. Property Injection (For Optional Dependencies)

```csharp
class Creature {
    public DebugLogger Logger { get; set; } // Optional

    void Update() {
        Logger?.Log("Updating creature");
    }
}
```

Use sparingly. Constructor injection preferred.

## Refactoring Rain World

### Current Structure

```csharp
// Singleton pattern
class RainWorld {
    static RainWorld instance;
}

class Creature {
    void Update() {
        game.rainWorld.options.soundVolume
    }
}
```

### After DI: Service Locator Pattern

```csharp
// Services container
class GameServices {
    public World World { get; }
    public Options Options { get; }
    public SoundLoader SoundLoader { get; }
    public ThreatTracker ThreatTracker { get; }
    public CreatureCommunities Communities { get; }

    public GameServices(World world, Options options, ...) {
        World = world;
        Options = options;
        // ... etc
    }
}

// Creatures receive services
class Creature {
    protected GameServices services;

    Creature(GameServices services) {
        this.services = services;
    }

    void Update() {
        var volume = services.Options.SoundVolume;
    }
}

// At creation site
var services = new GameServices(world, options, soundLoader, ...);
var lizard = new Lizard(services);
```

### Creation Flow

```csharp
// RainWorldGame.SetupWorld()
void SetupWorld() {
    var options = LoadOptions();
    var soundLoader = new SoundLoader();
    var world = new World();

    var services = new GameServices(
        world: world,
        options: options,
        soundLoader: soundLoader,
        threatTracker: new ThreatTracker(),
        communities: new CreatureCommunities()
    );

    // All creatures get services
    foreach (var room in world.rooms) {
        room.SpawnCreatures(services);
    }
}

class Room {
    void SpawnCreatures(GameServices services) {
        var lizard = new Lizard(services);
        AddObject(lizard);
    }
}
```

## Benefits for Rain World

### 1. Testability

**Current**: Must initialize entire RainWorld singleton
```csharp
[Test]
void LizardFlees_WhenPlayerNearby() {
    // 50+ lines of setup
    var rainWorld = new RainWorld();
    rainWorld.LoadAssets();
    // ...

    var lizard = new Lizard();
    lizard.Update();
}
```

**With DI**: Mock only what's needed
```csharp
[Test]
void LizardFlees_WhenPlayerNearby() {
    var mockOptions = new MockOptions();
    var mockTracker = new MockThreatTracker();
    var services = new GameServices(
        world: null, // Not needed for this test
        options: mockOptions,
        soundLoader: null,
        threatTracker: mockTracker,
        communities: null
    );

    var lizard = new Lizard(services);
    lizard.Update();

    Assert.True(lizard.isFleeing);
}
```

**5 lines instead of 50.**

### 2. Explicit Dependencies

**Constructor reveals scope**:
```csharp
class Lizard : Creature {
    Lizard(GameServices services) { ... }
}
```

Code reviewer sees: "Lizard depends on GameServices". No hidden dependencies.

**Compare to**:
```csharp
class Lizard : Creature {
    void Update() {
        // Hidden dependency only visible in implementation
        game.rainWorld.threatTracker...
    }
}
```

### 3. Modding

**Current**: Mods patch global singleton
```csharp
// Mod A adds state
RainWorld.instance.modAState = new ModAState();

// Mod B adds state
RainWorld.instance.modBState = new ModBState();

// Conflicts if both touch same systems
```

**With DI**: Mods inject custom services
```csharp
// Mod A
class ModAServices : GameServices {
    public ModAState ModAState { get; }

    ModAServices(..., ModAState modAState) {
        ModAState = modAState;
    }
}

var modServices = new ModAServices(..., new ModAState());
var lizard = new Lizard(modServices);
```

**No global state pollution.** Each mod can have its own service instance.

### 4. Lifecycle Control

**Current**: Singleton lives forever
```csharp
RainWorld.instance // Always exists, shared everywhere
```

**With DI**: Services scoped to game session
```csharp
class GameSession {
    private GameServices services;

    void Start() {
        services = new GameServices(...);
        // Session-specific
    }

    void End() {
        services = null; // Cleaned up
    }
}
```

Different sessions can have different configurations.

### 5. Parallel Worlds

**Current**: Impossible (one singleton)
```csharp
var world1 = new World();
var world2 = new World();
// Both access RainWorld.instance (same one!)
```

**With DI**: Multiple service instances
```csharp
var services1 = new GameServices(world1, ...);
var services2 = new GameServices(world2, ...);

world1.Update(services1);
world2.Update(services2);
```

**Use cases**:
- Split-screen multiplayer
- Simulation comparison tools
- Parallel region loading
- Test fixtures

### 6. Easier Refactoring

**Want to change ThreatTracker implementation?**

**Current**: Find every `game.rainWorld.threatTracker` reference
```csharp
// Scattered across hundreds of files
game.rainWorld.threatTracker.GetThreats()
```

**With DI**: Change service registration
```csharp
var services = new GameServices(
    threatTracker: new ImprovedThreatTracker() // Swap implementation
);
```

All classes using ThreatTracker automatically use new version.

## Tradeoffs

### Verbosity

❌ **Con**: Longer constructors
```csharp
Creature(GameServices services) // OK
Creature(World world, Options options, SoundLoader soundLoader,
         ThreatTracker tracker, CreatureCommunities communities) // Verbose
```

✅ **Mitigation**: Use Service Locator pattern (single `GameServices` parameter)

### Refactoring Cost

❌ **Con**: Massive codebase change
- Thousands of `game.rainWorld` references
- Every class that uses singleton
- All object creation sites

✅ **Mitigation**: Incremental refactoring
1. Create `GameServices` class
2. Add services field to base classes
3. Gradually convert classes to use services
4. Keep singleton during transition (adapter pattern)

**Example adapter**:
```csharp
class Creature {
    protected GameServices services;

    Creature(GameServices services = null) {
        // During transition: fallback to singleton
        this.services = services ?? new GameServices(
            world: game.World,
            options: game.rainWorld.options,
            // ... etc
        );
    }
}
```

Old code continues working, new code passes services.

### Not All Classes Need It

❌ **Con**: Over-engineering risk

**Good candidates for DI**:
- Creatures (need world, options, sound)
- AI systems (need tracker, communities)
- Physics (need options, world)

**Bad candidates**:
- Utility classes (Math, String helpers)
- Pure data structures (Vector2, Color)
- Stateless functions

Don't inject dependencies into classes that don't have dependencies.

### Learning Curve

❌ **Con**: Team must understand DI
- New pattern for developers
- More upfront design

✅ **Pro**: Industry-standard pattern
- Well-documented
- Common in modern game engines (Unity, Unreal)

## Real-World Examples

### Unity
```csharp
class PlayerController : MonoBehaviour {
    [Inject] private GameSettings settings;
    [Inject] private AudioManager audio;

    // Zenject DI framework
}
```

### ASP.NET
```csharp
class Controller {
    private IDatabase database;

    Controller(IDatabase database) {
        this.database = database;
    }
}
```

### Unreal Engine
```cpp
class AMyActor : AActor {
    UPROPERTY()
    UGameSettings* Settings;

    // Injected by engine
}
```

**DI is standard practice in software engineering.**

## Implementation Path

### Phase 1: Create Services (1 week)
```csharp
class GameServices {
    public World World { get; }
    public Options Options { get; }
    // ... etc
}
```

### Phase 2: Adapter Pattern (1 week)
```csharp
class Creature {
    protected GameServices services;

    Creature(GameServices services = null) {
        this.services = services ?? GetServicesFromSingleton();
    }
}
```

Backward compatible.

### Phase 3: Incremental Conversion (3-6 months)
- Convert base classes (Creature, PhysicalObject)
- Convert systems (AI, Physics)
- Convert UI and menus
- Remove fallbacks

### Phase 4: Remove Singleton (1 week)
- Delete RainWorld singleton
- All code now uses services
- Break builds to find remaining references

**Total**: 4-8 months for full migration.

## Combining with ECS

DI works great with ECS:
```csharp
class PhysicsSystem {
    private GameServices services;

    PhysicsSystem(GameServices services) {
        this.services = services;
    }

    void Update(EntityManager em) {
        var gravity = services.Options.Gravity;
        // ...
    }
}
```

Systems receive services, process entities. Clean separation.

## Verdict

**Should Rain World migrate to DI?**

**For shipped game**: Medium priority. Enables better modding and testing.

**For sequel**: High priority. Start with DI from day one.

**Incremental approach**: ✅ **Yes**
- Add GameServices
- Convert high-value systems (AI, creatures)
- Leave low-priority code as-is
- Provides path forward without full rewrite

## Related Documents

- [Singleton Coupling Problem](../problems/singleton-coupling.md)
- [Architecture Overview](../overview.md)
- [ECS Migration Proposal](ecs-migration.md)
- [2025-12-14 Analysis](../analysis/2025-12-14-review.md)
