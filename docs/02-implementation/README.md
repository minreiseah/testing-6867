# Layer 2: Implementation Details

**Purpose**: Understanding how Rain World is built
**Audience**: Modders, implementers, advanced developers
**Reading time**: 8-16 hours for complete layer
**Prerequisites**: [Layer 1: Systems Architecture](../01-systems/)

## What This Layer Provides

This layer explains how Rain World's systems are implemented through classes, algorithms, and data flow. After reading this layer, you'll understand:

- The complete class hierarchies (abstract layer, realized layer, AI, graphics)
- Core design patterns and their implementation
- Algorithms for pathfinding, collision, AI, and spawning
- Data flow through the update loop
- How to mod Rain World effectively

Each document focuses on **how it's built** (classes, algorithms, patterns), not why those choices were made (that's Layer 1) or C#/Unity specifics (that's Layer 3).

## When to Read This Layer

**Read this layer if you want to:**
- Create Rain World mods
- Understand specific algorithms (pathfinding, collision, AI)
- See how design patterns are applied in practice
- Learn implementation techniques for similar games

**Skip to Layer 3 if:**
- You need C#-specific or Unity-specific details
- You want performance optimization insights

## Topics Covered

### Class Hierarchies
**The object-oriented structure of the codebase**

- [Overview](class-hierarchies/overview.md) - Complete hierarchy diagrams
- [Abstract Layer Classes](class-hierarchies/abstract-layer-classes.md) - AbstractWorldEntity tree
- [Realized Layer Classes](class-hierarchies/realized-layer-classes.md) - PhysicalObject tree
- [AI Classes](class-hierarchies/ai-classes.md) - Intelligence hierarchy
- [Graphics Classes](class-hierarchies/graphics-classes.md) - GraphicsModule tree

**Key question answered**: *What classes exist and how do they relate?*

### Core Patterns
**Design patterns applied throughout the codebase**

- [Patterns Overview](core-patterns/patterns-overview.md) - All patterns explained
- [Singleton Pattern](core-patterns/singleton-pattern.md) - RainWorld singleton implementation
- [State Machine Pattern](core-patterns/state-machine-pattern.md) - ProcessManager states
- [Object Pool Pattern](core-patterns/object-pool-pattern.md) - Entity pooling (if used)
- [Observer Pattern](core-patterns/observer-pattern.md) - Event systems (if used)

**Key question answered**: *What patterns structure the code?*

### Data Flow
**How data moves through the system each frame**

- [Update Flow](data-flow/update-flow.md) - Frame-by-frame execution
- [Input Processing](data-flow/input-processing.md) - Input → Player → World
- [AI Decision Flow](data-flow/ai-decision-flow.md) - Sensory → Decision → Action
- [Save/Load Flow](data-flow/save-load-flow.md) - Serialize → Disk → Deserialize

**Key question answered**: *How does data flow through the game?*

### Algorithms
**Core algorithms that power the simulation**

- [Pathfinding](algorithms/pathfinding.md) - Overview of both abstract and realized
- [Abstract Room Pathfinding](algorithms/abstract-room-pathfinding.md) - Dijkstra on room graph
- [Realized Tile Pathfinding](algorithms/realized-tile-pathfinding.md) - A* on tile grid
- [Collision Algorithms](algorithms/collision-algorithms.md) - Circle-circle, circle-terrain
- [AI State Machines](algorithms/ai-state-machines.md) - Behavior transitions
- [Spawning Algorithms](algorithms/spawning-algorithms.md) - Lineages, dens, timing

**Key question answered**: *How do pathfinding, collision, and AI actually work?*

### Systems Implementation
**Internals of core system classes**

- [ProcessManager Implementation](systems-implementation/processmanager-impl.md) - State transitions, update routing
- [World Implementation](systems-implementation/world-impl.md) - World class internals
- [Room Implementation](systems-implementation/room-impl.md) - Room class internals
- [Creature Implementation](systems-implementation/creature-impl.md) - Creature class internals
- [PhysicalObject Implementation](systems-implementation/physicalobject-impl.md) - Base object internals

**Key question answered**: *How are core classes implemented?*

### Modding
**Practical guides for modding Rain World**

- [Modding Guide](modding/modding-guide.md) - Getting started with mods
- [Hook Points](modding/hook-points.md) - Common hooks (BepInEx)
- [Custom Creatures](modding/custom-creatures.md) - Adding new creature types
- [Custom Items](modding/custom-items.md) - Adding new items
- [Custom Regions](modding/custom-regions.md) - Creating new regions

**Key question answered**: *How do I mod Rain World?*

## Layer Navigation

**Current layer**: Layer 2: Implementation Details

**Go deeper**:
- [Layer 3: Technical Deep Dive](../03-technical/) - C#, Unity, performance
- [Layer 4: Metal Level](../04-metal/) - Memory, CPU, compiler

**Go up**:
- [Layer 1: Systems Architecture](../01-systems/) - System-level understanding
- [Layer 0: Executive Overview](../00-executive/) - High-level summary

**Supplementary**:
- [Appendix: Problems](../appendix/problems/) - Architectural issues
- [Appendix: Improvements](../appendix/improvements/) - Proposed solutions

## Reading Paths

**Path 1: Complete Implementation** (16 hours)
Read all documents in order. Best for comprehensive implementation knowledge.

**Path 2: Modder's Fast Track** (4 hours)
- Class Hierarchies Overview
- Patterns Overview
- All of Modding section
- Relevant algorithm docs as needed

**Path 3: Algorithm Focus** (6 hours)
- All of Algorithms section
- Data Flow docs
- Systems Implementation as needed

**Path 4: Architecture Student** (8 hours)
- Class Hierarchies
- Core Patterns
- Data Flow
- Systems Implementation

## Next Steps

After completing this layer:

1. **For C#/Unity specifics**: Proceed to [Layer 3](../03-technical/)
2. **For hardware-level details**: Jump to [Layer 4](../04-metal/)
3. **To start modding**: See [Modding Guide](modding/modding-guide.md)
4. **For architectural context**: Return to [Layer 1](../01-systems/)
