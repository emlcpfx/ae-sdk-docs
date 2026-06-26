# Cpu Optimization

> 1 Q&A · source: AE plugin dev community Discord

### What tool can automatically compile multiple CPU instruction set versions and switch between them at runtime?

ISPC (Intel SPMD Program Compiler) can compile multiple versions of kernels with different CPU instruction sets (like AVX, AVX2) and automatically switch between them at runtime. This is useful for plugin developers who want to support a range of hardware while taking advantage of advanced SIMD instructions on capable processors.

*Tags: `build`, `cpu-optimization`, `cross-platform`, `performance`*

---
