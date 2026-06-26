# Pixel Data

> 3 Q&As · source: AE plugin dev community Discord

### How can you efficiently copy a OpenCV Mat to an After Effects layer?

Use the Iterate Suite and copy entire rows at a time with memcpy() instead of copying pixel-by-pixel. Respect the layer->rowBytes value. Advance both the OpenCV buffer by its own rowbyte and the layer buffer by its own rowbyte to maintain proper memory alignment. Ensure both buffers have the same pixel format (ARGB) and are uchar type for safe memory alignment.

```cpp
static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        char* layerRow = (char*)layer->data + (y * layer->rowbytes);
        char* matRow = mat.ptr(y);
        memcpy(layerRow, matRow, layer->width * sizeof(PF_Pixel));
    }
}
```

*Tags: `caching`, `memory`, `opencv`, `pixel-data`*

---

### What is the pixel format difference between After Effects and OpenCV Mat?

After Effects uses ARGB format by default in dumb render mode, while OpenCV Mat uses BGRA format by default. The color channels are in different order, so you need to convert between them when copying data between the two formats.

*Tags: `color-format`, `pixel-data`*

---

### How can I access pixel data from an AEGP plugin?

An AEGP plugin cannot directly access layer pixels with effects, masks, or transformations applied. Instead, you can access the source pixels of a layer by getting the source item and then using AEGP_RenderAndCheckoutFrame() to render and checkout the frame, followed by AEGP_GetReceiptWorld() to retrieve the pixel data. The source provides pixels without any masks, effects, or transformation applied.

*Tags: `aegp`, `layer-checkout`, `pixel-data`, `reference`*

---
