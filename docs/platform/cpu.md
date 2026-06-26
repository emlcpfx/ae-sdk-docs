# Cpu

> 3 Q&As · source: AE plugin dev community Discord

### How can a plugin support AVX2 requirements given After Effects 2023+ requires it, and should developers worry about older CPU compatibility?

After Effects has required AVX2 since version 2023. Based on Steam hardware survey data, 96.30% of systems support AVX2 (growing 1.63% per month) and 98.25% support AVX (growing 0.77% per month), though this may not directly overlap with video editing users. Most developers using After Effects in 2025, even older versions, are unlikely to encounter machines without AVX2 support. For selective CPU feature support, ISPC (Intel SPMD Program Compiler) can automatically compile multiple versions and switch between them at runtime.

*Tags: `build`, `cpu`, `cross-platform`, `deployment`, `windows`*

---

### What is the minimum CPU instruction set support required for After Effects plugins in 2025?

After Effects has required AVX2 support since version 2023. Based on Steam hardware survey data, 98.25% of machines support AVX (growing 0.77% monthly) and 96.30% support AVX2 (growing 1.63% monthly). Even considering only video editing machines, the overlap suggests virtually no users in 2025 are on machines without AVX2 support. This makes it safe to target higher instruction sets like x86-64-v2 (up to SSSE3) or AVX2 for significant performance improvements without runtime checks.

*Tags: `build`, `cpu`, `performance`, `windows`*

---

### How much faster is GPU rendering compared to CPU path for plugin operations?

According to benchmarks, there is a significant difference: GPU processing can complete operations in milliseconds while the CPU path takes several seconds. However, performance depends on the actual operations being offloaded—if critical work like UV unwrapping still falls back to CPU, the GPU advantage is negated.

*Tags: `benchmark`, `cpu`, `gpu`, `optimization`, `performance`*

---
