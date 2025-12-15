# Appendix: Supplementary Material

**Purpose**: Problems, proposals, and source material
**Audience**: All readers
**Prerequisites**: None (reference as needed)

## What This Appendix Provides

Supplementary documentation including:
- Identified architectural problems
- Proposed improvements
- Comprehensive analysis reports
- Source material and extraction notes

These documents complement the main 5-layer structure by providing critical analysis and proposals.

## Contents

### Problems
**Identified architectural issues**

- [Singleton Coupling](problems/singleton-coupling.md) - Global state coupling issues
- [Inheritance Rigidity](problems/inheritance-rigidity.md) - Deep hierarchy problems
- [LOD Data Loss](problems/lod-data-loss.md) - State lost during transitions

Each problem document includes:
- Concrete examples
- Real impact analysis
- Severity assessment
- Related improvement proposals

### Improvements
**Proposed architectural enhancements**

- [ECS Migration](improvements/ecs-migration.md) - Entity Component System proposal
- [Dependency Injection](improvements/dependency-injection.md) - Replacing singleton pattern
- [Seamless LOD](improvements/seamless-lod.md) - Preserving more state
- [Soft Room Boundaries](improvements/soft-room-boundaries.md) - Cross-room interactions

Each improvement document includes:
- Detailed design
- Implementation path
- Benefits vs costs analysis
- Timeline estimates

### Analysis
**Comprehensive architectural reviews**

- [2025-12-14 Review](analysis/2025-12-14-review.md) - Complete architectural analysis

Reviews include:
- Executive summary
- Strengths and weaknesses
- Quantified metrics
- Recommendations

### Source Material
**Raw materials and extraction notes**

- [Wiki Extraction Notes](source-material/wiki-extraction-notes.md) - How wiki was processed
- [Class Documentation](source-material/class-documentation/) - Per-class reference from wiki

## Navigation

**Return to main documentation**:
- [Layer 0: Executive](../00-executive/)
- [Layer 1: Systems](../01-systems/)
- [Layer 2: Implementation](../02-implementation/)
- [Layer 3: Technical](../03-technical/)
- [Layer 4: Metal](../04-metal/)

**Quick links**:
- [Glossary](glossary.md) - Terms and definitions
- [Main README](../README.md) - Documentation home
