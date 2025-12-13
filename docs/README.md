# Rain World Architecture Analysis

> Deep dive into Rain World's code architecture, design patterns, and potential improvements

## What This Is

This documentation analyzes Rain World's codebase architecture based on the official modding wiki. The goal is to understand design decisions from first principles and evaluate potential macro-level improvements.

## Quick Start

- **Current Architecture**: See [Architecture Overview](architecture/overview.md) for the big picture
- **Core Patterns**: [Dual LOD system, ProcessManager, Room-based world](architecture/core-patterns.md)
- **Problems Identified**: [Singleton coupling](architecture/problems/singleton-coupling.md), [Deep inheritance](architecture/problems/inheritance-rigidity.md)
- **Proposed Solutions**: [ECS migration](architecture/improvements/ecs-migration.md), [Dependency injection](architecture/improvements/dependency-injection.md)

## Latest Analysis

- [2025-12-14: Architectural Review](architecture/analysis/2025-12-14-review.md) - Initial comprehensive analysis

## Structure

```
architecture/
├── overview.md              # Current architecture overview
├── core-patterns.md         # Key patterns explained
├── problems/                # Identified issues
│   ├── singleton-coupling.md
│   └── inheritance-rigidity.md
├── improvements/            # Proposed solutions
│   ├── ecs-migration.md
│   └── dependency-injection.md
└── analysis/                # Timestamped analyses
    └── 2025-12-14-review.md
```

## Key Findings

Rain World uses several interesting patterns:

1. **Dual LOD System** - Abstract (distant) vs Realized (nearby) entities
2. **ProcessManager** - Clean state machine for game modes
3. **Room-based Streaming** - Progressive world loading
4. **BodyChunks Physics** - Circle-based collision system

Main architectural challenges:

- **Singleton coupling** - Global `RainWorld` instance creates tight coupling
- **Deep inheritance** - Rigid class hierarchies limit flexibility
- **LOD data loss** - Non-persistent entities discarded on abstraction

## Browse the Analysis

Use the sidebar to navigate through the documentation, or start with the [Architecture Overview](architecture/overview.md).
