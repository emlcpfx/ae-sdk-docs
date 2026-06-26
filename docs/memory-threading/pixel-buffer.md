# Pixel Buffer

> 2 Q&As · source: AE plugin dev community Discord

### How do I copy pixel data row by row between Premiere buffers and a 3D API (Vulkan/OpenGL), and why does memcpy crash when copying from the 3D API back to Premiere?

For copying from Premiere to the 3D API, iterate row by row using the row bytes from the Premiere world definition. The crash when copying back likely relates to incorrect buffer mapping or row byte calculations. In Premiere, row bytes can be negative, and image buffers are 16-byte aligned with possible padding at the end of each line. For Vulkan specifically, check the actual VkSubresourceLayout via vkGetImageSubresourceLayout to ensure correct copy offsets. An access violation usually means a calculation is wrong or one of the buffers is not mapped correctly.

```cpp
// Premiere to 3D API (works):
Void PrPtr = (Char)PRData + y * PrRowByte;
Void ApiPtr = (Char)ApiData + y * APIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);

// APIRowByte = width * 4 (channels) * sizeOfPixel (1 for 8-bit, 4 for 32-bit)
// PrRowByte comes from Premiere world def, often negative
```

*Tags: `memcpy`, `negative-row-bytes`, `opengl`, `pixel-buffer`, `premiere`, `row-bytes`, `vulkan`*

---

### Is PF_LayerDef::rowbytes always a multiple of 4 in After Effects?

Yes, you can be 100% confident that rowbytes will always be a multiple of sizeof(PF_Pixel8), i.e., 4 bytes. However, 16-byte alignment is NOT always guaranteed. According to Daniel Wilk from Adobe, rowbytes is technically arbitrary and your code should be prepared to deal with any stride. In practice it is often aligned to 16-byte boundaries for SIMD and GPU performance. Important: a buffer might be a sub-reference to another buffer, so you should never write into bytes outside your image reference frame into the rowbytes gutter. If rowbytes were not a multiple of the pixel type's alignment, you would be dereferencing misaligned pointers, which is undefined behavior per the C/C++ spec (C spec 6.3.2.3 paragraph 7).

*Tags: `memory-alignment`, `pf-layerdef`, `pixel-buffer`, `rowbytes`, `simd`, `undefined-behavior`*

---
