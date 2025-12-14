# Rain World: A Code Architecture Study

This documentation provides a comprehensive analysis of Rain World's codebase architecture, written specifically for developers who want to understand how this unique survival platformer actually works under the hood.

## What is Rain World?

Rain World is a survival platformer where you play as a slugcat navigating a hostile, procedurally-populated ecosystem. The game's defining feature is its living world: creatures hunt, migrate, and interact with each other independently of the player. Everything from lizards to vultures operates within an interconnected food chain, creating emergent narratives as you struggle to survive the deadly rain cycles.

This ambitious design—simulating an entire ecosystem while maintaining smooth gameplay—required architectural choices that go far beyond typical platformer engines. Rain World's architecture is specifically engineered to handle:

- **Persistent ecosystem simulation** across dozens of interconnected rooms
- **Dynamic LOD (Level of Detail)** switching as creatures move between nearby and distant areas
- **Memory-efficient streaming** of a large, interconnected world
- **Complex creature AI** that operates both in detail (when near) and abstractly (when far)
- **Deterministic cycle-based progression** where each game session follows predictable time limits

## Documentation Structure

This documentation follows a **macro-to-micro** approach. The first layer gives you the big picture—how Rain World's architecture enables its unique gameplay. Each subsequent layer adds technical detail. You can stop reading at any layer and walk away with useful knowledge.

### Layer 1: Game Architecture (Start Here)

[**Architecture Overview**](architecture/overview.md) - Understand the complete system in 10 minutes. This covers:
- Why Rain World's architecture exists the way it does
- The three-tier update hierarchy (RainWorld → ProcessManager → Game/Menu states)
- The dual-world system (Abstract vs Realized entities)
- How rooms stream in and out
- Why the BodyChunk physics system was chosen

Start here. Even if you read nothing else, this will give you a comprehensive understanding of Rain World's technical design.

### Layer 2: Core Systems

Dive deeper into the systems that make Rain World tick:

- [**The Game Loop**](architecture/game-loop.md) - From startup to each frame's update cycle
- [**Dual LOD System**](architecture/dual-lod.md) - How creatures exist in two forms simultaneously
- [**World & Room Management**](architecture/world-rooms.md) - Memory-efficient streaming and ecosystem persistence
- [**Creature Intelligence**](architecture/creature-ai.md) - AI that works both abstractly and concretely
- [**Physics & Collision**](architecture/physics.md) - Why BodyChunks, not traditional physics

### Layer 3: Implementation Details

Technical specifics for modders and implementers:

- [**Class Hierarchy Reference**](architecture/class-hierarchy.md)
- [**ProcessManager Deep Dive**](architecture/processmanager.md)
- [**AbstractPhysicalObject vs PhysicalObject**](architecture/lod-details.md)
- [**Graphics Module System**](architecture/graphics.md)
- [**Save System & RegionState**](architecture/save-system.md)

### Layer 4: Architectural Analysis

Critical analysis for those interested in the design tradeoffs:

- [**Design Patterns Used**](architecture/design-patterns.md)
- [**Identified Issues**](architecture/problems/) - Singleton coupling, inheritance rigidity
- [**Potential Improvements**](architecture/improvements/) - ECS migration, dependency injection

## Quick Reference

**I want to understand...** | **Read this**
--- | ---
How the whole thing works | [Architecture Overview](architecture/overview.md)
How creatures behave far away | [Dual LOD System](architecture/dual-lod.md)
How the world loads/unloads | [World & Room Management](architecture/world-rooms.md)
How physics/collision works | [Physics & Collision](architecture/physics.md)
How to mod Rain World | Start with [Architecture Overview](architecture/overview.md), then [Class Hierarchy](architecture/class-hierarchy.md)
Why certain design decisions were made | [Architecture Overview](architecture/overview.md) + [Design Patterns](architecture/design-patterns.md)

## Philosophy of This Documentation

Unlike typical API documentation that lists classes and methods, this documentation **explains the reasoning**. Every architectural decision in Rain World exists to solve a specific problem related to simulating a living ecosystem in a memory-constrained environment.

We focus on:
- **Why** systems exist, not just what they do
- **Concrete examples** from actual gameplay
- **Tradeoffs** in each design choice
- **Rain World-specific** context (not generic game engine patterns)

## Contributing

This is a living document. If you discover inaccuracies or find sections that could be clearer, contributions are welcome.

---

**Ready?** Start with the [Architecture Overview](architecture/overview.md) →
