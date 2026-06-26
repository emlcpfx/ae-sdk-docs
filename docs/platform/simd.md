# Simd

> 2 Q&As · source: AE plugin dev community Discord

### Can AE plugins target AVX2 instruction set for better performance?

AE has required AVX2 since version 2023, and Steam surveys show 96%+ CPU support for AVX2. It's reasonable to target x86-64-v3 (AVX2) instead of v2 (SSSE3), which could double performance for SIMD-heavy processing. ISPC is a good option for automatic multi-ISA compilation (compiles multiple versions and auto-switches at runtime). Alternatively, use runtime detection with if(HasAVX()) dispatch, though this adds complexity.

*Tags: `avx2`, `instruction-set`, `ispc`, `optimization`, `performance`, `simd`*

---

### Is PF_LayerDef::rowbytes always a multiple of 4 in After Effects?

Yes, you can be 100% confident that rowbytes will always be a multiple of sizeof(PF_Pixel8), i.e., 4 bytes. However, 16-byte alignment is NOT always guaranteed. According to Daniel Wilk from Adobe, rowbytes is technically arbitrary and your code should be prepared to deal with any stride. In practice it is often aligned to 16-byte boundaries for SIMD and GPU performance. Important: a buffer might be a sub-reference to another buffer, so you should never write into bytes outside your image reference frame into the rowbytes gutter. If rowbytes were not a multiple of the pixel type's alignment, you would be dereferencing misaligned pointers, which is undefined behavior per the C/C++ spec (C spec 6.3.2.3 paragraph 7).

*Tags: `memory-alignment`, `pf-layerdef`, `pixel-buffer`, `rowbytes`, `simd`, `undefined-behavior`*

---
