# Layer 4: Metal Level

**Purpose**: Understanding how hardware executes Rain World
**Audience**: Systems programmers, hardware engineers
**Reading time**: 4-8 hours for complete layer
**Prerequisites**: [Layer 3: Technical Deep Dive](../03-technical/)

⚠️ **IMPORTANT**: This layer contains THEORETICAL ANALYSIS. Without access to the source code, we cannot provide exact implementation details. Documents at this layer provide:
- **Confirmed**: Observable behavior and documented features
- **Inferred**: Logical architectural deductions (marked)
- **Theoretical**: Likely implementation based on C#/Unity best practices (clearly marked)

## What This Layer Provides

This layer explains how the CPU, GPU, memory, and compiler execute Rain World's code. After reading this layer, you'll understand:

- Memory layout for structs, classes, and arrays
- CPU execution (pipeline, branch prediction, cache)
- Compiler transformations (IL2CPP, JIT/AOT, inlining)
- GPU rendering pipeline (batching, shaders, VRAM)
- Platform-specific constraints and optimizations

Each document focuses on **hardware-level execution**, not the C# code itself (Layer 3) or the algorithms (Layer 2).

## When to Read This Layer

**Read this layer if you want to:**
- Understand memory layout and cache behavior
- Learn CPU and GPU execution details
- Study compiler optimizations in games
- Build extremely high-performance systems

**Note**: This layer provides the deepest technical analysis possible without source code access.

## Topics Covered

### Memory Layout
**How data is organized in RAM**

- [Struct Memory Layout](memory-layout/struct-memory-layout.md) - BodyChunk memory layout (THEORETICAL)
- [Class Memory Layout](memory-layout/class-memory-layout.md) - Object headers, vtables (THEORETICAL)
- [Array Memory Layout](memory-layout/array-memory-layout.md) - Contiguous storage (THEORETICAL)
- [Cache Lines](memory-layout/cache-lines.md) - 64-byte boundaries (THEORETICAL)
- [False Sharing](memory-layout/false-sharing.md) - Multi-threading issues (THEORETICAL)

**Key question answered**: *How is data laid out in memory?*

### CPU Execution
**How the processor executes the game**

- [Instruction Pipeline](cpu-execution/instruction-pipeline.md) - CPU pipeline stages (THEORETICAL)
- [Branch Prediction](cpu-execution/branch-prediction.md) - AI branching cost (THEORETICAL)
- [SIMD Opportunities](cpu-execution/simd-opportunities.md) - Vector operations (THEORETICAL)
- [CPU Cache Hierarchy](cpu-execution/cpu-cache-hierarchy.md) - L1/L2/L3 cache (THEORETICAL)

**Key question answered**: *How does the CPU execute the game loop?*

### Compiler Transformations
**How C# code becomes machine code**

- [IL2CPP Output](compiler-transformations/il2cpp-output.md) - C# → C++ transformation (THEORETICAL)
- [JIT vs AOT](compiler-transformations/jit-vs-aot.md) - Runtime vs ahead-of-time compilation (THEORETICAL)
- [Inlining](compiler-transformations/inlining.md) - Method inlining (THEORETICAL)
- [Loop Optimizations](compiler-transformations/loop-optimizations.md) - Unrolling, vectorization (THEORETICAL)
- [GC Safepoints](compiler-transformations/gc-safepoints.md) - Garbage collection pauses (THEORETICAL)

**Key question answered**: *How is C# code compiled?*

### GPU Rendering
**How graphics are rendered**

- [Sprite Batching](gpu-rendering/sprite-batching.md) - Draw call batching (THEORETICAL)
- [Shader Execution](gpu-rendering/shader-execution.md) - Fragment shader cost (THEORETICAL)
- [GPU Memory](gpu-rendering/gpu-memory.md) - VRAM usage (THEORETICAL)
- [Render Pipeline](gpu-rendering/render-pipeline.md) - CPU → GPU → Screen (THEORETICAL)

**Key question answered**: *How does the GPU render the game?*

### Platform Specifics
**Platform constraints and optimizations**

- [Console Constraints](platform-specifics/console-constraints.md) - Console memory limits (THEORETICAL)
- [PC Optimizations](platform-specifics/pc-optimizations.md) - PC-specific opts (THEORETICAL)
- [Mobile Considerations](platform-specifics/mobile-considerations.md) - Mobile constraints (THEORETICAL)

**Key question answered**: *How do platform differences affect implementation?*

## Theoretical Content Marking

Each document uses this format:

```markdown
**Source access**: ⚠️ THEORETICAL (architectural analysis, no source code)

## What we know (from wiki/observation):
[Documented behavior from official sources]

## What we infer (architectural analysis):
[Logical deductions based on architecture]

## Theoretical analysis:
[What the system LIKELY does, with caveats]
```

Three tiers of confidence:
- ✓ **CONFIRMED**: From wiki, docs, observable behavior
- ⚙️ **INFERRED**: Logical architectural deduction (clearly marked)
- ⚠️ **THEORETICAL**: Requires source code (marked, theoretical analysis provided)

## Layer Navigation

**Current layer**: Layer 4: Metal Level

**Go up**:
- [Layer 3: Technical](../03-technical/) - C#, Unity, performance
- [Layer 2: Implementation](../02-implementation/) - Classes, algorithms
- [Layer 1: Systems](../01-systems/) - System architecture
- [Layer 0: Executive](../00-executive/) - High-level overview

**Supplementary**:
- [Appendix](../appendix/) - Problems, improvements, analysis

## Reading Paths

**Path 1: Complete Metal** (8 hours)
Read all documents in order. Best for systems programmers.

**Path 2: Memory Focus** (3 hours)
- All of Memory Layout
- CPU Cache Hierarchy
- Compiler Transformations (IL2CPP, Inlining)

**Path 3: Performance Analysis** (4 hours)
- Memory Layout
- CPU Execution
- Cache Hierarchy
- GPU Rendering

**Path 4: Academic Research** (6 hours)
- All documents with focus on theoretical analysis methodology
- Compare with [Layer 3 performance data](../03-technical/performance-optimization/)

## Caveats and Limitations

**What this layer can provide**:
- Theoretical analysis based on C#/Unity/hardware best practices
- Inferred behavior from architectural patterns
- Likely optimizations based on target platforms

**What this layer cannot provide without source access**:
- Exact memory layouts
- Precise compiler output
- Specific optimization flags used
- Actual profiling data

**Recommended approach**:
1. Read theoretical analysis
2. Understand the principles
3. If building similar systems, profile your own implementation
4. Validate assumptions through measurement

## Next Steps

After completing this layer:

1. **Review the full architecture**: Return to [Layer 1](../01-systems/)
2. **See practical applications**: Review [Layer 2 Modding](../02-implementation/modding/)
3. **Understand limitations**: Read [Appendix: Problems](../appendix/problems/)
4. **Explore improvements**: See [Appendix: Improvements](../appendix/improvements/)
