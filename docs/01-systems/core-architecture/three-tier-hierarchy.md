# Three-Tier Hierarchy

**Layer**: 1 (Systems Architecture)
**Reading time**: 15 minutes

## Purpose

This document explains Rain World's organizational structure: a three-level hierarchy from RainWorld singleton → ProcessManager state machine → MainLoopProcess game states.

## The Hierarchy

```
RainWorld (singleton, lifecycle manager)
  └─ ProcessManager (state machine)
      └─ MainLoopProcess (current game state)
          ├─ MenuProcess (various menus)
          └─ RainWorldGame (actual gameplay)
              └─ World (current region)
                  └─ Rooms → Entities
```

Each level has a specific responsibility:
- **Level 1 (RainWorld)**: Game lifecycle and global resources
- **Level 2 (ProcessManager)**: High-level state management
- **Level 3 (MainLoopProcess)**: Specific game mode logic

## Level 1: RainWorld Singleton

### What It Is

`RainWorld` is a Unity MonoBehaviour that persists for the entire game session. It's the root object that Unity calls, and everything else hangs off it.

**Lifetime**: Application start → Application quit
**Instance count**: Exactly one
**Update cycle**: Every render frame (Unity's Update())

### Responsibilities

**1. Lifecycle Management**:
- Initialize on game start
- Manage ProcessManager
- Clean up on application quit
- Handle scene transitions

**2. Global Resources**:
```csharp
RainWorld {
    options: Options                    // Player settings
    processManager: ProcessManager      // Current game state
    soundSystem: SoundLoader            // Audio management
    setup: RainWorldSetup              // Configuration data
    shaders: Dictionary<string, FShader> // Graphics shaders
}
```

**3. Modding Hooks**:
- `OnModsInit()` - Called when mods load
- `PreModsEnabled()` - Before mod activation
- `PostModsEnabled()` - After mod activation

**4. Resource Loading**:
- Load/unload shaders
- Initialize sound system
- Load configuration files
- Prepare asset bundles

### Why a Singleton?

**Pros**:
- Convenient global access: `game.rainWorld.options`
- Simple Unity integration
- No dependency injection needed

**Cons**:
- Tight coupling throughout codebase
- Hard to test (must initialize entire engine)
- Global state makes mod conflicts likely

See [Appendix: Singleton Coupling Problem](../../appendix/problems/singleton-coupling.md) for detailed analysis.

### Update Routing

```csharp
// Unity calls this every render frame
RainWorld.Update() {
    // Route to ProcessManager
    processManager.Update();
}
```

RainWorld does minimal work itself—it primarily delegates to ProcessManager.

## Level 2: ProcessManager (State Machine)

### What It Is

ProcessManager implements a **state machine** for high-level game modes. At any moment, exactly one MainLoopProcess is active.

**Lifetime**: Created by RainWorld, persists for game session
**Instance count**: One per RainWorld
**Update cycle**: Called by RainWorld.Update()

### States (MainLoopProcess Types)

```
MainLoopProcess (abstract base class)
├─ MenuProcess
│   ├─ MainMenuProcess
│   ├─ PauseMenuProcess
│   ├─ OptionsMenuProcess
│   └─ ...
├─ SlideShowMenuProcess (cutscenes)
├─ FastTravelScreen
└─ RainWorldGame (gameplay)
    ├─ StoryGameMode
    └─ ArenaGameMode
```

Each state is a self-contained game mode with its own logic.

### State Transitions

**Transition triggers**:
- User input (start game, pause, quit)
- Game events (death, sleep, win)
- Programmatic requests

**Transition process**:
```
1. currentProcess.ShutDown() called
2. Fade effect begins
3. New process constructed
4. newProcess.CommunicateWithUpcoming(oldProcess)
   (optional data transfer)
5. Old process destroyed
6. currentProcess = newProcess
7. Fade completes
```

**Clean state**: Each transition fully tears down the old process. No state bleeds between modes.

### Responsibilities

**1. Update Routing**:
```csharp
ProcessManager.Update() {
    currentMainLoop.Update();     // 40 TPS logic
    currentMainLoop.RawUpdate();  // 60 FPS input
    currentMainLoop.GrafUpdate(); // 60 FPS graphics
}
```

**2. Frame Timing**:
- Maintains target 40 FPS for game logic
- Separates logic (40 TPS) from rendering (60 FPS)
- Handles frame accumulation and delta time

**3. Subprocess Management**:
- SideProcesses: Background tasks (music player)
- DialogBoxes: UI overlays
- Fade effects during transitions

**4. Input Handling**:
```csharp
ProcessManager.Update() {
    if (escape key pressed && !in menu) {
        RequestMainProcessSwitch(new PauseMenu());
    }
}
```

### Why a State Machine?

**Pros**:
- Clean separation between game modes
- No `if (inMenu) ... else if (inGame)` scattered everywhere
- Each mode is self-contained and testable
- Easy to add new modes

**Cons**:
- Teardown/setup overhead on transitions
- Can't easily share state between modes (by design)
- Full restart required for significant changes

## Level 3: MainLoopProcess (Game Mode)

### What It Is

Abstract base class for all game modes. Each mode implements its own update logic.

**Lifetime**: Created during state transition, destroyed on next transition
**Instance count**: One active at a time
**Update cycle**: Called by ProcessManager

### Key Subclasses

#### RainWorldGame (Gameplay)

The most important MainLoopProcess:
```csharp
RainWorldGame {
    world: World                     // Current region
    cameras: RoomCamera[]            // What player sees
    session: GameSession             // Story/Arena mode logic
    globalRain: GlobalRain           // Rain cycle timer
    Players: Player[]                // Player entities
}
```

**Manages**:
- The living world (creatures, rooms, ecosystem)
- Player entities and cameras
- Rain cycle timer (the deadline)
- Game session (story progress, arena rules)

**Lifecycle**:
```
Created: When starting game or entering new region
Updated: Every frame during gameplay
Destroyed: On death, sleep, or quit to menu

IMPORTANT: Sleeping creates a NEW RainWorldGame
(ensures clean state between cycles)
```

#### MenuProcess (UI)

Various menu screens:
- MainMenu: Title screen, save file selection
- PauseMenu: In-game pause
- OptionsMenu: Settings
- SleepScreen: End-of-cycle summary

**Simpler** than RainWorldGame:
- No World, no creatures, no physics
- Just UI elements and input handling
- Lightweight update loop

### Update Methods

All MainLoopProcess subclasses implement:

**Update()** (40 TPS, deterministic):
```csharp
void Update() {
    // Game logic
    // Physics simulation
    // AI decisions
    // State changes
}
```

**RawUpdate()** (60 FPS, variable):
```csharp
void RawUpdate() {
    // Input processing
    // Immediate UI response
    // Frame-rate dependent updates
}
```

**GrafUpdate()** (60 FPS, rendering):
```csharp
void GrafUpdate() {
    // Graphics interpolation
    // Camera updates
    // Sprite positioning
}
```

## Data Flow Through Hierarchy

### Input Flow (Bottom-Up)

```
Unity Input System
    ↓
RainWorld.Update() captures input
    ↓
ProcessManager.RawUpdate() processes input
    ↓
currentMainLoop.RawUpdate() handles input
    ↓
(if RainWorldGame)
    Player.Update() receives input
        ↓
    BodyChunks apply forces
```

### Logic Flow (Top-Down)

```
RainWorld.Update() @ 60 FPS
    ↓
ProcessManager.Update() enforces 40 TPS
    ↓
RainWorldGame.Update() @ 40 TPS
    ↓
World.Update()
    ↓
AbstractRooms.Update() (abstract creatures)
    ↓
Room.Update() (realized entities)
        ↓
    PhysicalObject.Update() (physics, AI)
```

### Render Flow (Top-Down)

```
Unity Rendering Pipeline
    ↓
RainWorld.GrafUpdate()
    ↓
ProcessManager.GrafUpdate()
    ↓
RainWorldGame.GrafUpdate()
    ↓
RoomCamera.GrafUpdate()
    ↓
GraphicsModule.DrawSprites()
    ↓
FSprite rendering
```

## State Persistence

### What Persists Across State Transitions?

**Always persists** (stored in RainWorld):
- Options (player settings)
- Progression (unlocks, karma, etc.)
- Save file data

**Sometimes persists** (via CommunicateWithUpcoming):
- Selected save file (Menu → Game)
- Death location (Game → Death Screen)
- Arena statistics (Arena → Results Screen)

**Never persists** (clean slate):
- World state (destroyed on Game → Menu)
- Creature positions (fresh start each cycle)
- Physics state (no momentum carries over)

### Example: Starting a New Game

```
1. MainMenu (MainLoopProcess)
   - User selects save file
   - User clicks "Start Game"

2. ProcessManager.RequestMainProcessSwitch(
       new RainWorldGame(saveFile)
   )

3. MainMenu.ShutDown()
   - Cleanup UI
   - Fade out

4. new RainWorldGame() constructed
   - Loads save file
   - Creates World for starting region
   - Spawns Player at shelter
   - Initializes rain timer

5. MainMenu destroyed
   RainWorldGame becomes currentMainLoop

6. Gameplay begins
   ProcessManager.Update() now calls RainWorldGame.Update()
```

## Performance Characteristics

### Overhead by Level

**RainWorld**: ~0.1ms per frame
- Minimal work
- Just routes to ProcessManager

**ProcessManager**: ~0.5ms per frame
- Frame timing logic
- State management
- Input routing

**MainLoopProcess**: Varies by type
- MenuProcess: ~2ms per frame (UI only)
- RainWorldGame: ~20-23ms per frame (full simulation)

**Total frame budget**: 25ms @ 40 FPS
- Hierarchy overhead: ~0.6ms (2.4%)
- Game logic: ~24.4ms (97.6%)

## Design Tradeoffs

| Decision | Benefit | Cost |
|----------|---------|------|
| **RainWorld singleton** | Convenient global access | Tight coupling, testing difficulty |
| **ProcessManager state machine** | Clean mode separation | Teardown/setup overhead |
| **MainLoopProcess abstraction** | Easy to add new modes | Some boilerplate per mode |
| **Clean state transitions** | No state bleeding bugs | Can't preserve world state easily |
| **40 TPS logic loop** | Deterministic gameplay | Requires interpolation for smooth graphics |

## Related Documentation

**Same layer**:
- [Overview](overview.md) - Complete architecture
- [Update Loop](update-loop.md) - Frame timing details
- [Dual LOD System](dual-lod-system.md) - Entity simulation

**Go deeper**:
- [Layer 2: ProcessManager Implementation](../../02-implementation/systems-implementation/processmanager-impl.md)
- [Layer 3: Unity Update Loop](../../03-technical/unity-specifics/unity-update-loop.md)

**Related topics**:
- [Singleton Coupling Problem](../../appendix/problems/singleton-coupling.md)
- [State Machine Pattern](../../02-implementation/core-patterns/state-machine-pattern.md)

## Key Takeaways

1. **Three levels**: RainWorld (lifecycle) → ProcessManager (states) → MainLoopProcess (modes)
2. **State machine**: Clean separation between menus and gameplay
3. **Clean transitions**: Each state fully tears down before next begins
4. **Update routing**: RainWorld → ProcessManager → current mode
5. **Frame timing**: ProcessManager enforces 40 TPS logic, 60 FPS graphics
6. **Single responsibility**: Each level has clear, focused purpose
