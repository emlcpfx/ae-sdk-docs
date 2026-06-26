# Q&A: output-rect

**65 entries** tagged with `output-rect`.

---

## How do you cache frames in an After Effects effect plugin and use the cached result as input for processing the next frame?

This requires implementing frame caching within your plugin's render loop. You need to store the output of frame N, then retrieve it to use as input for frame N+1. This typically involves maintaining a persistent buffer or cache structure across multiple render calls. The exact implementation depends on whether you're using SmartFX or legacy plugins, but generally involves allocating memory to store intermediate results and managing that cache through the plugin's lifecycle to ensure the previous frame's output is available when processing subsequent frames.

*Tags: `caching`, `render-loop`, `memory`, `smartfx`, `output-rect`*

---

## How do you expand the output buffer with smartFX to prevent outlines from being cut off by the layer bounding box?

In smartFX, you need to modify the output rectangle to be larger than the input layer bounds. This is typically done by adjusting the output_rect parameter in your render function to expand beyond the original layer boundaries, allowing your outline effect to render fully without clipping.

*Tags: `smartfx`, `output-rect`, `render-loop`*

---

## What needs to be updated when using PF_COPY buffer operations?

When using PF_COPY or other buffer operations, the source and destination rectangles (src and dst rects) will need to be updated to reflect the operation.

*Tags: `mfr`, `output-rect`, `memory`*

---

## Can I directly modify the output PF_LayerDef pointer or do I need to use PF_NewWorld?

You can directly adjust the output PF_LayerDef data from the Render command without needing PF_NewWorld. However, you must not modify the layer settings (width, height, rowbytes) directly as these are defined by the host app. Instead of directly assigning mat.data to layerDef->data, you should copy the data from your matrix into layerDef->data using a loop while respecting the layerDef->rowbytes alignment. The proper approach is shown in the skeleton plugin example, which uses the Iterate Suite to write into output->data.

```cpp
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
```

*Tags: `mfr`, `memory`, `output-rect`, `pipl`*

---

## When do I need to implement PF_Cmd_FRAME_SETUP to customize output size?

You only need to implement PF_Cmd_FRAME_SETUP for non-SmartFX effects if your effect expands the output buffer size, such as with glow or drop shadow effects. For effects that maintain the same output dimensions as the input, this command is not necessary.

*Tags: `mfr`, `smartfx`, `output-rect`*

---

## How should I properly handle pixel data alignment when copying a matrix into a layer?

When copying pixel data into a PF_LayerDef, you must respect the layer's rowbytes alignment. Use a helper function like sampleIntegral32 that calculates the pixel address as (char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)). Ensure that mat.data has the same alignment as the layer rowbytes. You can optimize the copy by using memcpy() to copy entire rows at a time instead of individual pixels.

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data +
    (y * def.rowbytes) +
    (x * sizeof(PF_Pixel)));
}
```

*Tags: `memory`, `mfr`, `output-rect`*

---

## Does smart render provide benefits for effects that require the full image each time they are processed?

Smart render optimizes by only requesting pixels required for specific processing. For effects that require the full image each time, smart render's primary optimization of selective pixel fetching provides limited benefit since the effect must process the entire image anyway. However, smart render can still provide benefits in other scenarios such as avoiding redundant computation when the effect is cached or when only part of the output is needed by downstream effects.

*Tags: `smart-render`, `render-loop`, `output-rect`, `caching`*

---

## Does smart render provide any benefit for effects that require the full image each time it is processed?

Smart render helps by only getting pixels required for specific processing. For effects that require the full image each time it is processed, smart render would not provide additional benefit since the entire image must be obtained regardless.

*Tags: `smart-render`, `smartfx`, `output-rect`*

---

## How are transforms applied to input buffers in the dumb pipeline versus SmartFX?

In SmartFX, if the layer is continuously rasterised, you receive the input buffer with transforms already applied. If it's not continuously rasterised, the transforms are applied after your effect populates the output buffer. In the dumb pipeline, transforms are applied after the effect populates the output buffer.

*Tags: `smartfx`, `render-loop`, `output-rect`*

---

## What is the purpose of the rowbytes parameter in layer data?

Rowbytes exists as an optimization so that layers can be cropped without having to allocate extra memory. You can crop a layer by simply updating width and height and making the data pointer point to the new top-left pixel, leaving rowbytes untouched while keeping a reference to the old data for proper deallocation. AE does this often, such as when dragging a layer partially off the composition.

*Tags: `memory`, `output-rect`, `layer-checkout`*

---

## What is the purpose of rowbytes in After Effects layer data?

rowbytes exists as an optimization so that layers can be cropped without allocating extra memory. You can crop a layer by simply updating width and height and making the data pointer reference the new top-left pixel, leaving rowbytes untouched while keeping a reference to the old data for proper cleanup. After Effects does this often, such as when dragging a layer partially off the composition.

*Tags: `memory`, `layer-checkout`, `output-rect`, `caching`*

---

## Why might rowbytes differ in an 8-bit EffectWorld when the layer is cropped?

Rowbytes can differ based on cropping and alignment. For example, in an 8bpc EffectWorld that is cropped 25% horizontally, rowbytes could be calculated as 32*4*width, reflecting the stride needed for the allocated buffer dimensions rather than just the visible dimensions.

*Tags: `memory`, `output-rect`, `gpu`*

---

## Why does the precomp result show only a masked region as the result rect with everything else black?

When the precomp is created, the result rect is determined by the masked layer inside it. Since the precomp is full comp size but contains only a masked layer, only the area around the mask is included in the result rect, leaving everything else black.

*Tags: `mfr`, `output-rect`, `layer-checkout`*

---

## How can a plugin render output that extends beyond the bounds of its adjustment layer?

You need to modify the output_rect to be larger than the layer's bounds. By setting the output rectangle in your render function to extend beyond the adjustment layer's dimensions, you can render content (like text) that appears outside the layer's original boundaries while still being contained within the composition.

*Tags: `output-rect`, `render-loop`, `adjustment-layer`*

---

## How can you ensure byte order doesn't affect encoding data into pixel values?

sampleImage always returns RGBA. As long as you encode each color separately and limit it to one byte per color, byte order or even bit depth won't matter.

*Tags: `gpu`, `memory`, `output-rect`*

---

## Is it possible to create a multi-layer effect that discovers material properties from per-layer dummy effects and performs global illumination rendering while properly communicating dependencies to After Effects' caching system?

Tobias Fleischer shared that he attempted a plugin-per-layer approach years ago but abandoned it due to render order troubles. Instead, he recommended using a single plugin with multiple layer parameters that pull in pixels from different layers, apply material/lighting options, and composite everything within that single plugin. This approach worked well despite AE's rigid parameter interface being somewhat clunky. The viability of the frankensteined multi-layer approach with proper dependency tracking depends on whether the SDK allows explicit communication of implicit dependencies to AE's caching system, though existing solutions tend to consolidate the logic into a single effect rather than relying on scattered per-layer effects.

*Tags: `params`, `layer-checkout`, `caching`, `smart-render`, `output-rect`*

---

## How can you limit the output buffer to a specific region when Premiere doesn't support smartFX for transitions?

Since Premiere Pro does not support smartFX like After Effects does, you cannot use the smartFX output_rect mechanism to specify a smaller region of interest. When building transitions for Premiere, the plugin receives frames at the full sequence resolution even if the source footage is smaller. You need to work with the full buffer dimensions provided by Premiere's API rather than requesting a subset.

*Tags: `smartfx`, `premiere`, `output-rect`, `transition`*

---

## How do you expand the output buffer with smartFX to prevent outlines from being clipped by layer bounds?

When using smartFX in After Effects, if you're generating outlines around an alpha image and they're getting cut off by the layer bounding box, you need to expand the output buffer. This is typically done by modifying the output rectangle in your smartFX implementation to be larger than the layer's current bounds, ensuring that generated content (like outlines) isn't clipped.

*Tags: `smartfx`, `output-rect`, `mfr`, `render-loop`*

---

## How do you expand the result rectangle in After Effects plugin rendering?

To expand the in_result.result_rect and in_result.max_result_rect, modify them before calling UnionLRect. The typical pattern is to expand these rectangles first, then use UnionLRect to union in_result.max_result_rect with in_result.result_rect to make the result rect the same as max result rect, and finally union with the extra->output->result_rect.

```cpp
UnionLRect(&in_result.max_result_rect, &in_result.result_rect); // make result rect the same as max result rect
UnionLRect(&in_result.max_result_rect, &extra->output->result_rect);
```

*Tags: `output-rect`, `render-loop`, `aegp`*

---

## What needs to be updated when using PF_COPY or buffer operations in a plugin?

When performing buffer operations like PF_COPY, the source and destination rectangles (src and dst rects) must be updated to reflect the actual regions being operated on.

*Tags: `smartfx`, `output-rect`, `render-loop`*

---

## Why must X and Y coordinates be cast to FIX when using subpixel_sample_float, and what sampling method should be used for smoother displacement results?

When using the SamplingFloatSuite1()->subpixel_sample_float() function, coordinates are cast to FIX format (fixed-point representation) rather than remaining as floats. This is part of the After Effects API's internal coordinate system. The FIX format provides the necessary precision for the sampling algorithm while maintaining compatibility with AE's rendering pipeline. For smoother displacement results comparable to After Effects' native displacement or professional plugins, ensure you're using subpixel_sample_float with proper coordinate conversion, and consider whether your displacement values are being calculated with sufficient precision before being passed to the sampling function. The quality difference may also stem from how displacement offsets are computed rather than the sampling function itself.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
        INT2FIX(newX),
        INT2FIX(newY),
        &giP->inSamp_pb,
        &samplePixel));
```

*Tags: `sampling`, `displacement`, `mfr`, `output-rect`, `aegp`*

---

## How do you reproduce warp stabilizer banner messages in the viewport—using the Canvas suite or drawing directly on the rendered frame?

The user is asking about the proper approach to display banner messages similar to the warp stabilizer effect. This involves choosing between using the Canvas suite for viewport drawing versus drawing directly on the rendered frame output.

*Tags: `ui`, `render-loop`, `aegp`, `output-rect`*

---

## What is the best practice for handling GPU out-of-memory errors in After Effects plugins?

When a plugin runs out of GPU memory, it should return PF_Err_OUT_OF_MEMORY. During MFR (multi-frame rendering), After Effects may retry the frame with a lower thread count. For interactive scrubbing where the user isn't rendering from the queue, one approach is to use a C++ text library to render an error message directly onto the frame (e.g., "Out of GPU memory") so the user understands why a black frame appeared. However, there is currently no reliable way to distinguish between a render queue request and interactive timeline scrubbing to conditionally show warnings in out_data.

*Tags: `gpu`, `memory`, `mfr`, `render-loop`, `output-rect`*

---

## Do I need to use PF_NewWorld to modify the output PF_LayerDef in the Render function, or can I directly adjust its data?

You should not modify the layer settings (width, height, rowbytes) directly as they are defined by the host app. Instead of directly assigning mat.data to layerDef->data, you should copy the data from your matrix into the layerDef->data buffer using a loop while respecting layerDef->rowbytes. The proper approach is demonstrated in the skeleton plugin, which uses the Iterate Suite to write into output->data rather than allocating output itself.

```cpp
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
```

*Tags: `aegp`, `memory`, `layer-checkout`, `output-rect`*

---

## How do I properly sample and write pixels to a PF_LayerDef respecting rowbytes alignment?

Use pointer arithmetic to calculate pixel addresses based on rowbytes. The formula is: (char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)). This ensures proper memory alignment. Reference: https://ae-plugins.docsforadobe.dev/effect-details/tips-tricks.html?highlight=center%20of%20a%20pixel#sampling-pixels-at-x-y

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data +
    (y * def.rowbytes) +
    (x * sizeof(PF_Pixel)));
}
```

*Tags: `aegp`, `memory`, `output-rect`, `reference`*

---

## When do I need to use PF_Cmd_FRAME_SETUP to customize output size in AE plugins?

You only need to use PF_Cmd_FRAME_SETUP (https://ae-plugins.docsforadobe.dev/effect-basics/command-selectors.html) if your effect expands the output buffer size, such as glow or drop shadow effects. For effects that maintain the same output dimensions as input, this step is not necessary.

*Tags: `aegp`, `output-rect`, `reference`*

---

## How can you efficiently copy pixel data from an OpenCV Mat to an After Effects layer?

Use the Iterate Suite and copy entire rows at a time with memcpy() instead of copying pixel-by-pixel. Respect the layer's rowBytes value when advancing pointers. Ensure both buffers have matching pixel format (ARGB for AE, BGRA for OpenCV by default) and are uchar type for proper memory alignment. Advance both the OpenCV buffer and layer buffer by their respective rowByte values during the copy operation.

```cpp
static PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
    return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        cv::Vec4b* matRow = mat.ptr<cv::Vec4b>(y);
        PF_Pixel* aeRow = (PF_Pixel*)((char*)layer->data + (y * layer->rowbytes));
        memcpy(aeRow, matRow, layer->width * sizeof(PF_Pixel));
    }
}
```

*Tags: `memory`, `caching`, `output-rect`, `render-loop`*

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

## Does smart render provide benefits for effects that require the full image for processing?

Smart render helps by only requesting pixels required for specific processing. For effects that require the full image each time they are processed (when visible), smart render may not provide additional optimization benefits since the entire image must be fetched anyway. The advantage of smart render is primarily for effects that can work on partial regions of interest.

*Tags: `smart-render`, `render-loop`, `output-rect`, `optimization`*

---

## How does transform application differ between SmartFX and standard pipelines in After Effects?

In SmartFX, if a layer is continuously rasterized, you receive the input buffer with transforms already applied. If not continuously rasterized, the transforms are applied after your effect populates the output buffer. This means you won't know if transforms changed in the standard pipeline unless using SmartFX.

*Tags: `smartfx`, `render-loop`, `mfr`, `output-rect`*

---

## What is the purpose of rowbytes in After Effects layer data, and how does it enable memory optimization?

Rowbytes exists as an optimization so that layers can be cropped without allocating extra memory. You can crop a layer by updating width and height and making the data pointer reference the new top-left pixel, while leaving rowbytes unchanged and maintaining a reference to the original data for proper deallocation. After Effects uses this optimization frequently, such as when dragging a layer partially off the composition. Rowbytes is calculated as 4*width for 32-bit data, but may be larger if a layer is loaded and moved partially off-comp; however, purging cache typically resets it back to 4*width.

*Tags: `memory`, `caching`, `output-rect`, `optimization`*

---

## What were the root causes of bugs in the FastBoxBlur implementation?

Two critical bugs were identified in FastBoxBlur: (1) Incorrect bit depth detection causing memory corruption, and (2) Improper stride handling for padded buffers. The solution involved fixing bit depth detection to use rowbytes calculation and implementing proper stride handling throughout the blur algorithm.

*Tags: `memory`, `debugging`, `render-loop`, `output-rect`*

---

## How can a plugin render content that extends beyond the adjustment layer bounds?

When rendering content that exceeds the layer boundaries (such as text on an adjustment layer smaller than the composition), you need to use the output_rect parameter to specify the actual rendering region. By setting output_rect to encompass the full area where your content will be drawn, rather than constraining it to the layer bounds, the plugin can render beyond the adjustment layer's dimensions and display the full content within the composition.

*Tags: `output-rect`, `render-loop`, `aegp`, `ui`*

---

## Can you pass text output from an effect to a text layer instead of rendering it yourself?

Yes, you can encode text data into pixels and pass it to a text layer. One approach is to use dead pixels (out-of-frame pixels or pixels with alpha of 0) to encode the string data in RGB color values. This allows an effect that produces both pixels and text to delegate text rendering to a native text layer rather than rendering it directly.

*Tags: `ui`, `params`, `aegp`, `output-rect`*

---

## What alignment guarantees does PF_LayerDef::rowbytes provide in After Effects?

PF_LayerDef::rowbytes is guaranteed to always be a multiple of sizeof(PF_Pixel8), which is 4 bytes. 16-byte alignment is not always guaranteed. The rowbytes value is arbitrary and your code should be prepared to deal with any stride. Additionally, a buffer might be a sub-reference to another buffer, so you should not write into the bytes outside your image reference frame into the rowbytes gutter.

*Tags: `memory`, `params`, `aegp`, `output-rect`*

---

## How can you specify a custom output region in Premiere transitions when smartFX is not available?

Premiere does not support smartFX, which means you cannot use the smart render feature to request only a specific region of the buffer. When making transitions, Premiere will provide frames at the sequence resolution regardless of the actual footage size. This is a limitation of Premiere's plugin architecture compared to After Effects, where smartFX allows you to define output rectangles and optimize rendering to only the needed buffer region.

*Tags: `smartfx`, `premiere`, `output-rect`, `render-loop`*

---

## How can I avoid noise artifacts when After Effects automatically converts 16/32-bit project input to 8-bit for my plugin?

Rather than relying on After Effects' automatic conversion, you should modify your plugin to accept 16 and 32 bits per channel (bpc) inputs and perform the conversion to 8-bit yourself. This approach provides several benefits: users will benefit from having the output remain in 16/32 bpc, you will avoid the warning sign next to your plugin in the UI, and you'll have full control over the conversion process to avoid dithering artifacts introduced by AE's automatic conversion.

*Tags: `params`, `output-rect`, `aegp`, `debugging`*

---

## How do you properly expand the buffer in SmartFX to avoid asymmetric expansion?

When expanding the buffer via SmartFX, put the expanded buffer rect in `extra->output->result_rect` rather than relying on `max_result_rect`. After expanding the input buffer (`in_result.max_result_rect`), put the expanded result in `extra->output->max_result_rect`. After Effects uses the rect from `extra->output->result_rect` for the render call. The key is deciding what the expansion is relative to: the input buffer as-is (cropped by mask or comp bounds), the original layer size, or the max rect of the input. The expanded input buffer should be centered in the intermediate buffer that matches the expanded output.

```cpp
PF_LRect expanded_rect = in_result.max_result_rect;
expanded_rect.left -= expansion;
expanded_rect.top -= expansion;
expanded_rect.right += expansion;
expanded_rect.bottom += expansion;
extra->output->result_rect = expanded_rect;
extra->output->max_result_rect = expanded_rect;
```

*Tags: `smartfx`, `buffer-expansion`, `smart-render`, `output-rect`*

---

## Is there a sample plugin that demonstrates resizing the output frame?

The 'Resizer' SDK sample project is the recommended reference for learning how to resize output frames in After Effects plugins.

*Tags: `reference`, `sdk`, `output-rect`*

---

## How can I create an After Effects effect that only provides settings without changing the layer visually?

There are several approaches: (1) Use AE's utils->Copy() function to copy the input buffer to the output buffer, which is fast and straightforward. (2) Set the PF_OutFlag_AUDIO_EFFECT_ONLY flag to prevent AE from calling your effect for image rendering—it will only be called for audio, which you must implement by copying input to output. This is faster than copying the image buffer. (3) Try setting the output rect size to 0 or negative during pre-render to make AE skip rendering, though this approach has inconsistent results. The simplest approach is using utils->Copy() to pass through the input unchanged.

```cpp
static PF_Err
Render(	PF_InData		*in_data,
		PF_OutData		*out_data,
		PF_ParamDef		*params[],
		PF_LayerDef		*output)
{
	PF_Err	err	= PF_Err_NONE;
	// Use utils->Copy() to copy input to output
	return err;
}
```

*Tags: `aegp`, `render-loop`, `memory`, `output-rect`*

---

## Why does a checked-out layer get resized and positioned incorrectly when the effect layer is moved and re-rendered?

The issue occurs because when a layer is moved partially outside the composition, After Effects optimizes rendering by asking plugins to render only the visible rectangle. If you checkout the full layer and copy it to a smaller output buffer, the copy operation resizes the source buffer to match the destination buffer size, causing the layer to be squeezed. The solution is to calculate the offset between the layer position and composition center, then use composite_rect() instead of copy() to properly position the checked-out layer in the output buffer while respecting the output_worldP->origin_x and output_worldP->origin_y values.

```cpp
// Calculate offset
A_long x = (A_long)(effectLayerData.position.x - in_data->width / 2);
A_long y = (A_long)(effectLayerData.position.y - in_data->height / 2);

// Use composite_rect instead of copy, and check origin values
err = worldTransformSuite->composite_rect(in_data->effect_ref,
  in_data->quality,
  PF_MF_Alpha_STRAIGHT,
  in_data->field,
  &checkout_layer_worldP->extent_hint,
  checkout_layer_worldP,
  &compMode,
  NULL,
  output_worldP);
```

*Tags: `layer-checkout`, `aegp`, `render-loop`, `output-rect`*

---

## How can I display text in an After Effects plugin render buffer?

After Effects' API doesn't offer built-in text generating functions for the render buffer (unlike the UI buffer which has the drawbot suite). However, you can fill the render buffer using OS-native text tools: use GDI+ on Windows or Quartz on macOS to generate text in a native OS buffer, then copy that content back to After Effects' render buffer.

*Tags: `ui`, `windows`, `macos`, `render-loop`, `output-rect`*

---

## Why isn't drawing to a newly created Effect World displaying correctly when using transform_world?

When drawing pixel-by-pixel to a newly created Effect World using manual pixel manipulation, ensure you are drawing directly to the output parameter rather than an intermediate world buffer. The issue in this case was a logic error in the broader code context, not the pixel-drawing approach itself. The technique of using sampleIntegral32() to get a pointer to a pixel and then modifying its RGBA values is correct, but verify the world is being used in the right scope and that PF_Fill works as a control test to confirm the world itself is valid.

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

static PF_Err drawPixel32(int x, int y, int size, PF_EffectWorld *output) {
  PF_Err err = PF_Err_NONE;
  int u, v;
  for (u=0; u < size; u++) {
    for (v=0; v < size; v++) {
      PF_Pixel *myPixel = sampleIntegral32(*output, x+u, y+v);
      myPixel->red = 0;
      myPixel->green = 0;
      myPixel->blue = 255;
      myPixel->alpha = 255;
    }
  }
  return err;
}
```

*Tags: `aegp`, `output-rect`, `memory`, `debugging`*

---

## What is the correct way to bypass rendering in an After Effects effect plugin?

To bypass rendering in an AE effect plugin, you have a few options: (1) Try setting the output rectangle during pre-render to a 0-sized rect or a rect where the right value is smaller than the left value to tell AE to skip the render, though this approach is not well-documented and may require experimentation. (2) The more reliable and practical approach is to manually copy the input buffer to the output buffer in your render function. According to community experts, the overhead of copying is negligible even when stacking multiple such effects, making this a pragmatic solution that works reliably on both audio and non-audio layers.

*Tags: `render-loop`, `output-rect`, `aegp`, `reference`*

---

## How do you copy pixel data from a loaded BMP file into a PF_EffectWorld?

Once you have loaded the BMP pixel data and allocated an effect world, you can access the effect world's pixel data using pointer arithmetic: PF_Pixel *p = world->data + x + y * (world->rowbytes / sizeof(PF_Pixel)). Then copy individual color channels from the BMP data to the effect world pixel, such as p->red = bmpPixel[0], p->green = bmpPixel[1], p->blue = bmpPixel[2], and p->alpha = 255. Be aware that the channel order and bit depth may differ between the BMP format and the effect world format, so value conversion may be required.

```cpp
PF_Pixel *p = world->data + x + y * (world->rowbytes / sizeof(PF_Pixel));
p->red = bmpPixel[0];
p->green = bmpPixel[1];
p->blue = bmpPixel[2];
p->alpha = 255;
```

*Tags: `memory`, `mfr`, `output-rect`, `aegp`*

---

## How can I create off-screen image buffers in After Effects SDK and draw to them?

It is possible to create off-screen image buffers in After Effects SDK. You can use intermediate buffers by working with the output buffer handed to the effect. The buffer has a 'data' pointer for the base address of image pixels and a 'rowbytes' variable for the number of bytes per row (stride). You can draw to custom buffers and then copy the result into the output buffer. After Effects provides native buffers called 'worlds' in the SDK documentation that can be used as intermediates.

*Tags: `sdk`, `memory`, `render-loop`, `output-rect`*

---

## What do the origin_x and origin_y fields in PF_EffectWorld represent when checking out a shape layer?

The origin_x and origin_y fields in the PF_EffectWorld structure represent the offset position of the checked-out shape within its buffer. When checking out a shape layer, the resulting buffer is the size of the shape itself (not the layer), and the origin fields indicate where that shape is positioned relative to the layer's coordinate space.

*Tags: `layer-checkout`, `shape`, `output-rect`, `sdk`*

---

## Why does applying a buffer-expanding effect like blur after a SmartFX effect cause an offset in the output?

When using SmartFX to expand the output buffer to the full composition size, subsequent buffer-expanding effects (such as blur or drop shadow) can alter the output origins. To fix this, you need to explicitly set the output->origin_x and output->origin_y parameters in your SmartFX preRender function. This ensures that downstream effects don't inadvertently offset your effect's output.

```cpp
UnionLRect(&req.rect, &extra->output->result_rect);
UnionLRect(&req.rect, &extra->output->max_result_rect);
// Also set:
extra->output->origin_x = /* appropriate value */;
extra->output->origin_y = /* appropriate value */;
```

*Tags: `smartfx`, `buffer`, `output-rect`, `sdk`*

---

## How do I fix the 'could not convert Unicode characters' error when setting an output file path in After Effects plugins?

Use A_UTF16Char instead of A_char for the output path parameter. The correct approach is to cast a wide string literal to A_UTF16Char and pass it to AEGP_SetOutputFilePath. Example: const A_UTF16Char *outPath = reinterpret_cast<const A_UTF16Char *>(L"C:\\whee.mov"); ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(0, 0, (A_char*)outPath));

```cpp
const A_UTF16Char *outPath = reinterpret_cast<const A_UTF16Char *>(L"C:\\whee.mov");
ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(0, 0, (A_char*)outPath));
```

*Tags: `aegp`, `scripting`, `unicode`, `output-rect`*

---

## Why is motion blur incorrect when using PF_TransformWorld with fast rotation?

When using PF_TransformWorld with only two matrices (previous and current), motion blur artifacts can occur during fast rotation, causing particles to appear size-limited. The solution is to use multiple intermediate matrices for in-between values rather than just the start and end positions. Additionally, ensure that the PF_Rect *dest_rect is expanded to encompass the full area of all rotated samples to capture the complete motion blur region.

```cpp
PF_Err transform_world (
  PF_InData *in_data,
  PF_Quality quality,
  PF_ModeFlags m_flags,
  PF_Field field,
  const PF_EffectWorld *src_world,
  const PF_CompositeMode *comp_mode,
  const PF_MaskWorld *mask_world0,
  const PF_FloatMatrix *matrices,
  A_long num_matrices,
  Boolean src2dst_matrix,
  const PF_Rect *dest_rect,
  PF_EffectWorld *dst_world);
```

*Tags: `mfr`, `transform`, `motion-blur`, `rendering`, `output-rect`*

---

## Why does my After Effects plugin display a white screen after parameter changes, requiring a purge to render correctly?

This is typically a buffer clearing issue. You need to fill the output buffer with empty pixels using PF_FillMatteSuite2()->fill(). Another common cause is accidentally writing data to the input buffer and then using it as an intermediate processing buffer, which causes After Effects to cache the tampered data. Always clear AEGP worlds before using them, as After Effects may reuse the same RAM block from a previous world call, which could contain junk data or previous frame data.

```cpp
PF_FillMatteSuite2()->fill()
```

*Tags: `smart-render`, `memory`, `caching`, `aegp`, `output-rect`*

---

## How can I get composition dimensions from within an After Effects plugin?

To get composition dimensions, use the following AEGP API sequence: (1) AEGP_GetEffectLayer() to get the effect layer, (2) AEGP_GetLayerParentComp() to get the parent composition, (3) AEGP_GetItemFromComp() to get the item from the composition, (4) AEGP_GetItemDimensions() to retrieve the dimensions. Note that this returns the comp size at full resolution, so you may need to account for downsample factors depending on your use case.

*Tags: `aegp`, `params`, `output-rect`*

---

## Why is my image appearing smeared when passing vector<int> pixel data through an iteration function in After Effects?

The issue is likely related to incorrect pixel data ordering or indexing assumptions. The user was storing pixels in one order (x0y0, x0y1, x0y2... x1y0, x1y1, x1y2...) but the iteration function was expecting them in a different order (x0y0, x1y0, x2y0... x0y1, x1y1, x2y1...), causing the image to appear distorted or on its side. To debug, simplify the project to verify that a basic pixel copy (outP = *inP) works correctly first, then incrementally add complexity to identify where the indexing breaks down.

```cpp
static PF_Pixel8
*getXY(PF_EffectWorld &def, int x , int y){
  return (PF_Pixel*)def.data + y * (def.rowbytes / sizeof(PF_Pixel)) + x;
}
```

*Tags: `render-loop`, `memory`, `debugging`, `output-rect`*

---

## How should pixel data be correctly indexed when extracting values from a PF_EffectWorld and storing them in a vector for later retrieval?

When extracting pixel data from a PF_EffectWorld, use proper pointer arithmetic accounting for rowbytes: cast the data pointer to PF_Pixel, then offset by (y * (rowbytes / sizeof(PF_Pixel)) + x). Store pixels in a consistent order (RGBA for each pixel sequentially). When retrieving from the vector in the iteration function, ensure the indexing matches: for pixel at (x, y), access indices starting at ((y * width + x) * 4) for RGBA values in order. The key is maintaining consistency between storage order and retrieval order.

```cpp
// Storage in vector
for(int i = 0; i < tInfo.width; i++){
  for(int j = 0; j < tInfo.height; j++){
    PF_Pixel8 currentPixel = *getXY(*tInfo.input, i, j);
    data.push_back(currentPixel.red);
    data.push_back(currentPixel.green);
    data.push_back(currentPixel.blue);
    data.push_back(currentPixel.alpha);
  }
}

// Retrieval in iteration function
int idx = (yL * tInfo.width + xL) * 4;
outP->red = tInfo->data[idx];
outP->green = tInfo->data[idx + 1];
outP->blue = tInfo->data[idx + 2];
outP->alpha = tInfo->data[idx + 3];
```

*Tags: `memory`, `render-loop`, `output-rect`, `debugging`*

---

## How can I resize the output buffer in an After Effects plugin and determine the correct size before rendering?

You cannot access the input image during Frame_Setup, but you have two options: (1) Save necessary data during the render call and trigger a re-render using PF_ForceRerender from AdvItemSuite, during which you can change the output size; or (2) Use the layer dimensions from in_data->width and in_data->height, which represent the unaffected layer dimensions. For more advanced control, consider migrating to SmartFX, which uses pre-render calls instead of Frame_Setup and supports 32-bit color depth.

*Tags: `output-rect`, `frame-setup`, `render-loop`, `aegp`*

---

## Can I change the output size during a re-render in After Effects plugins?

Yes, you can change the output size during a re-render. When you call for a re-render using PF_ForceRerender, you receive a Frame_Setup call first, during which you can modify the size even if the frame was previously rendered. This allows you to adjust output dimensions based on calculations from the previous render pass.

*Tags: `output-rect`, `frame-setup`, `aegp`, `render-loop`*

---

## How can I access and modify pixels in DrawSparseFrame of an AEIO without using Iterate8Suite?

You can create a PF_InData struct and fill it with whatever data you have available, which should allow you to use the iterate suites. Alternatively, you can iterate through the buffer's pixels directly using nested loops to access individual pixels via sampleIntegral32. However, be careful with coordinate ordering—ensure you're passing coordinates in the correct order (x,y) rather than swapped (y,x), and account for thumbnail resolution differences which can cause crashes or unexpected behavior.

```cpp
for (A_long i = 0; i < 1000; i++)
for (A_long j = 0; j < 1000; j++){
  PF_Pixel *pixel = sampleIntegral32(wP, i, j);
  pixel->alpha = PF_MAX_CHAN8;
  pixel->red = PF_MAX_CHAN8;
  pixel->green = PF_MAX_CHAN8;
  pixel->blue = PF_MAX_CHAN8;
}
```

*Tags: `aeio`, `aegp`, `memory`, `output-rect`, `debugging`*

---

## How can I iterate through 16-bit and 32-bit per channel pixels directly without using the iterate suites?

You can iterate 16bpc and 32bpc pixels directly by casting the buffer data to the appropriate pixel type. For 16-bit pixels, use: `PF_Pixel16 pix = (PF_Pixel16)buffer->data + xCount + yCount * (buffer->rowbytes / sizeof(Pixel16));`. Convert similarly for 32-bit pixels. The rest of the process is the same as in the CCU sample. Replace all PF_Pixel8 references with PF_Pixel16 or PF_Pixel32 as needed, and update constants like PF_MAX_CHAN8 to PF_MAX_CHAN16 (or 1.0f for 32-bit). For 16-bit support, set PF_OutFlag_DEEP_COLOR_AWARE in out_data->out_flags. For 32-bit support, use SmartFX (see the smartyPants sample).

```cpp
PF_Pixel16 pix = (PF_Pixel16)buffer->data + xCount + yCount * (buffer->rowbytes / sizeof(Pixel16));
```

*Tags: `mfr`, `memory`, `render-loop`, `output-rect`*

---

## How do you get the correct coordinate in layer coordinate system when using the iterate function?

When a layer is partially off the comp, After Effects may give you a reduced buffer. The effectWorld structure shows you the offset values needed to correctly map pixel coordinates. When iterating, you need to account for this offset by using outPixel=GetColor(x+offsetX,y+offsetY). Additionally, there is an "iterate offset function" available in the SDK that can help handle this automatically. The input and output buffers for non-collapsed layers are in layer coordinates, but AE diminishes the buffer at 20% increments based on what part of the layer is out of sight, not just individual pixels.

```cpp
outPixel=GetColor(x+offsetX,y+offsetY)
```

*Tags: `layer-checkout`, `iterate`, `output-rect`, `sdk`*

---

## How can I check out only a portion of a frame at a different time instead of the entire frame?

You can use checkout_layer_pixels() which takes a request info structure as an argument that allows you to request a smaller rectangle. Alternatively, use the render suite to request the source item directly and set an ROI (region of interest) to get only the part you need. The render suite method doesn't hint AE in advance about your intentions, so AE won't cache items in advance—test both methods to see which is faster for your use case, as getting a potentially cached full frame might sometimes be faster than requesting a partial frame without caching.

*Tags: `layer-checkout`, `aegp`, `caching`, `output-rect`, `memory`, `render-loop`*

---

## How can I draw anti-aliased shapes and text directly into an After Effects effect's output buffer?

DrawBot is not suitable for rendering into the output buffer—it only operates on AE's internal interface graphics buffers. Instead, use OS-native drawing tools: on macOS, use Quartz (Core Graphics) APIs like CGBitmapContextCreate(), CGContextShowTextAtPoint(), and CGContextDrawPath(); on Windows, use GDI+ or equivalent APIs. Create an OS graphics context in memory allocated via the Memory Suite, draw your shapes and text to that context, then copy the pixel data from the OS buffer to the effect's output buffer by locking the memory handle and iterating through pixels. See the CCU (Custom Color UI) sample code in the SDK for an example of accessing the output buffer directly in RAM.

```cpp
osBufferBaseAddress = suites.HandleSuite1()->host_lock_handle(osBufferMemHandle);
// Copy pixel data from OS context to output buffer
// Then unlock and free the OS graphics context
```

*Tags: `output-rect`, `macos`, `windows`, `drawbot`, `reference`*

---

## Is it possible to create a custom output device in After Effects beyond the built-in options?

Yes, it is possible to create a custom output device in After Effects. The recommended approach is to base your plugin on the EMP (External Monitor Preview) sample plugin that is included in the SDK. This sample demonstrates how to implement custom output device functionality.

*Tags: `output-rect`, `aegp`, `reference`, `deployment`*

---

## What sample plugin should be used as a reference for building custom output device plugins?

The EMP (External Monitor Preview) sample plugin is the recommended reference for creating custom output devices in After Effects. This sample has been discussed in the Adobe forums and is included in the SDK documentation. On macOS, developers may also want to examine the Quicktime VOUT sample for additional reference.

*Tags: `output-rect`, `aegp`, `reference`, `macos`, `open-source`*

---

## How can I create an After Effects plugin that samples the current frame and displays information graphically in a separate window?

You should build an AEGP (After Effects General Plugin) rather than an effect plugin. Use the 'panelator' sample as a base to create a dockable panel for displaying your graphical output. To access frame data, you have two approaches: (1) Use the 'EMP' (external monitor preview) sample which provides the MyBlit() function to receive the currently viewed composition buffer, registered during entryPoint(). (2) Use AEGP functions to render project items on demand: AEGP_GetMostRecentlyUsedComp, AEGP_RenderAndCheckoutFrame, and AEGP_GetReceiptWorld. The second approach can leverage AE's render cache to avoid re-rendering if the frame has already been processed.

*Tags: `aegp`, `ui`, `output-rect`, `reference`*

---

## What is the EMP sample and how does it deliver frame data to plugins?

EMP (External Monitor Preview) is a sample plugin template that demonstrates how to receive image data from After Effects. It uses the MyBlit() function as the callback where AE delivers the currently viewed composition buffer to the plugin. This function is registered during the entryPoint() function and is useful for plugins that need real-time access to the composition being viewed.

*Tags: `aegp`, `output-rect`, `reference`, `sample`*

---
