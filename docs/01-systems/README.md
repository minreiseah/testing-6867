# Layer 1: Systems Architecture

**Purpose**: Understanding how Rain World works at the systems level
**Audience**: Game developers, architects, students
**Reading time**: 4-8 hours for complete layer
**Prerequisites**: None (start here if you're new)

## What This Layer Provides

This layer explains Rain World's core systems, design decisions, and architectural tradeoffs. After reading this layer, you'll understand:

- How the three-tier hierarchy (RainWorld → ProcessManager → Game) organizes the codebase
- Why the dual LOD system enables ecosystem simulation
- How room streaming manages memory efficiently
- Why custom BodyChunk physics was chosen over traditional physics engines
- The tradeoffs behind every major architectural decision

Each document at this layer focuses on **what the system does** and **why it was designed that way**, not implementation details (that's Layer 2).

## When to Read This Layer

**Read this layer if you want to:**
- Understand Rain World's architecture to create mods
- Learn ecosystem simulation patterns for your own game
- Study architectural tradeoffs in game development
- Get a comprehensive mental model of how Rain World works

**Skip to Layer 2 if:**
- You already understand the systems and need implementation details
- You want to know specific class hierarchies or algorithms

## Topics Covered

### Core Architecture
**How the game is organized and updated**

- [Architecture Overview](core-architecture/overview.md) - Complete system overview
- [Three-Tier Hierarchy](core-architecture/three-tier-hierarchy.md) - RainWorld → ProcessManager → Game
- [Dual LOD System](core-architecture/dual-lod-system.md) - Abstract vs Realized entities
- [Update Loop](core-architecture/update-loop.md) - Frame timing, 40 TPS vs 60 FPS

**Key question answered**: *How is the entire game structured?*

### World Systems
**How the world loads, streams, and persists**

- [World and Rooms](world-systems/world-and-rooms.md) - World structure and room management
- [Room Streaming](world-systems/room-streaming.md) - Progressive loading pipeline
- [Region Structure](world-systems/region-structure.md) - How regions are laid out
- [Save and Persistence](world-systems/save-and-persistence.md) - SaveState and RegionState

**Key question answered**: *How does Rain World manage a large interconnected world?*

### Entity Systems
**How entities exist in abstract and realized forms**

- [Abstract Entities](entity-systems/abstract-entities.md) - Lightweight simulation layer
- [Realized Entities](entity-systems/realized-entities.md) - Full physics/graphics layer
- [LOD Transitions](entity-systems/lod-transitions.md) - How entities realize and abstractize
- [Entity Lifecycle](entity-systems/entity-lifecycle.md) - Spawn → Update → Destroy

**Key question answered**: *How do entities exist both near and far from the player?*

### Creature Systems
**How creature AI works at both abstract and realized levels**

- [Creature Intelligence](creature-systems/creature-intelligence.md) - AI overview
- [Abstract AI](creature-systems/abstract-ai.md) - Migration and abstract decision-making
- [Realized AI](creature-systems/realized-ai.md) - Detailed behavior and pathfinding
- [Creature State](creature-systems/creature-state.md) - Hunger, relationships, memory

**Key question answered**: *How do creatures think and behave?*

### Physics Systems
**How custom BodyChunk physics works**

- [BodyChunk Physics](physics-systems/bodychunk-physics.md) - Circle-based physics overview
- [Collision Detection](physics-systems/collision-detection.md) - Layers and collision resolution
- [Terrain Collision](physics-systems/terrain-collision.md) - Creature-terrain interaction
- [Physics Constraints](physics-systems/physics-constraints.md) - Soft-body connections

**Key question answered**: *Why custom physics instead of Unity Physics?*

### Graphics Systems
**How rendering is separated from logic**

- [Graphics Separation](graphics-systems/graphics-separation.md) - GraphicsModule pattern
- [Rendering Pipeline](graphics-systems/rendering-pipeline.md) - Sprite → Shader → Screen
- [Camera System](graphics-systems/camera-system.md) - RoomCamera and effects
- [Visual Effects](graphics-systems/visual-effects.md) - Particles, lighting, shaders

**Key question answered**: *How does rendering work independently from simulation?*

### Design Decisions
**Why Rain World is architected this way**

- [Tradeoffs](design-decisions/tradeoffs.md) - Every major tradeoff explained
- [Why Dual LOD?](design-decisions/why-dual-lod.md) - Alternatives considered
- [Why BodyChunks?](design-decisions/why-bodychunks.md) - vs traditional physics
- [Why Room-Based?](design-decisions/why-room-based.md) - vs open world, portals

**Key question answered**: *Why these specific architectural choices?*

## Layer Navigation

**Current layer**: Layer 1: Systems Architecture

**Go deeper**:
- [Layer 2: Implementation Details](../02-implementation/) - Classes, algorithms, data flow
- [Layer 3: Technical Deep Dive](../03-technical/) - C#, Unity, performance
- [Layer 4: Metal Level](../04-metal/) - Memory, CPU, compiler

**Go up**:
- [Layer 0: Executive Overview](../00-executive/) - 5-minute high-level summary

**Supplementary**:
- [Appendix: Problems](../appendix/problems/) - Identified architectural issues
- [Appendix: Improvements](../appendix/improvements/) - Proposed solutions

## Reading Paths

**Path 1: Complete Understanding** (8 hours)
Read all documents in order. Best for learning the full architecture.

**Path 2: Quick Overview** (2 hours)
- Architecture Overview
- Dual LOD System
- BodyChunk Physics
- Tradeoffs

**Path 3: Modder's Path** (4 hours)
- Architecture Overview
- Entity Lifecycle
- Creature Intelligence
- Graphics Separation
- Then jump to [Layer 2: Modding](../02-implementation/modding/)

**Path 4: Game Developer's Path** (6 hours)
- All of Core Architecture
- All of Design Decisions
- Then jump to [Layer 2: Algorithms](../02-implementation/algorithms/)

## Next Steps

After completing this layer:

1. **For implementation details**: Proceed to [Layer 2](../02-implementation/)
2. **For technical depth**: Jump to [Layer 3](../03-technical/)
3. **For specific topics**: Use the search function (top right)
4. **For supplementary analysis**: See [Appendix](../appendix/)
