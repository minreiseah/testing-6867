# Rain World: Executive Architecture Overview

**Reading time**: 5 minutes
**Last updated**: 2025-12-15

## The Challenge

Rain World is a survival platformer with a simulated ecosystem. The technical challenge: simulate hundreds of creatures hunting, migrating, and interacting across 70+ interconnected rooms per region, while the player smoothly explores a small portion of that space on limited hardware (consoles, lower-end PCs).

Traditional approaches fail:
- **Simulate everything always**: Performance collapse with hundreds of fully-simulated creatures
- **Only simulate visible entities**: World feels fake, creatures don't persist
- **Standard LOD (graphics only)**: Doesn't solve AI and physics simulation cost

## The Solution: Dual-World Architecture

Rain World's core innovation is a **dual-world architecture** where every entity exists in two forms simultaneously:

### 1. Abstract Form (Always Active, Lightweight)
When creatures are far from the player:
- Move between rooms using probability models
- Interact via simplified dice-roll logic
- Update every few frames with minimal CPU cost
- Track essential state only (hunger, location, relationships)

Think: strategic board game where pieces move across a map.

### 2. Realized Form (Near Player, Full Detail)
When creatures are in or near the player's room:
- Full physics simulation with BodyChunks
- Detailed AI with pathfinding and behavior trees
- Graphics rendering with animations
- Update every frame at 40 TPS

Think: detailed platformer combat and movement.

### Seamless Transitions
As the player moves, entities smoothly transition between forms:
- **Realization**: Abstract → Realized (load full detail)
- **Abstraction**: Realized → Abstract (save state, free memory)

**Result**: An ecosystem that feels alive across the entire region while only paying full simulation cost for nearby entities.

## Three-Tier Hierarchy

The codebase is organized in three layers:

```
RainWorld (singleton, lifecycle manager)
  └─ ProcessManager (state machine for game modes)
      └─ MainLoopProcess (current state: menu, gameplay, etc.)
          └─ RainWorldGame (actual gameplay)
              └─ World (current region)
                  └─ Rooms (streaming level geometry)
                      └─ Entities (creatures, items)
```

- **RainWorld**: Persistent Unity MonoBehaviour, manages global resources
- **ProcessManager**: Clean state machine for menus vs gameplay vs cutscenes
- **MainLoopProcess**: Current game mode (story, arena, menu)
- **RainWorldGame**: The actual game simulation
- **World**: Current region being played
- **Rooms**: Streamed in/out as player explores

## Key Technical Decisions

| Decision | Benefit | Cost | Rationale |
|----------|---------|------|-----------|
| **Dual LOD** | Simulate entire ecosystem | State loss on transitions | Enable persistent living world on limited hardware |
| **Custom BodyChunk physics** | Performance, squishiness, determinism | Less realistic than polygon physics | Creatures need soft-body deformation, circle collision is faster |
| **Room-based streaming** | Memory efficient, predictable | Visible boundaries, no cross-room vision | Manageable chunks, clean loading |
| **Graphics/logic separation** | Memory savings, moddability | Sync complexity | Graphics optional when off-camera |
| **40 TPS physics, 60 FPS graphics** | Deterministic gameplay, smooth rendering | Interpolation required | Fair difficulty needs determinism |

## What This Enables

**Persistent ecosystem**: A lizard you encounter continues existing off-screen. Return later and it might still be there, or it might have migrated hunting for food.

**Emergent narratives**: Stumble onto aftermath scenes (scavenger corpses, territorial disputes) that happened while you were elsewhere.

**Fair challenge**: Deterministic simulation makes brutal difficulty feel fair—same cycle, same outcomes.

**Large interconnected world**: Regions with 70+ rooms fit in memory and run smoothly via streaming + abstract simulation.

## Performance Characteristics

**Typical world state**:
- **Total entities**: 200-400 creatures + items across region
- **Abstract entities**: 95% (lightweight simulation)
- **Realized entities**: 5% (full simulation)
- **Performance difference**: Abstract entities ~70x cheaper than realized

**Update cycle**:
- **Physics**: 40 TPS (ticks per second) - deterministic simulation
- **Graphics**: 60 FPS - smooth rendering
- **Abstract AI**: Every 3-5 frames - strategic decisions
- **Realized AI**: Every frame - tactical decisions

**Memory management**:
- **Loaded rooms**: 1-3 at any time
- **Room streaming**: Progressive background loading
- **Graphics modules**: Created/destroyed as entities enter/leave view
- **Entity pooling**: Minimizes garbage collection

## Architectural Patterns

**Strengths**:
- Clean state machine (ProcessManager)
- Clear separation of concerns (physics, graphics, AI)
- Modular entity system (UpdatableAndDeletable lifecycle)
- Effective LOD strategy for ecosystem simulation

**Known Issues**:
- Singleton coupling (RainWorld globally accessible)
- Deep inheritance hierarchies (rigid creature categorization)
- State loss during LOD transitions
- Hard room boundaries (no cross-room interactions)

See [Appendix: Problems](../appendix/problems/) for detailed analysis.

## Technology Stack

- **Language**: C#
- **Engine**: Unity (2017-2019 era)
- **Physics**: Custom BodyChunk system (not Unity Physics)
- **Rendering**: Unity sprite rendering
- **AI**: Custom abstract + realized AI systems
- **Modding**: BepInEx framework

## Applicability to Other Games

**This architecture is ideal for**:
- Ecosystem simulation games
- Large persistent worlds with AI
- Games needing deterministic simulation
- Memory-constrained platforms (consoles, mobile)

**This architecture is overkill for**:
- Small confined levels
- Games with few AI entities
- Pure action games without persistence
- PC-only titles with ample memory

**Key reusable patterns**:
- Dual LOD for AI and physics (not just graphics)
- Room-based streaming for large worlds
- Separate update rates for physics vs rendering
- Graphics module pattern for optional rendering

## Further Reading

**5-minute reads**:
- [Key Innovations](key-innovations.md) - What makes Rain World unique
- [Architecture at a Glance](architecture-at-a-glance.md) - Visual diagram
- [Quick Reference](quick-reference.md) - FAQ

**Deeper dives**:
- [Layer 1: Systems Architecture](../01-systems/) (4-8 hours) - How each system works
- [Layer 2: Implementation](../02-implementation/) (8-16 hours) - Classes and algorithms
- [Layer 3: Technical](../03-technical/) (6-12 hours) - C#, Unity, performance
- [Layer 4: Metal](../04-metal/) (4-8 hours) - Memory, CPU, compiler

**Supplementary**:
- [Appendix: Problems](../appendix/problems/) - Architectural issues
- [Appendix: Improvements](../appendix/improvements/) - Proposed solutions
- [Appendix: Analysis](../appendix/analysis/) - Comprehensive review

---

**Document scope**: Executive summary suitable for decision-makers, researchers, and first-time readers. For technical details, proceed to [Layer 1](../01-systems/).
