# Buffer Expansion in After Effects SmartFX Plugins

## Table of Contents
1. [Overview](#overview)
2. [The Problem](#the-problem)
3. [How Buffer Expansion Works](#how-buffer-expansion-works)
4. [Critical Concepts](#critical-concepts)
5. [Implementation Guide](#implementation-guide)
6. [Common Pitfalls](#common-pitfalls)
7. [Real-World Example](#real-world-example)
8. [Debugging Tips](#debugging-tips)

---

## Overview

Buffer expansion allows After Effects plugins to return pixels beyond the original layer boundaries. This is essential for effects like:
- **Edge extension** - Extending colors beyond alpha boundaries
- **Blurs** - Gaussian/box blurs that extend beyond input
- **Glows** - Light effects that extend outward
- **Shadows** - Drop shadows extending beyond layer
- **Morphological operations** - Dilate/erode that grow/shrink masks

Without buffer expansion, these effects get **cropped at layer/mask boundaries**, creating hard edges and incorrect results.

---

## The Problem

### Symptom: Cropping at Boundaries

When you apply an edge-extending effect to a masked layer:
```
Input layer (masked):     Desired output:          What you get (WRONG):
+---------------+          +---------------+          +---------------+
|   +===+       |          |   +===+       |          |   +===+       |
|   |img|       |   ->     |  +' img '+    |   BUT:   |   |img|       |
|   +===+       |          | +'       '+   |          |   +===+       |
|               |          |+'         '+  |          |               |
+---------------+          +---------------+          +---------------+
                           Extended beyond mask      Cropped!
```

The extension gets **clipped** because:
1. Output buffer = Input size
2. Processing can't write beyond buffer bounds
3. Result is cropped to original dimensions

---

## How Buffer Expansion Works

### Two-Stage Process

#### Stage 1: PreRender (Request Expansion)
```cpp
// Tell AE we need LARGER output buffers
static PF_Err PreRender(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_PreRenderExtra *extra)
{
    // 1. Calculate how much expansion we need
    float max_extension = CalculateMaxExtension(params);
    A_long expansion = (A_long)(max_extension + 10); // +10 safety margin

    // 2. Expand OUTPUT rectangles
    extra->output->result_rect = in_result.result_rect;
    extra->output->result_rect.left   -= expansion;
    extra->output->result_rect.right  += expansion;
    extra->output->result_rect.top    -= expansion;
    extra->output->result_rect.bottom += expansion;

    extra->output->max_result_rect = in_result.max_result_rect;
    extra->output->max_result_rect.left   -= expansion;
    extra->output->max_result_rect.right  += expansion;
    extra->output->max_result_rect.top    -= expansion;
    extra->output->max_result_rect.bottom += expansion;

    // 3. CRITICAL: Tell AE we're returning extra pixels
    extra->output->flags = PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS;

    return PF_Err_NONE;
}
```

**What this does:**
- Requests output buffers LARGER than input
- AE allocates expanded `output_worldP`
- Input still comes in at original size
- You must position input within expanded output

#### Stage 2: SmartRender (Handle Expansion)
```cpp
static PF_Err SmartRender(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_SmartRenderExtra *extra)
{
    // Input is ORIGINAL size
    PF_EffectWorld *input_worldP  = &extra->input->worldP;   // e.g., 1920x1080

    // Output is EXPANDED size
    PF_EffectWorld *output_worldP = &extra->output->worldP;  // e.g., 2120x1280

    // WHERE does input sit in the expanded output?
    // AE provides output_origin_x/y to tell us!
    A_long input_x = in_data->output_origin_x;  // e.g., 100
    A_long input_y = in_data->output_origin_y;  // e.g., 100
}
```

> **Note:** In SmartRender, use `extra->cb->checkout_layer_pixels()` and `extra->cb->checkout_output()` to get world pointers.

---

## Critical Concepts

### 1. Output Origin Coordinates

**`in_data->output_origin_x` and `in_data->output_origin_y`** tell you where the original input layer sits within the expanded output buffer.

```
Expanded output buffer (2120x1280):
+-------------------------------------+
| <-100px->                           |
|   v100px                            |
|   +-------------------+ <- Input    |
|   |                   |   (1920x1080)|
|   |  Original Input   |             |
|   |                   |             |
|   +-------------------+             |
|                                     |
+-------------------------------------+
   output_origin = (100, 100)
```

### 2. The Output Origin Gotcha

**CRITICAL ISSUE**: When you expand output rects in comp-space coordinates, AE sometimes sets `output_origin_x/y` to **ZERO** instead of the calculated offset!

```cpp
// What you'd EXPECT:
// output_origin_x = (output_width - input_width) / 2 = 100

// What AE ACTUALLY gives you:
// output_origin_x = 0
// output_origin_y = 0
```

**Solution: Calculate offset manually**
```cpp
// Calculate expected offset
A_long offset_x = (output_worldP->width - input_worldP->width) / 2;
A_long offset_y = (output_worldP->height - input_worldP->height) / 2;

// Use output_origin if non-zero, otherwise use calculated offset
A_long actual_offset_x = (in_data->output_origin_x != 0)
                         ? in_data->output_origin_x
                         : offset_x;
A_long actual_offset_y = (in_data->output_origin_y != 0)
                         ? in_data->output_origin_y
                         : offset_y;
```

### 3. Processing Strategy: Output-Sized Buffers

**WRONG APPROACH**:
```cpp
// DON'T allocate temp buffers at INPUT size
PF_NewWorld(input_worldP->width, input_worldP->height, temp1);
// Process at input size
ProcessEffect(temp1);  // Extension gets CLIPPED here!
// Copy to expanded output
PF_COPY(temp1, output_worldP);  // Too late - already clipped!
```

**CORRECT APPROACH**:
```cpp
// Allocate temp buffers at OUTPUT size
PF_NewWorld(output_worldP->width, output_worldP->height, temp1);

// Clear expanded areas
PF_Pixel clear = {0, 0, 0, 0};
fillMatteSuite->fill(temp1, NULL, &clear);

// Copy input to CENTER of temp buffer
PF_Rect dest_rect;
dest_rect.left   = actual_offset_x;
dest_rect.top    = actual_offset_y;
dest_rect.right  = dest_rect.left + input_worldP->width;
dest_rect.bottom = dest_rect.top + input_worldP->height;
PF_COPY(input_worldP, temp1, NULL, &dest_rect);

// Now process at OUTPUT size - extension has room to grow!
ProcessEffect(temp1);  // Extension fills expanded areas

// Copy result to output
PF_COPY(temp1, output_worldP, NULL, NULL);
```

---

## Implementation Guide

### Step 1: PreRender - Request Expansion

```cpp
static PF_Err PreRender(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_PreRenderExtra *extra)
{
    PF_Err err = PF_Err_NONE;
    PF_RenderRequest req = extra->input->output_request;
    PF_CheckoutResult in_result;

    // Checkout input layer to get its rect
    ERR(extra->cb->checkout_layer(
        in_data->effect_ref,
        LAYER_INPUT,
        LAYER_INPUT,
        &req,
        in_data->current_time,
        in_data->time_step,
        in_data->time_scale,
        &in_result));

    // Get parameters to calculate expansion
    PF_ParamDef erosion_param, blur_param;
    AEFX_CLR_STRUCT(erosion_param);
    AEFX_CLR_STRUCT(blur_param);

    ERR(PF_CHECKOUT_PARAM(in_data, PARAM_EROSION,
                          in_data->current_time,
                          in_data->time_step,
                          in_data->time_scale,
                          &erosion_param));
    ERR(PF_CHECKOUT_PARAM(in_data, PARAM_BLUR,
                          in_data->current_time,
                          in_data->time_step,
                          in_data->time_scale,
                          &blur_param));

    float erosion = (float)erosion_param.u.fs_d.value;
    float blur_radius = (float)blur_param.u.fs_d.value;

    // Calculate total expansion needed
    // For edge extension: erosion + max blur + safety margin
    float max_extension = erosion + blur_radius;
    A_long expansion = (A_long)(max_extension + 10.0f); // +10 safety

    // Expand output rectangles ON ALL SIDES
    extra->output->result_rect = in_result.result_rect;
    extra->output->result_rect.left   -= expansion;
    extra->output->result_rect.right  += expansion;
    extra->output->result_rect.top    -= expansion;
    extra->output->result_rect.bottom += expansion;

    extra->output->max_result_rect = in_result.max_result_rect;
    extra->output->max_result_rect.left   -= expansion;
    extra->output->max_result_rect.right  += expansion;
    extra->output->max_result_rect.top    -= expansion;
    extra->output->max_result_rect.bottom += expansion;

    // CRITICAL: Set flag to tell AE we're returning extra pixels
    extra->output->flags = PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS;

    // SmartFX auto-checks-in params, no need for PF_CHECKIN_PARAM

    return err;
}
```

### Step 2: SmartRender - Position Input in Expanded Output

```cpp
static PF_Err SmartRender(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_SmartRenderExtra *extra)
{
    PF_Err err = PF_Err_NONE;

    // Get input and output buffers
    PF_EffectWorld *input_worldP, *output_worldP;
    ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, LAYER_INPUT, &input_worldP));
    ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));

    // Calculate offset for centering input in expanded output
    A_long offset_x = (output_worldP->width - input_worldP->width) / 2;
    A_long offset_y = (output_worldP->height - input_worldP->height) / 2;

    // Use output_origin if provided (non-zero), otherwise use calculated
    A_long actual_offset_x = (in_data->output_origin_x != 0)
                             ? in_data->output_origin_x
                             : offset_x;
    A_long actual_offset_y = (in_data->output_origin_y != 0)
                             ? in_data->output_origin_y
                             : offset_y;

    // Log for debugging
    std::stringstream log;
    log << "Buffer sizes - Input: " << input_worldP->width << "x" << input_worldP->height
        << ", Output: " << output_worldP->width << "x" << output_worldP->height
        << ", Origin: (" << in_data->output_origin_x << "," << in_data->output_origin_y << ")"
        << ", Calculated: (" << offset_x << "," << offset_y << ")"
        << ", Using: (" << actual_offset_x << "," << actual_offset_y << ")";
    // Your logging function here

    // Allocate temp buffers at OUTPUT size
    PF_EffectWorld *temp1_worldP = nullptr, *temp2_worldP = nullptr;
    AEFX_SuiteScoper<PF_WorldSuite2> wsP(in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);

    ERR(wsP->PF_NewWorld(in_data->effect_ref,
                         output_worldP->width,   // OUTPUT width
                         output_worldP->height,  // OUTPUT height
                         PF_NewWorldFlag_CLEAR_PIXELS,
                         temp1_worldP));

    ERR(wsP->PF_NewWorld(in_data->effect_ref,
                         output_worldP->width,
                         output_worldP->height,
                         PF_NewWorldFlag_CLEAR_PIXELS,
                         temp2_worldP));

    // Copy input to center of temp buffer
    PF_Rect dest_rect;
    dest_rect.left   = actual_offset_x;
    dest_rect.top    = actual_offset_y;
    dest_rect.right  = dest_rect.left + input_worldP->width;
    dest_rect.bottom = dest_rect.top + input_worldP->height;

    ERR(PF_COPY(input_worldP, temp1_worldP, NULL, &dest_rect));

    // Now process at OUTPUT size
    // Your effect processing here - operates on OUTPUT-sized buffers
    // Extension/blur will fill the expanded areas
    ERR(ProcessEffect(in_data, out_data, temp1_worldP, temp2_worldP));

    // Copy result to output
    ERR(PF_COPY(temp2_worldP, output_worldP, NULL, NULL));

    // Cleanup
    if (temp1_worldP) wsP->PF_DisposeWorld(in_data->effect_ref, temp1_worldP);
    if (temp2_worldP) wsP->PF_DisposeWorld(in_data->effect_ref, temp2_worldP);

    return err;
}
```

### Step 3: Handle "Keep Original Alpha" Mode

If you have a mode that constrains output to original alpha:

```cpp
// Allocate buffer to store original input for alpha constraint
PF_EffectWorld *original_worldP = nullptr;
ERR(wsP->PF_NewWorld(in_data->effect_ref,
                     output_worldP->width,
                     output_worldP->height,
                     PF_NewWorldFlag_CLEAR_PIXELS,
                     original_worldP));

// Clear it first
AEFX_SuiteScoper<PF_FillMatteSuite2> fillMatteSuite(
    in_data, kPFFillMatteSuite, kPFFillMatteSuiteVersion2, out_data);
PF_Pixel clear_pixel = {0, 0, 0, 0};
fillMatteSuite->fill(in_data->effect_ref, &clear_pixel, NULL, original_worldP);

// Copy input to center using the SAME offset calculation
PF_Rect dest_rect;
dest_rect.left   = actual_offset_x;
dest_rect.top    = actual_offset_y;
dest_rect.right  = dest_rect.left + input_worldP->width;
dest_rect.bottom = dest_rect.top + input_worldP->height;

ERR(PF_COPY(input_worldP, original_worldP, NULL, &dest_rect));

// Later, when applying alpha constraint, original_worldP and
// your processed result are both OUTPUT-sized, so coordinates match!
```

---

## Common Pitfalls

### Pitfall 1: Processing at Input Size
```cpp
// WRONG - temp buffers at input size
PF_NewWorld(input_worldP->width, input_worldP->height, temp);
ProcessEffect(temp);  // Extension gets clipped!
PF_COPY(temp, output_worldP);  // Just copying clipped result to larger buffer
```

**Fix**: Allocate at output size, copy input to center, process at output size.

### Pitfall 2: Trusting output_origin_x/y
```cpp
// WRONG - assuming output_origin is always set correctly
PF_Rect dest_rect;
dest_rect.left = in_data->output_origin_x;  // Might be 0!
dest_rect.top  = in_data->output_origin_y;  // Might be 0!
```

**Fix**: Calculate offset manually and use output_origin as fallback:
```cpp
A_long offset_x = (output_worldP->width - input_worldP->width) / 2;
A_long actual_offset_x = (in_data->output_origin_x != 0)
                         ? in_data->output_origin_x
                         : offset_x;
```

### Pitfall 3: Inconsistent Offsets
```cpp
// WRONG - different offset calculations for different buffers
PF_COPY(input, temp1, NULL, &rect1);  // Uses one offset
PF_COPY(input, temp2, NULL, &rect2);  // Uses different offset!
```

**Fix**: Calculate offset ONCE, use everywhere:
```cpp
A_long offset_x = (output_worldP->width - input_worldP->width) / 2;
A_long offset_y = (output_worldP->height - input_worldP->height) / 2;
A_long actual_offset_x = (in_data->output_origin_x != 0)
                         ? in_data->output_origin_x : offset_x;
A_long actual_offset_y = (in_data->output_origin_y != 0)
                         ? in_data->output_origin_y : offset_y;

// Use actual_offset_x/y for ALL buffer positioning
```

### Pitfall 4: Forgetting PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS
```cpp
// WRONG - expanding rects but not setting flag
extra->output->result_rect.left -= expansion;
// ... expand other sides ...
// Missing: extra->output->flags = PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS;
```

**Fix**: Always set the flag in PreRender when expanding output.

### Pitfall 5: Wrong Pixel Format in PF_NewWorld
```cpp
// WRONG - using hardcoded pixel format
PF_NewWorld(width, height, PF_PixelFormat_ARGB32, temp);
```

**Fix**: Match input pixel format or use deep color flags:
```cpp
PF_PixelFormat format = PF_PixelFormat_INVALID;
ERR(wsP->PF_GetPixelFormat(input_worldP, &format));
ERR(wsP->PF_NewWorld(in_data->effect_ref, width, height,
                     (format == PF_PixelFormat_ARGB128),
                     format,
                     temp_worldP));
```

---

## Real-World Example: Edge Extension

Complete example showing all pieces together:

```cpp
// PreRender.cpp
static PF_Err PreRender(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_PreRenderExtra *extra)
{
    PF_Err err = PF_Err_NONE;
    PF_RenderRequest req = extra->input->output_request;
    PF_CheckoutResult in_result;

    ERR(extra->cb->checkout_layer(in_data->effect_ref, LAYER_INPUT, LAYER_INPUT,
                                   &req, in_data->current_time, in_data->time_step,
                                   in_data->time_scale, &in_result));

    // Get parameters
    PF_ParamDef erosion_param, blur_param, iterations_param;
    AEFX_CLR_STRUCT(erosion_param);
    AEFX_CLR_STRUCT(blur_param);
    AEFX_CLR_STRUCT(iterations_param);

    ERR(PF_CHECKOUT_PARAM(in_data, PARAM_EROSION, in_data->current_time,
                          in_data->time_step, in_data->time_scale, &erosion_param));
    ERR(PF_CHECKOUT_PARAM(in_data, PARAM_BLUR, in_data->current_time,
                          in_data->time_step, in_data->time_scale, &blur_param));
    ERR(PF_CHECKOUT_PARAM(in_data, PARAM_ITERATIONS, in_data->current_time,
                          in_data->time_step, in_data->time_scale, &iterations_param));

    float erosion = (float)erosion_param.u.fs_d.value;
    float blur_per_iteration = (float)blur_param.u.fs_d.value;
    int iterations = iterations_param.u.sd.value;

    // Calculate maximum possible extension
    float max_blur = blur_per_iteration * iterations;
    float max_extension = erosion + max_blur;
    A_long expansion = (A_long)(max_extension + 10.0f);

    // Expand output rectangles
    extra->output->result_rect = in_result.result_rect;
    extra->output->result_rect.left   -= expansion;
    extra->output->result_rect.right  += expansion;
    extra->output->result_rect.top    -= expansion;
    extra->output->result_rect.bottom += expansion;

    extra->output->max_result_rect = in_result.max_result_rect;
    extra->output->max_result_rect.left   -= expansion;
    extra->output->max_result_rect.right  += expansion;
    extra->output->max_result_rect.top    -= expansion;
    extra->output->max_result_rect.bottom += expansion;

    extra->output->flags = PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS;

    return err;
}

// SmartRender.cpp
static PF_Err SmartRender(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_SmartRenderExtra *extra)
{
    PF_Err err = PF_Err_NONE;

    // Checkout input/output
    PF_EffectWorld *input_worldP, *output_worldP;
    ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, LAYER_INPUT, &input_worldP));
    ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));

    // Calculate offset
    A_long offset_x = (output_worldP->width - input_worldP->width) / 2;
    A_long offset_y = (output_worldP->height - input_worldP->height) / 2;
    A_long actual_offset_x = (in_data->output_origin_x != 0) ? in_data->output_origin_x : offset_x;
    A_long actual_offset_y = (in_data->output_origin_y != 0) ? in_data->output_origin_y : offset_y;

    // Get pixel format
    AEFX_SuiteScoper<PF_WorldSuite2> wsP(in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);
    PF_PixelFormat format = PF_PixelFormat_INVALID;
    ERR(wsP->PF_GetPixelFormat(input_worldP, &format));
    PF_Boolean deep = (format == PF_PixelFormat_ARGB128);

    // Allocate temp buffers at OUTPUT size
    PF_EffectWorld *temp1_worldP = nullptr, *temp2_worldP = nullptr, *original_worldP = nullptr;

    ERR(wsP->PF_NewWorld(in_data->effect_ref, output_worldP->width, output_worldP->height,
                         deep, format, temp1_worldP));
    ERR(wsP->PF_NewWorld(in_data->effect_ref, output_worldP->width, output_worldP->height,
                         deep, format, temp2_worldP));
    ERR(wsP->PF_NewWorld(in_data->effect_ref, output_worldP->width, output_worldP->height,
                         deep, format, original_worldP));

    // Clear temp buffers
    AEFX_SuiteScoper<PF_FillMatteSuite2> fillMatteSuite(
        in_data, kPFFillMatteSuite, kPFFillMatteSuiteVersion2, out_data);
    PF_Pixel clear_pixel = {0, 0, 0, 0};
    fillMatteSuite->fill(in_data->effect_ref, &clear_pixel, NULL, temp1_worldP);
    fillMatteSuite->fill(in_data->effect_ref, &clear_pixel, NULL, temp2_worldP);
    fillMatteSuite->fill(in_data->effect_ref, &clear_pixel, NULL, original_worldP);

    // Copy input to center of all buffers using same offset
    PF_Rect dest_rect;
    dest_rect.left   = actual_offset_x;
    dest_rect.top    = actual_offset_y;
    dest_rect.right  = dest_rect.left + input_worldP->width;
    dest_rect.bottom = dest_rect.top + input_worldP->height;

    ERR(PF_COPY(input_worldP, temp1_worldP, NULL, &dest_rect));
    ERR(PF_COPY(input_worldP, original_worldP, NULL, &dest_rect));

    // Process at OUTPUT size
    ERR(ApplyEdgeExtension(in_data, out_data, temp1_worldP, temp2_worldP));

    // If "Keep Original Alpha" mode, constrain to original alpha
    bool keep_alpha = GetKeepAlphaParam(in_data);
    if (keep_alpha) {
        ERR(ConstrainToOriginalAlpha(temp2_worldP, original_worldP, output_worldP));
    } else {
        ERR(PF_COPY(temp2_worldP, output_worldP, NULL, NULL));
    }

    // Cleanup
    if (temp1_worldP) wsP->PF_DisposeWorld(in_data->effect_ref, temp1_worldP);
    if (temp2_worldP) wsP->PF_DisposeWorld(in_data->effect_ref, temp2_worldP);
    if (original_worldP) wsP->PF_DisposeWorld(in_data->effect_ref, original_worldP);

    return err;
}
```

---

## Debugging Tips

### 1. Log Buffer Sizes and Offsets
```cpp
std::stringstream log;
log << "BUFFER EXPANSION DEBUG:"
    << "\n  Input:  " << input_worldP->width << "x" << input_worldP->height
    << "\n  Output: " << output_worldP->width << "x" << output_worldP->height
    << "\n  output_origin: (" << in_data->output_origin_x << "," << in_data->output_origin_y << ")"
    << "\n  Calculated offset: (" << offset_x << "," << offset_y << ")"
    << "\n  Using offset: (" << actual_offset_x << "," << actual_offset_y << ")";
YourDebugLog(log.str());
```

### 2. Visual Debugging
Fill expanded areas with a bright color to verify they're being used:
```cpp
// After copying input to center, fill border with debug color
PF_Pixel debug_color = {255, 0, 255, 128};  // Bright magenta
// Fill top border
PF_Rect top_border = {0, 0, output_worldP->width, actual_offset_y};
fillMatteSuite->fill(in_data->effect_ref, &debug_color, &top_border, temp1_worldP);
// Repeat for other borders...
```

If you see magenta borders in output, expansion is working!

### 3. Check for Coordinate Mismatches
```cpp
// If Keep Original Alpha shows offset, check if you're using
// the same offset for both original_worldP and processed result
assert(original_worldP->width == temp2_worldP->width);
assert(original_worldP->height == temp2_worldP->height);
// Both must be OUTPUT-sized for coordinates to match!
```

### 4. Verify PF_COPY Destination Rect
```cpp
PF_Rect dest_rect;
dest_rect.left   = actual_offset_x;
dest_rect.top    = actual_offset_y;
dest_rect.right  = dest_rect.left + input_worldP->width;
dest_rect.bottom = dest_rect.top + input_worldP->height;

// Sanity check
assert(dest_rect.right <= output_worldP->width);
assert(dest_rect.bottom <= output_worldP->height);
assert(dest_rect.left >= 0);
assert(dest_rect.top >= 0);

ERR(PF_COPY(input_worldP, temp1_worldP, NULL, &dest_rect));
```

### 5. Test Cases
1. **No mask** - Should extend symmetrically on all sides
2. **Simple mask** - Should extend beyond mask boundary
3. **Animated mask** - Verify expansion updates each frame
4. **Nested compositions** - Test in comp within comp
5. **4K+ resolution** - Ensure no performance issues with large buffers

---

## Summary Checklist

When implementing buffer expansion:

- [ ] **PreRender**: Expand `result_rect` and `max_result_rect` on all sides
- [ ] **PreRender**: Set `PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS` flag
- [ ] **SmartRender**: Calculate offset: `(output_size - input_size) / 2`
- [ ] **SmartRender**: Use `output_origin` if non-zero, else calculated offset
- [ ] **SmartRender**: Allocate ALL temp buffers at OUTPUT size
- [ ] **SmartRender**: Clear temp buffers before use
- [ ] **SmartRender**: Copy input to CENTER using calculated offset
- [ ] **SmartRender**: Use SAME offset for all buffer positioning
- [ ] **Processing**: Operate on OUTPUT-sized buffers throughout
- [ ] **Testing**: Verify with masked layers, no cropping occurs
- [ ] **Debugging**: Log buffer sizes and offsets during development

---

## Additional Resources

- **After Effects SDK**: `PF_SmartRenderExtra` documentation
- **SDK Examples**: `SmartyPants` (adds colored borders), `SDK_Noise` (basic SmartFX)
- **Forum Discussions**: Adobe Community forums on buffer expansion
---

*Document Version: 1.1*
*Last Updated: 2025-12-06*
*Based on: Real-world experience implementing buffer expansion for edge extension effects*
