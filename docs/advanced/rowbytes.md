# Rowbytes

> 4 Q&As · source: AE plugin dev community Discord

### How do you properly copy pixel data from OpenCV cv::Mat to AE PF_LayerDef?

Never manually allocate layerDef->data - AE owns that memory. Copy pixel data respecting rowbytes alignment. Use the sampleIntegral32 pattern to correctly index pixels. AE uses ARGB format while OpenCV uses BGRA, so channel swizzling is needed. When creating a cv::Mat pointing to AE layer data, pass layerDef->rowbytes as the step parameter. For performance, use memcpy per row (if same pixel format) and the IterateSuite for multi-threaded processing.

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y) {
    return (PF_Pixel*)((char*)def.data +
        (y * def.rowbytes) +
        (x * sizeof(PF_Pixel)));
}

static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        for (int x = 0; x < layer->width; ++x) {
            cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
            PF_Pixel& aePixel = *sampleIntegral32(*layer, x, y);
            aePixel.alpha = pixel[3];
            aePixel.red = pixel[2];
            aePixel.green = pixel[1];
            aePixel.blue = pixel[0];
        }
    }
}

// Create cv::Mat from AE layer with proper stride:
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Tags: `argb`, `bgra`, `cv-mat`, `opencv`, `pixel-format`, `rowbytes`*

---

### Why do rowbytes sometimes differ from width * bytes_per_pixel?

Rowbytes can be larger than expected because AE uses it as an optimization for layer cropping. When a layer is cropped (e.g., dragged partially off the comp), AE updates width/height and moves the data pointer to the new top-left pixel, but leaves rowbytes unchanged. This avoids memory reallocation. After a cache purge, rowbytes may return to the expected value. Always use rowbytes (not width * pixel_size) when iterating rows.

*Tags: `buffer-stride`, `cropping`, `memory-layout`, `pixel-access`, `rowbytes`*

---

### What causes artifacts in 16/32-bit mode when a Track Matte is applied?

The root causes are typically: (1) Incorrect bit depth detection causing memory corruption - using wrong pixel sizes for memory operations. (2) Improper stride/rowbytes handling for padded buffers - when a track matte is applied, the layer gets cropped to a different size, changing the relationship between width and rowbytes. Fix bit depth detection to use rowbytes calculation and implement proper stride handling throughout your processing algorithm. The effect works in 8-bit because the rowbytes happen to align, but in 16/32-bit the padding differences cause corruption.

*Tags: `16-bit`, `32-bit`, `artifacts`, `bit-depth`, `memory-corruption`, `rowbytes`, `track-matte`*

---

### Is PF_LayerDef::rowbytes always a multiple of 4 in After Effects?

Yes, you can be 100% confident that rowbytes will always be a multiple of sizeof(PF_Pixel8), i.e., 4 bytes. However, 16-byte alignment is NOT always guaranteed. According to Daniel Wilk from Adobe, rowbytes is technically arbitrary and your code should be prepared to deal with any stride. In practice it is often aligned to 16-byte boundaries for SIMD and GPU performance. Important: a buffer might be a sub-reference to another buffer, so you should never write into bytes outside your image reference frame into the rowbytes gutter. If rowbytes were not a multiple of the pixel type's alignment, you would be dereferencing misaligned pointers, which is undefined behavior per the C/C++ spec (C spec 6.3.2.3 paragraph 7).

*Tags: `memory-alignment`, `pf-layerdef`, `pixel-buffer`, `rowbytes`, `simd`, `undefined-behavior`*

---
