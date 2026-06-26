# wunk

**3 contributions** to AE SDK community knowledge.

Top topics: `code-signing`, `macos`, `ae-25-2`, `notarization`, `installer`, `apple-silicon`, `avx2`, `simd`, `performance`, `ispc`

---

## What caused plugins to fail to load in AE 25.2 on macOS?

The issue was related to code signature validation. Adobe built AE 25.2 with a newer Xcode that enforces stricter code signature checks. Multiple root causes were identified: (1) Unsigned plugins now fail to load on Apple Silicon. (2) Installers that don't properly clean up old files can break code signatures - if v1 had file X and v2 removes it, but the installer leaves it, the signature breaks. (3) Some plugins had 0-byte .plugin files after the AE upgrade. Solution: Ensure proper code signing, notarization, and clean installation (remove all old files before installing new version).

*Source: adobe-plugin-devs · 2025-04-08 · Tags: `code-signing`, `macos`, `ae-25-2`, `notarization`, `installer`, `apple-silicon` · [View in Q&A](../qa/code-signing/)*

---

## Can AE plugins target AVX2 instruction set for better performance?

AE has required AVX2 since version 2023, and Steam surveys show 96%+ CPU support for AVX2. It's reasonable to target x86-64-v3 (AVX2) instead of v2 (SSSE3), which could double performance for SIMD-heavy processing. ISPC is a good option for automatic multi-ISA compilation (compiles multiple versions and auto-switches at runtime). Alternatively, use runtime detection with if(HasAVX()) dispatch, though this adds complexity.

*Source: adobe-plugin-devs · 2025-03-19 · Tags: `avx2`, `simd`, `performance`, `ispc`, `instruction-set`, `optimization` · [View in Q&A](../qa/avx2/)*

---

## How do I copy pixel data row by row between Premiere buffers and a 3D API (Vulkan/OpenGL), and why does memcpy crash when copying from the 3D API back to Premiere?

For copying from Premiere to the 3D API, iterate row by row using the row bytes from the Premiere world definition. The crash when copying back likely relates to incorrect buffer mapping or row byte calculations. In Premiere, row bytes can be negative, and image buffers are 16-byte aligned with possible padding at the end of each line. For Vulkan specifically, check the actual VkSubresourceLayout via vkGetImageSubresourceLayout to ensure correct copy offsets. An access violation usually means a calculation is wrong or one of the buffers is not mapped correctly.

```cpp
// Premiere to 3D API (works):
Void PrPtr = (Char)PRData + y * PrRowByte;
Void ApiPtr = (Char)ApiData + y * APIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);

// APIRowByte = width * 4 (channels) * sizeOfPixel (1 for 8-bit, 4 for 32-bit)
// PrRowByte comes from Premiere world def, often negative
```

*Source: adobe-plugin-devs · 2023-03-04 · Tags: `premiere`, `memcpy`, `row-bytes`, `vulkan`, `opengl`, `pixel-buffer`, `negative-row-bytes` · [View in Q&A](../qa/premiere/)*

---
