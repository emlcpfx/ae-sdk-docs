# Q&A: pixel-format

**5 entries** tagged with `pixel-format`.

---

## How do you properly copy pixel data from OpenCV cv::Mat to AE PF_LayerDef?

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

*Contributors: [**fad**](../contributors/fad/), [**tlafo**](../contributors/tlafo/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2024-06-09 · Tags: `opencv`, `pixel-format`, `argb`, `bgra`, `rowbytes`, `cv-mat`*

---

## How do you detect bit depth of a PF_LayerDef without the World Suite?

You can check world_flags: PF_WorldFlag_DEEP indicates 16bpc, and PF_WorldFlag_RESERVED1 indicates 32bpc (undocumented). However, this uses private/undocumented API and may not work in future versions. The official way in Premiere is PF_PixelFormatSuite1 (see SDK_Noise sample). You can also calculate from the rowbytes:width ratio, though rowbytes can be larger than expected due to AE's buffer cropping optimization.

```cpp
int get_bitdepth(const PF_LayerDef& layer) {
    if (layer.world_flags & PF_WorldFlag_DEEP) {
        return 16;
    } if (layer.world_flags & PF_WorldFlag_RESERVED1) {
        return 32;
    } else {
        return 8;
    }
}
```

*Contributors: [**fad**](../contributors/fad/), [**rowbyte**](../contributors/rowbyte/), [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: adobe-plugin-devs · 2024-12-26 · Tags: `bit-depth`, `world-flags`, `pf-layer-def`, `undocumented`, `pixel-format`*

---

## How can I make a plugin for 8-bit, 16-bit, and 32-bit projects share the same internal codebase?

Use C++ templates to create 3 instances of your algorithm with different data types and range constants. This allows you to write the core logic once and instantiate it for PF_Pixel8, PF_Pixel16, and PF_PixelFloat with appropriate range constants for each bit depth.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-11-01 · Tags: `bit-depth`, `templates`, `pixel-format`, `smart-render`, `code-reuse`*

---

## What is the optimal way to make an AE effect plugin compatible with all bit depths (8, 16, and 32 bpc)?

The simplest approach is to write wrapper/converter functions to convert to 32bpc and back, then write your processing code once in 32bpc. These converter functions can also use the multi-threaded iteration suites. The alternative is branching with switch statements for each bit depth, but that results in three separate pieces of code that only differ in type names.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2025-05-23 · Tags: `bit-depth`, `8bpc`, `16bpc`, `32bpc`, `pixel-format`, `conversion`*

---

## How can you efficiently convert an After Effects layer to OpenCV Mat format while handling pixel format conversion and ROI?

When converting a PF_LayerDef to cv::Mat, use PF_GET_PIXEL_DATA8 to get the pixel data pointer, then create a Mat with the proper stride parameter (rowbytes). Use cv::mixChannels with a fromTo mapping array to handle channel reordering (e.g., ARGB to BGRA: fromTo[] = {0, 3, 1, 2, 2, 1, 3, 0}). The Mat constructor accepts a step parameter for row stride: cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes). This allows proper handling of extent_hint for ROI-based processing.

```cpp
static cv::Mat ConvertLayerToMat(PF_LayerDef* layerDef, PF_InData* in_data) {
    PF_Pixel8* pixelData = nullptr;
    PF_GET_PIXEL_DATA8(layerDef, NULL, &pixelData);
    int width = layerDef->width;
    int height = layerDef->height;
    cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
    cv::Rect roi(
        in_data->extent_hint.left - in_data->output_origin_x,
        in_data->extent_hint.top - in_data->output_origin_y,
        width, height
    );
    if (roi.x < 0 || roi.y < 0 || roi.x + roi.width > width || roi.y + roi.height > height) {
        throw std::runtime_error("ROI goes outside the image dimensions.");
    }
    cv::Mat argbRoi = argb(roi);
    cv::Mat bgra(height, width, CV_8UC4);
    int fromTo[] = {0, 3, 1, 2, 2, 1, 3, 0};
    cv::mixChannels(&argbRoi, 1, &bgra, 1, fromTo, 4);
    return bgra;
}
```

*Tags: `opencv`, `pixel-format`, `memory`, `layer-checkout`, `output-rect`*

---
