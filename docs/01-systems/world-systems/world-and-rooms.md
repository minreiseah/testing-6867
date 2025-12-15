# World & Room Management

Regions contain 70+ interconnected rooms but only 1-3 are loaded in memory at any time. This document covers the World and Room streaming system.

## The Big Picture

Think of Rain World's architecture as a matryoshka doll:

```
RainWorldGame
  └─ World (represents one region: Outskirts, Industrial, etc.)
      └─ AbstractRoom[] (all rooms in the region, always exist)
          └─ Room (realized room geometry, loaded on-demand)
              └─ PhysicalObject[] (realized entities, exist when room is active)
```

**Key insight**: The World always contains *all* rooms, but most exist only as lightweight `AbstractRoom` data structures. Only rooms near the player are realized as full `Room` instances with geometry and physics.

## World Class: The Region Container

The `World` object represents a single region (e.g., Outskirts, Industrial Complex, Drainage System). It's created when you start a game cycle and destroyed when the cycle ends (sleep, death, or region change).

### What World Manages

**Spatial data**:
- `abstractRooms[]`: Array of all rooms in the region (always present)
- `activeRooms[]`: List of currently realized rooms (1-3 at a time)
- `loadingRooms[]`: List of RoomPreparers (rooms being loaded in background)

**Ecosystem data**:
- `spawners[]`: Creature spawn points and respawn timers
- `lineages[]`: Special spawning systems for certain creature types
- `offScreenDen`: A special abstract room for roaming creatures (scavengers, vultures, miros birds)

**Region metadata**:
- `name`: Region acronym (SU = Outskirts, HI = Industrial, etc.)
- `region`: RegionData with layout info, subregions, spawn chances
- `shelters[]`: Room indices of all shelter rooms
- `brokenShelters[]`: Which shelters are broken (can't save)
- `gates[]`: Room indices of region gates (connections to other regions)
- `rainCycle`: Rain timer state

**Update processes**:
- `worldProcesses[]`: Global processes (sky rendering, background ambiance, etc.)

### The World Update Loop

Each frame, `World.Update()` does:

1. **Update all AbstractRooms** (cheap):
   - Abstract entity AI updates
   - Migration calculations
   - Abstract interactions (combat, predation)

2. **Update realized rooms** (expensive):
   - Only for rooms in `activeRooms[]`
   - Full physics, AI, graphics

3. **Update spawners and lineages**:
   - Check respawn timers
   - Decide when to spawn new creatures

4. **Handle room realization/abstraction**:
   - Monitor player position
   - Queue next room for loading if approaching boundary
   - Unload distant rooms

**Optimization**: Abstract room updates are staggered—not all abstract rooms update every frame. The World cycles through them, updating a subset each frame.

### World Persistence

Importantly, **World does NOT fully initialize itself from save data**. That's handled by `WorldLoader`, which runs asynchronously after the World is constructed.

**Why asynchronous loading?**
- Large regions (70+ rooms) take time to parse and initialize
- Asynchronous loading prevents freezing the game
- The World can start ticking while background data loads

This creates a brief window where `World` exists but some data isn't initialized yet. This is why some World members are null-checked throughout the codebase.

## AbstractRoom: The Lightweight Room

Every room in the region exists as an `AbstractRoom`, always. This is the room's "identity" and container for abstract entities.

### What AbstractRoom Contains

**Identification**:
- `index`: Unique room index in the region
- `name`: Room name (e.g., "SU_C04")
- `offScreenDen`: Boolean (is this the special off-screen den?)

**Connections**:
- `connections[]`: Array of indices of connected rooms
- `exitConnections[]`: Exit nodes (which exit leads to which room)

**Entities**:
- `entities[]`: List of AbstractWorldEntity (creatures, items) currently in this room

**State**:
- `shelter`: Is this a shelter room?
- `gate`: Is this a gate room?
- `AttractionForCreature()`: Pathfinding attractiveness for abstract AI

**Realized form** (when active):
- `realizedRoom`: Reference to the realized Room instance (null when not realized)

### AbstractRoom Update

Each frame, `AbstractRoom.Update()` handles:

1. **Abstract entity updates**:
   - Creatures decide whether to migrate
   - Abstract combat resolution
   - State updates (hunger, fear)

2. **Spawn/despawn logic**:
   - Check if creatures should spawn from dens
   - Remove dead entities

3. **Realization bookkeeping**:
   - If `realizedRoom` exists, sync data between abstract and realized forms

**Performance**: AbstractRoom updates are cheap—just state machines and probability checks, no physics or pathfinding.

## Room: The Realized Geometry

When the player approaches, an AbstractRoom "realizes" into a full `Room` instance.

### What Room Contains

**Geometry**:
- `Tiles[][]`: 2D array of tile types (solid, floor, wall, etc.)
- `GetTile(x, y)`: Query tile at coordinates
- `aimap`: Pathfinding grid for AI
- `shortcuts[]`: Shortcut (pipe) entrance/exit points

**Physics**:
- `gravity`: Gravity direction and strength
- `water`: Water level and flow
- `FloatWaterLevel()`: Water surface height

**Entities**:
- `physicalObjects[][]`: 2D array of PhysicalObject lists (indexed by collision layer)
- `updateList[]`: All UpdatableAndDeletable entities in room

**Graphics**:
- `roomSettings`: Lighting, palette, effects
- `cameraPositions[]`: Camera anchor points
- `waterObject`: Rendered water effects
- `shortcuts`: Visual shortcut pipes

**Metadata**:
- `abstractRoom`: Back-reference to AbstractRoom
- `world`: Back-reference to World

### Room Update Loop

Each frame, `Room.Update()` processes:

1. **Pre-update**:
   - Handle room transitions (creatures entering/leaving)
   - Process shortcut (pipe) travel

2. **Update all entities**:
   ```csharp
   foreach (UpdatableAndDeletable entity in updateList) {
       entity.Update();
   }
   ```
   This includes:
   - Creature AI
   - Physics objects
   - Cosmetic effects
   - Background ambiance

3. **Physics resolution**:
   - Terrain collision
   - Inter-object collision (within each layer)
   - Water simulation

4. **Graphics update** (if visible):
   - Update graphics modules
   - Particle effects
   - Lighting

**Performance**: Rooms are expensive, which is why only 1-3 exist at once.

## Progressive Room Loading

Rain World uses **progressive loading** to hide the cost of realizing rooms. When you approach a room boundary:

### The Loading Process

1. **Player approaches boundary**:
   - World detects player within ~2 screens of an exit
   - Checks if the adjacent room is already realized
   - If not, creates a `RoomPreparer`

2. **RoomPreparer background loading**:
   ```csharp
   RoomPreparer preparer = new RoomPreparer(nextAbstractRoom);
   world.loadingRooms.Add(preparer);
   ```

3. **Over several frames**:
   - Load room file (geometry data)
   - Parse tiles, objects, spawn points
   - Initialize aimap (pathfinding data structure)
   - Realize abstract entities in that room
   - Instantiate room processes (water, background)

4. **Loading completes**:
   - `RoomPreparer.done` = true
   - Realized Room added to `world.activeRooms[]`
   - Player can now enter smoothly (no loading hitch)

5. **Old room cleanup**:
   - After player moves far enough away
   - Old Room abstracts: entities save state, geometry unloads
   - Memory is freed

**Key detail**: Loading happens *while you're still in the current room*. By the time you reach the exit, the next room is ready. This creates seamless exploration.

### Handling Fast Movement

What if the player moves faster than progressive loading can prepare rooms?

**Shortcut (pipe) travel**:
- When entering a pipe, the game briefly pauses updates
- Loads the destination room synchronously (blocking)
- Teleports player to destination
- Resumes updates

This is why pipe travel has a brief loading moment—it's a synchronous room load.

**Pole climb between regions**:
- Similar to shortcuts
- Synchronous load + brief freeze
- Acceptable because region transitions are rare

**Solution for normal movement**: Preemptive loading starts well before the player reaches the boundary. Unless the player is moving extremely fast (unusual), the room is ready before they arrive.

## Memory Management

### Memory Budget

Rain World targets limited hardware (original release was console-focused). Memory management is critical:

**Per realized room**: ~20-50 MB (varies by room complexity)
- Tile data: ~2 MB
- Entities: ~5-20 MB (depends on creature count)
- Graphics: ~10-20 MB (sprites, shaders, effects)
- Physics: ~3-5 MB (collision grid, body chunks)

**Abstract room**: ~100-500 KB
- Just data structures (entity lists, connections, state)

**Budget**: With ~200 MB available for gameplay (after OS, Unity, base systems), we can afford ~3-4 realized rooms maximum.

**Typical usage**:
- Current room: 30 MB
- Next room (preloading): 25 MB
- Previous room (buffered): 20 MB
- **Total**: ~75 MB for realized rooms
- Leaves headroom for abstract layer, AI, and other systems

### Cleanup Strategy

**Eager cleanup**:
- Room abstracts as soon as it's >3 rooms away from player
- Realized entities destroyed immediately
- Graphics modules disposed
- Tile data unloaded

**Delayed cleanup** (optional buffer):
- Previous room stays realized briefly (20-30 seconds) in case player backtracks
- If player returns within buffer time: instant, no loading
- If not: room abstracts normally

This buffer creates smoother backtracking while not holding too much memory.

## Special Room Types

### Shelter Rooms

**Characteristics**:
- Save points
- Safe from rain
- All creatures (except player) abstract when player sleeps
- Broken shelters: marked in `world.brokenShelters[]`, can't save

**Sleep cycle**:
1. Player enters shelter before rain
2. Game triggers sleep cutscene
3. All non-player creatures abstract
4. World saves state to disk
5. Game destroys current RainWorldGame
6. Creates new RainWorldGame for next cycle
7. Player wakes in same shelter (new cycle, fresh world state)

### Gate Rooms

**Characteristics**:
- Connect two regions
- Require karma level to pass
- Trigger region transition (synchronous load)

**Gate transition**:
1. Player passes karma check
2. Gate opens
3. Player enters gate
4. Current World unloads completely
5. New World (next region) loads synchronously
6. Player spawns in new region's gate room

This is one of the few full-blocking loads in Rain World.

### Off-Screen Den

**Special abstract room** not tied to geometry:
- `world.offScreenDen`
- Houses creatures that aren't bound to specific rooms (scavengers, vultures, miros birds)
- These creatures migrate *from* the off-screen den into the region
- Allows creatures to "enter" the region dynamically

**Example**:
- Vulture spawns in off-screen den
- Abstract AI decides to hunt
- Migrates from off-screen den to room SU_A42
- If player is in SU_A42, vulture realizes and attacks

## Pathfinding Across Rooms

### Abstract Pathfinding

Abstract creatures navigate using a **room connectivity graph**:

- Nodes = AbstractRooms
- Edges = room connections
- Weights = attractiveness scores (`AbstractRoom.AttractionForCreature()`)

**Algorithm**: Modified Dijkstra's algorithm
- Find shortest path through room network
- Weighted by attractiveness (avoid dangerous rooms, prefer hunting grounds)
- Result: sequence of room indices to traverse

**Migration**:
Each frame, abstract creature AI:
```csharp
if (timeSinceLastMigration > random(300, 500)) {
    Path path = CalculatePath(currentRoom, targetRoom);
    if (path.length > 0) {
        MoveToRoom(path.nextRoom);
    }
}
```

Creatures "teleport" between rooms (no detailed navigation), but the path makes ecological sense.

### Realized Pathfinding

Realized creatures use detailed pathfinding within rooms:

- `room.aimap`: Pathfinding grid (tiles marked as accessible, inaccessible, dangerous)
- Algorithm: A* or simpler heuristic-based navigation
- Creatures navigate pixel-level geometry: poles, tunnels, ledges

**Cross-room pursuit**:
When chasing across room boundaries:
1. Predator follows prey to room exit
2. Predator's abstract form migrates to next room
3. If player is in next room, predator realizes there
4. Chase continues seamlessly (from player's perspective)

## Performance Characteristics

| Operation | Cost | Frequency |
|-----------|------|-----------|
| Abstract room update | Very low (~100 instructions) | Every 5-10 frames |
| Realized room update | High (~50,000 instructions) | Every frame |
| Room realization | Very high (~10 million instructions) | ~Every 30 seconds |
| Room abstraction | High (~500,000 instructions) | ~Every 30 seconds |
| Abstract creature migration | Low (~300 instructions) | Every few hundred frames |

**Optimization insight**: Because realization/abstraction is expensive, the game predicts player movement to start progressive loading early. This amortizes the cost over many frames.

## Summary

Key components:
- Abstract rooms: Always exist (lightweight)
- Realized rooms: Created on-demand (1-3 at a time)
- Progressive loading: Hides realization cost
- Memory budget: ~200MB for 3-4 rooms

## Related Documentation

- [Dual LOD System](dual-lod.md)
- [Creature Intelligence](creature-ai.md)
- [Architecture Overview](overview.md)
