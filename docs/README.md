# Rain World Code Architecture

Comprehensive technical documentation for Rain World's codebase, organized in five progressive depth layers.

## Quick Start

**New here? Start with**: [Executive Overview](00-executive/overview.md) (5-minute read)

**Want to build similar games?**: [Key Innovations](00-executive/key-innovations.md) → [Layer 1: Systems](01-systems/)

**Modding Rain World?**: [Layer 1: Systems](01-systems/) → [Layer 2: Modding Guide](02-implementation/modding/modding-guide.md)

**Academic research?**: Read all layers progressively

## Documentation Structure

This documentation is organized as a **tree with 5 depth layers**. Each layer is independently readable - you can "slice" at any layer and get complete understanding at that abstraction level.

### Layer 0: Executive Overview
**5-15 minutes | For executives, researchers, first-time readers**

- [Overview](00-executive/overview.md) - Complete architectural summary
- [Key Innovations](00-executive/key-innovations.md) - What makes Rain World unique
- [Architecture at a Glance](00-executive/architecture-at-a-glance.md) - Visual diagram
- [Quick Reference](00-executive/quick-reference.md) - FAQ and lookup table

**When to read**: You want a high-level understanding without technical depth.

### Layer 1: Systems Architecture
**4-8 hours | For game developers, architects, students**

[**→ Start Here**](01-systems/)

Complete system-level documentation covering:
- Core architecture (three-tier hierarchy, dual LOD, update loop)
- World systems (room streaming, persistence, regions)
- Entity systems (abstract/realized, LOD transitions, lifecycle)
- Creature systems (AI, behavior, state)
- Physics systems (BodyChunks, collision, constraints)
- Graphics systems (rendering pipeline, separation pattern)
- Design decisions (tradeoffs, alternatives considered)

**When to read**: You want to understand how Rain World works and why it's designed this way.

### Layer 2: Implementation Details
**8-16 hours | For modders, implementers, advanced developers**

[**→ Start Here**](02-implementation/)

Implementation-level documentation covering:
- Class hierarchies (abstract layer, realized layer, AI, graphics)
- Core patterns (singleton, state machine, object pool, observer)
- Data flow (update flow, input processing, AI decisions, save/load)
- Algorithms (pathfinding, collision, AI state machines, spawning)
- Systems implementation (ProcessManager, World, Room, Creature internals)
- Modding guides (hooks, custom creatures, items, regions)

**When to read**: You want to mod Rain World or implement similar systems.

### Layer 3: Technical Deep Dive
**6-12 hours | For engine developers, performance engineers**

[**→ Start Here**](03-technical/)

Technical documentation covering:
- C# specifics (memory management, structs vs classes, collections)
- Unity specifics (MonoBehaviour lifecycle, update loops, IL2CPP)
- Performance optimization (profiling, LOD performance, collision, AI, cache)
- Algorithms deep dive (complexity analysis, integration, constraint solving)
- Theoretical limits (max creatures, rooms, BodyChunks, scaling)

**When to read**: You want to understand C#/Unity implementation and performance optimization.

### Layer 4: Metal Level
**4-8 hours | For systems programmers, hardware engineers**

[**→ Start Here**](04-metal/)

⚠️ **Note**: This layer contains theoretical analysis (no source code access).

Hardware-level documentation covering:
- Memory layout (structs, classes, arrays, cache lines)
- CPU execution (pipeline, branch prediction, SIMD, cache hierarchy)
- Compiler transformations (IL2CPP, JIT/AOT, inlining, optimizations)
- GPU rendering (batching, shaders, VRAM, pipeline)
- Platform specifics (console constraints, PC/mobile optimizations)

**When to read**: You want to understand how hardware executes the game.

### Appendix: Supplementary Material
**Reference as needed | For all readers**

[**→ Browse**](appendix/)

Supplementary documentation:
- **Problems**: Identified architectural issues (singleton coupling, inheritance rigidity, LOD data loss)
- **Improvements**: Proposed solutions (ECS migration, dependency injection, seamless LOD)
- **Analysis**: Comprehensive reviews and reports
- **Source Material**: Wiki extraction notes, class documentation
- **Glossary**: Terms and definitions

**When to read**: You want critical analysis or reference material.

## How to Use This Documentation

### Progressive Reading (Recommended)
Read layers in order from 0→4:
1. Layer 0 (15 min) - Get oriented
2. Layer 1 (8 hours) - Understand systems
3. Layer 2 (16 hours) - Learn implementation
4. Layer 3 (12 hours) - Technical depth
5. Layer 4 (8 hours) - Hardware level

**Total**: ~44 hours for complete mastery

### Targeted Reading
Jump directly to the layer that matches your needs:
- **Executives**: Layer 0 only
- **Game developers**: Layers 0-1
- **Modders**: Layers 0-2
- **Engine developers**: Layers 0-3
- **Systems programmers**: All layers

### Topic-Based Reading
Use search (top right) to find specific topics, then read:
1. The document you found
2. "Related at this layer" links for context
3. "Go deeper" links for more detail

## Navigation Tips

Each document includes:
- **Go deeper**: Links to more detailed layers
- **Go up**: Links to higher-level overviews
- **Related at this layer**: Lateral navigation
- **Layer Navigation**: Quick jumps between layers

## What Rain World Achieves

Rain World's architecture enables:
- **Persistent ecosystem**: 200-400 creatures simulated across 70+ rooms
- **Deterministic gameplay**: Fair, repeatable challenge
- **Smooth performance**: 40 TPS physics + 60 FPS graphics on limited hardware
- **Emergent narratives**: Off-screen creature interactions create discovered stories
- **Moddability**: Clear separation enables extensive modding

## Key Innovations

1. **Dual LOD for AI/Physics** - Not just graphics LOD
2. **Custom BodyChunk Physics** - Soft-body circle-based system
3. **Graphics/Logic Separation** - Optional graphics modules
4. **Two-Tier Update Rates** - 40 TPS physics, 60 FPS graphics
5. **Room-Based Streaming** - Progressive loading without pop-in
6. **Abstract AI Ecosystem** - Persistent off-screen simulation

See [Key Innovations](00-executive/key-innovations.md) for details.

## Technology Stack

- **Language**: C#
- **Engine**: Unity (2017-2019 era)
- **Physics**: Custom BodyChunk system
- **Rendering**: Unity sprite rendering
- **Modding**: BepInEx framework

## Documentation Status

**Phase 1 Complete** (Current):
- ✅ 5-layer structure established
- ✅ Layer README files for all layers
- ✅ Executive overview and key innovations
- ✅ Core architecture documentation
- ✅ Update loop detailed
- ✅ Navigation system in place

**Phase 2 In Progress**:
- ⏳ Remaining Layer 1 files (creature AI, physics details, graphics, design decisions)
- ⏳ Layer 2 implementation details
- ⏳ Layer 3 technical deep dives
- ⏳ Layer 4 metal-level analysis

**Target**: ~120 files, ~28,400 lines of comprehensive documentation

## Contributing

This documentation is generated from:
- Official Rain World Modding Wiki (50+ HTML pages)
- Architectural analysis and inference
- Design pattern identification
- Performance analysis

See [Appendix: Source Material](appendix/source-material/) for extraction notes.

## Getting Started

1. **First time**: Read [Executive Overview](00-executive/overview.md) (5 min)
2. **Want to learn**: Start [Layer 1: Systems](01-systems/) (8 hours)
3. **Want to mod**: Jump to [Modding Guide](02-implementation/modding/modding-guide.md)
4. **Specific question**: Use search (top right)

---

**Live documentation**: https://minreiseah.github.io/crawl/
**Last updated**: 2025-12-15
