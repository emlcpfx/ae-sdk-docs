# Ispc

> 1 Q&A · source: AE plugin dev community Discord

### Can AE plugins target AVX2 instruction set for better performance?

AE has required AVX2 since version 2023, and Steam surveys show 96%+ CPU support for AVX2. It's reasonable to target x86-64-v3 (AVX2) instead of v2 (SSSE3), which could double performance for SIMD-heavy processing. ISPC is a good option for automatic multi-ISA compilation (compiles multiple versions and auto-switches at runtime). Alternatively, use runtime detection with if(HasAVX()) dispatch, though this adds complexity.

*Tags: `avx2`, `instruction-set`, `ispc`, `optimization`, `performance`, `simd`*

---
