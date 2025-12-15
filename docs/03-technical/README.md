# Layer 3: Technical Deep Dive

**Purpose**: Understanding how C# and Unity make Rain World work
**Audience**: Engine developers, performance engineers
**Reading time**: 6-12 hours for complete layer
**Prerequisites**: [Layer 1: Systems](../01-systems/), [Layer 2: Implementation](../02-implementation/)

## What This Layer Provides

This layer explains the C# language features, Unity engine specifics, and performance optimizations that enable Rain World's architecture. After reading this layer, you'll understand:

- How C# memory management (GC, structs vs classes) impacts design
- Unity-specific implementations (MonoBehaviour, Update loops, IL2CPP)
- Performance optimization techniques and their tradeoffs
- Algorithmic complexity analysis
- Theoretical performance limits

Each document focuses on **how C# and Unity enable the implementation**, not the high-level architecture (Layer 1) or the algorithms themselves (Layer 2).

## When to Read This Layer

**Read this layer if you want to:**
- Optimize performance in similar games
- Understand C# and Unity best practices in practice
- Learn why specific technical choices were made
- Build high-performance game systems

**Skip to Layer 4 if:**
- You need hardware-level details (memory layout, CPU, GPU)

## Topics Covered

### C# Specifics
**How C# language features are used**

- [Memory Management](csharp-specifics/memory-management.md) - GC, allocations, pooling
- [Structs vs Classes](csharp-specifics/structs-vs-classes.md) - When each is used and why
- [Value Types vs Reference Types](csharp-specifics/value-types-reference-types.md) - BodyChunk (struct) vs Creature (class)
- [Collections](csharp-specifics/collections.md) - List<> vs arrays, Dictionary usage
- [Delegates and Events](csharp-specifics/delegates-events.md) - Event systems, callbacks
- [LINQ Performance](csharp-specifics/linq-performance.md) - When LINQ is used or avoided

**Key question answered**: *How does C# enable this architecture?*

### Unity Specifics
**How Unity engine features are leveraged**

- [MonoBehaviour Lifecycle](unity-specifics/monobehaviour-lifecycle.md) - Awake → Start → Update → Destroy
- [Unity Update Loop](unity-specifics/unity-update-loop.md) - Update vs FixedUpdate vs LateUpdate
- [IL2CPP Compilation](unity-specifics/il2cpp-compilation.md) - IL2CPP vs Mono, implications
- [Unity Physics Integration](unity-specifics/unity-physics-integration.md) - Why NOT using Unity Physics
- [Unity Rendering](unity-specifics/unity-rendering.md) - Sprite rendering, batching
- [Unity Serialization](unity-specifics/unity-serialization.md) - Save system integration

**Key question answered**: *How does Unity enable this architecture?*

### Performance Optimization
**Techniques used to optimize performance**

- [Profiling Hotspots](performance-optimization/profiling-hotspots.md) - Known bottlenecks
- [LOD Performance](performance-optimization/lod-performance.md) - Why abstract is 70x cheaper
- [Collision Optimization](performance-optimization/collision-optimization.md) - Layers, spatial partitioning
- [AI Optimization](performance-optimization/ai-optimization.md) - Staggered updates, early exits
- [Memory Optimization](performance-optimization/memory-optimization.md) - Pooling, struct packing
- [Cache Locality](performance-optimization/cache-locality.md) - Data layout for cache efficiency

**Key question answered**: *How is the game optimized for performance?*

### Algorithms Deep Dive
**Complexity analysis and optimization details**

- [Pathfinding Complexity](algorithms-deep-dive/pathfinding-complexity.md) - Big-O analysis, optimizations
- [Collision Detection Complexity](algorithms-deep-dive/collision-detection-complexity.md) - Broad phase, narrow phase
- [Physics Integration](algorithms-deep-dive/physics-integration.md) - Verlet integration details
- [Constraint Solving](algorithms-deep-dive/constraint-solving.md) - Iterative constraint solver
- [AI Evaluation Cost](algorithms-deep-dive/ai-evaluation-cost.md) - Cost per AI decision

**Key question answered**: *What is the computational cost of each system?*

### Theoretical Limits
**Maximum scalability analysis**

- [Max Creatures](theoretical-limits/max-creatures.md) - Theoretical max entity count
- [Max Rooms](theoretical-limits/max-rooms.md) - Memory constraints
- [Max BodyChunks](theoretical-limits/max-bodychunks.md) - Physics simulation limits
- [Scaling Analysis](theoretical-limits/scaling-analysis.md) - O(n) vs O(n²) scalability

**Key question answered**: *What are the system's limits?*

## Layer Navigation

**Current layer**: Layer 3: Technical Deep Dive

**Go deeper**:
- [Layer 4: Metal Level](../04-metal/) - Memory layout, CPU, compiler, GPU

**Go up**:
- [Layer 2: Implementation](../02-implementation/) - Classes, algorithms, patterns
- [Layer 1: Systems](../01-systems/) - System-level architecture
- [Layer 0: Executive](../00-executive/) - High-level overview

## Reading Paths

**Path 1: Complete Technical** (12 hours)
Read all documents in order. Best for engine developers.

**Path 2: Performance Focus** (4 hours)
- All of Performance Optimization
- Algorithms Deep Dive
- Theoretical Limits

**Path 3: C# and Unity** (6 hours)
- All of C# Specifics
- All of Unity Specifics

**Path 4: Optimization Engineer** (8 hours)
- Performance Optimization
- Algorithms Deep Dive
- Cache Locality
- Then jump to [Layer 4: Memory Layout](../04-metal/memory-layout/)

## Next Steps

After completing this layer:

1. **For hardware-level details**: Proceed to [Layer 4](../04-metal/)
2. **To review architecture**: Return to [Layer 1](../01-systems/)
3. **For implementation context**: See [Layer 2](../02-implementation/)
