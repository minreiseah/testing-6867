# Problem: Singleton Coupling

**Status**: Identified
**Date**: 2025-12-14
**Severity**: High
**Related**: [Dependency Injection Proposal](../improvements/dependency-injection.md)

## The Pattern

Rain World uses a singleton pattern where the `RainWorld` object is globally accessible throughout the codebase.

```csharp
class RainWorld {
    static RainWorld instance;

    // Global access
    Options options;
    SoundLoader soundLoader;
    ProcessManager processManager;
    // ... dozens more
}
```

Classes access it through chains like:
```csharp
game.rainWorld.options.soundVolume
room.game.rainWorld.soundLoader
abstractCreature.world.game.rainWorld
```

## Concrete Problems

### 1. Hidden Dependencies

**Symptom**: Method signatures don't reveal what they need.

```csharp
class Lizard : Creature {
    void UpdateAI() {
        // Where did 'game' come from? Not in parameters!
        var threatLevel = game.rainWorld.threatTracker.GetThreat(this);
        var soundVolume = game.rainWorld.options.soundVolume;
    }
}
```

Looking at `UpdateAI()` tells you nothing about dependencies. Must read implementation to discover it touches:
- ThreatTracker
- Options
- Potentially more

**Impact**:
- Code review is harder
- Refactoring is risky
- Understanding scope requires reading entire method

### 2. Tight Coupling

Every class that uses the singleton is coupled to:
- RainWorld's entire API surface
- Its initialization order
- Its lifecycle
- All subsystems attached to it

**Example**: Want to change how Options initializes?

```csharp
// Must find every place that does this
game.rainWorld.options.someValue

// Across potentially hundreds of files:
// - Creature AI
// - Graphics modules
// - Menu systems
// - Physics code
// - Sound system
```

All coupled to `RainWorld.options`. Changing it requires auditing the entire codebase.

### 3. Testing Nightmare

To unit test a single creature behavior:

```csharp
[Test]
void LizardFlees_WhenPlayerNearby() {
    var lizard = new Lizard(...);
    lizard.UpdateAI();
    Assert.True(lizard.isFleeing);
}
```

**This fails** because:
1. Lizard.UpdateAI() accesses `game.rainWorld`
2. Must initialize entire RainWorld singleton
3. RainWorld requires ProcessManager
4. ProcessManager requires loaded game
5. Game requires World
6. World requires loaded rooms
7. Rooms require asset loading
8. Assets require file system access

**Actual test setup**:
```csharp
[Test]
void LizardFlees_WhenPlayerNearby() {
    // Initialize entire game engine
    var rainWorld = new RainWorld();
    rainWorld.LoadAssets();
    rainWorld.InitializeOptions();
    rainWorld.InitializeSoundSystem();
    // ... 50 more lines

    var lizard = new Lizard(...);
    lizard.UpdateAI();
    Assert.True(lizard.isFleeing);
}
```

Test becomes so expensive developers skip testing entirely.

### 4. Mod Conflicts

Multiple mods patching methods that access the singleton:

```csharp
// Mod A patches Creature.Update
void Update_ModA(Creature self) {
    game.rainWorld.modACustomState.DoThing();
    // original update
}

// Mod B patches Creature.Update
void Update_ModB(Creature self) {
    game.rainWorld.modBCustomState.DoOtherThing();
    // original update
}
```

**Problems**:
- Both mods mutating shared global state
- Initialization order matters (which mod loads first?)
- Race conditions if mods touch same subsystems
- Singleton becomes dumping ground for mod data

**Real Impact**: Popular mods become incompatible not because of feature conflicts, but because of singleton pollution.

### 5. Initialization Order Dependencies

```csharp
class CreatureAI {
    CreatureAI() {
        // Crashes if threatTracker hasn't initialized yet
        var threats = game.rainWorld.threatTracker.GetThreats();
    }
}
```

Singletons create hidden initialization requirements:
- RainWorld must initialize in specific order
- Subsystems have implicit dependencies
- Order is undocumented and fragile

**Example failure**:
1. Mod adds new creature type
2. Creature spawns during world load
3. CreatureAI tries to access threatTracker
4. ThreatTracker not initialized yet
5. Null reference exception

This fails **only in specific scenarios** (race condition based on load order).

### 6. Parallel Processing Impossible

Want to simulate two worlds in parallel?

```csharp
var world1 = new World();
var world2 = new World();

// Both access same singleton
world1.Update(); // Uses RainWorld.instance
world2.Update(); // Uses RainWorld.instance (same one!)
```

Impossible with singleton. All worlds share state.

**Use cases blocked**:
- Split-screen multiplayer
- Simulation comparison tools
- Parallel region loading
- Test fixtures running in parallel

## Real Impact in Rain World

Based on the wiki documentation:

**Dependency Chain**:
```
RainWorld (singleton)
  ↑
Game
  ↑
World
  ↑
Room
  ↑
Creature
  ↑
AI, Graphics, Physics
```

Low-level AI code knows about high-level game management. Everything is coupled to everything.

**Mod Ecosystem Impact**:
- Most mods patch into singleton
- Common pattern: Add custom state to RainWorld
- Mods frequently conflict
- Community maintains compatibility spreadsheets

## Why It Happened

**Historical reasons**:
1. **Convenience**: Quick access from anywhere
2. **Rapid development**: Avoids threading dependencies through constructors
3. **Single-player game**: Only ever one world active
4. **No test culture**: Testing wasn't priority

These are valid reasons for small projects, but technical debt for larger codebases.

## Measurement

**Coupling metrics** (if we had access to source):
- Classes directly referencing RainWorld singleton: Likely 100+
- Transitive dependencies on RainWorld: Likely 500+
- Average dependency chain length: 4-5 levels

## Solutions

See: [Dependency Injection Proposal](../improvements/dependency-injection.md)

**Summary**:
- Pass dependencies explicitly
- Use service locator for shared services
- Decouple subsystems from global state

## Related Documents

- [Architecture Overview](../overview.md)
- [Dependency Injection Proposal](../improvements/dependency-injection.md)
- [2025-12-14 Analysis](../analysis/2025-12-14-review.md)
