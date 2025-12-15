# Rain World Architecture Analysis

Comprehensive documentation of Rain World's code architecture, organized in five progressive depth layers from executive overview to hardware-level analysis.

## View Documentation

**Live Documentation**: https://minreiseah.github.io/crawl/

The documentation is automatically deployed via GitHub Pages from the `/docs` folder.

## Quick Start

- **New here?** Start with the [Executive Overview](https://minreiseah.github.io/crawl/#/00-executive/overview) (5 min)
- **Want to build similar games?** Read [Key Innovations](https://minreiseah.github.io/crawl/#/00-executive/key-innovations) → [Layer 1: Systems](https://minreiseah.github.io/crawl/#/01-systems/)
- **Modding Rain World?** Go to Layer 1: Systems → Layer 2: Modding Guide (coming soon)
- **Academic research?** Read all layers progressively

## Documentation Structure

5-layer progressive depth architecture:

- **Layer 0: Executive** (5-15 min) - High-level overview for decision-makers
- **Layer 1: Systems** (4-8 hrs) - Architecture for game developers
- **Layer 2: Implementation** (8-16 hrs) - Details for modders/implementers
- **Layer 3: Technical** (6-12 hrs) - C#/Unity depth for engine devs
- **Layer 4: Metal** (4-8 hrs) - Hardware-level for systems programmers
- **Appendix** - Problems, improvements, analysis

Each layer is independently readable - you can "slice" at any layer and get complete understanding at that abstraction level.

## What's Inside

### Current Architecture
- **Dual LOD system**: Abstract (strategic AI, ~8-13 TPS) vs Realized (tactical AI, 40 TPS)
- **Three-tier hierarchy**: RainWorld → ProcessManager → MainLoopProcess
- **Custom BodyChunk physics**: Circle-based soft-body system
- **Graphics/Logic separation**: Optional GraphicsModule for memory efficiency
- **Room-based streaming**: Progressive loading without pop-in

### Key Innovations
1. Dual LOD for AI/Physics (not just graphics)
2. Custom circle-based soft-body physics
3. Graphics/logic separation pattern
4. Two-tier update rates (40 TPS physics, 60 FPS graphics)
5. Room-based streaming
6. Abstract AI ecosystem (persistent off-screen simulation)

### Problems Identified
- Singleton coupling (global RainWorld access)
- Deep inheritance hierarchies
- LOD transition data loss
- Hard room boundaries

### Proposed Solutions
- Entity Component System (ECS) migration
- Dependency Injection pattern
- Seamless LOD transitions
- Soft room boundaries

## Repository Structure

```
/
├── docs/                          # Documentation (GitHub Pages source)
│   ├── 00-executive/             # Layer 0: Executive overview
│   ├── 01-systems/               # Layer 1: Systems architecture
│   ├── 02-implementation/        # Layer 2: Implementation details
│   ├── 03-technical/             # Layer 3: Technical deep dive
│   ├── 04-metal/                 # Layer 4: Metal level
│   ├── appendix/                 # Supplementary material
│   ├── index.html                # Docsify loader
│   ├── _sidebar.md               # Navigation
│   └── README.md                 # Documentation home
│
├── _wiki_*.html                  # Source: Rain World Modding Wiki
└── README.md                     # This file
```

## Documentation Status

**Phase 1 Complete**: 5-layer structure, executive overview, navigation
**Phase 2 In Progress**: Layer 1 core systems (creature AI, physics, graphics, hierarchy)
**Target**: ~120 files, ~28,400 lines of comprehensive documentation

## Technology Stack Documented

- **Language**: C#
- **Engine**: Unity (2017-2019 era)
- **Physics**: Custom BodyChunk system (not Unity Physics)
- **Rendering**: Unity sprite rendering
- **Modding**: BepInEx framework

## Contributing

Documentation generated from:
- [Rain World Modding Wiki](https://rainworldmodding.miraheze.org/wiki/Rain_World_Code_Structure) (50+ HTML pages)
- Architectural analysis and inference
- Design pattern identification
- Performance analysis

## Local Development (Optional)

If you want to preview changes locally before pushing:

```bash
# Using Python 3
cd docs
python3 -m http.server 3000

# Or using npx
cd docs
npx serve
```

Then open: http://localhost:3000

**Note**: Changes pushed to main are automatically deployed to GitHub Pages.

## Links

- **Live Documentation**: https://minreiseah.github.io/crawl/
- **Source Wiki**: https://rainworldmodding.miraheze.org/wiki/Rain_World_Code_Structure
- **GitHub Repository**: https://github.com/minreiseah/crawl/

---

**Last updated**: 2025-12-15
