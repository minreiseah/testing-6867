# Rain World Code Architecture

Technical documentation for Rain World's codebase structure.

## Overview

Rain World is a survival platformer with a simulated ecosystem. Creatures hunt, migrate, and interact across ~70 interconnected rooms per region while the player explores a small portion of that space.

The architecture solves this problem:
- Simulate an ecosystem across dozens of rooms
- Maintain smooth performance on limited hardware
- Preserve creature persistence (they continue existing when off-screen)
- Support dynamic LOD transitions as the player moves through the world

## Documentation Structure

Documentation is organized from high-level architecture down to implementation details.

### Layer 1: Game Architecture

[**Architecture Overview**](architecture/overview.md) - Complete system overview covering:
- Three-tier update hierarchy (RainWorld → ProcessManager → Game/Menu states)
- Dual-world system (Abstract vs Realized entities)
- Room streaming
- BodyChunk physics system

### Layer 2: Core Systems

- [**The Game Loop**](architecture/game-loop.md) - Update cycle and frame timing
- [**Dual LOD System**](architecture/dual-lod.md) - Abstract vs realized entity representation
- [**World & Room Management**](architecture/world-rooms.md) - Room streaming and memory management
- [**Creature Intelligence**](architecture/creature-ai.md) - AI systems for abstract and realized states
- [**Physics & Collision**](architecture/physics.md) - BodyChunk-based physics implementation

### Layer 3: Implementation Details

- [**Class Hierarchy Reference**](architecture/class-hierarchy.md)
- [**ProcessManager Deep Dive**](architecture/processmanager.md)
- [**AbstractPhysicalObject vs PhysicalObject**](architecture/lod-details.md)
- [**Graphics Module System**](architecture/graphics.md)
- [**Save System & RegionState**](architecture/save-system.md)

### Layer 4: Architectural Analysis

- [**Design Patterns Used**](architecture/design-patterns.md)
- [**Identified Issues**](architecture/problems/)
- [**Potential Improvements**](architecture/improvements/)

## Quick Reference

**I want to understand...** | **Read this**
--- | ---
How the whole thing works | [Architecture Overview](architecture/overview.md)
How creatures behave far away | [Dual LOD System](architecture/dual-lod.md)
How the world loads/unloads | [World & Room Management](architecture/world-rooms.md)
How physics/collision works | [Physics & Collision](architecture/physics.md)
How to mod Rain World | Start with [Architecture Overview](architecture/overview.md), then [Class Hierarchy](architecture/class-hierarchy.md)
Why certain design decisions were made | [Architecture Overview](architecture/overview.md) + [Design Patterns](architecture/design-patterns.md)

