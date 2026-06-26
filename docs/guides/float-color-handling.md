# Float Color Workflow Documentation

## Overview

This document describes how to maintain correct color handling across 8-bit, 16-bit, and 32-bit float color modes by using a **pure float workflow** internally. All color processing happens in normalized floating-point values (0.0 to 1.0 range), regardless of the input or output bit depth. This ensures consistent behavior and prevents precision loss or conversion artifacts.

## Bit Depth Detection

### Detection Method

```cpp
// Detect bit depth for any PF_LayerDef
bool is_32bit = false;
int bytes_expected_32 = layer->width * sizeof(PF_PixelFloat);
if (layer->rowbytes >= bytes_expected_32 && layer->rowbytes < bytes_expected_32 * 2) {
    is_32bit = true;
}

bool is_16bit = PF_WORLD_IS_DEEP(layer);
// If neither 32-bit nor 16-bit, it's 8-bit
```

> **Note:** The recommended approach for pixel format detection is `PF_WorldSuite2::PF_GetPixelFormat()`, which returns the format directly without heuristics. The rowbytes method above works but can fail if rowbytes includes unexpected padding.

### Why the Heuristic Works

1. **32-bit float detection**: Check if rowbytes matches expected size for float pixels (16 bytes per pixel: 4 floats x 4 bytes)
2. **16-bit detection**: Use the SDK's `PF_WORLD_IS_DEEP()` macro which reliably identifies 16-bit integer mode
3. **8-bit fallback**: If neither condition is met, the layer is 8-bit (4 bytes per pixel)

## The Pure Float Pipeline

### Core Principle

All color values are immediately converted to normalized float (0.0-1.0) when read, processed as float throughout the pipeline, and only converted back to the target format when writing output.

```
Input (8/16/32-bit) -> Normalize to Float -> Process as Float -> Denormalize to Output (8/16/32-bit)
```

### Benefits

1. **No precision loss**: Float has enough precision for all formats
2. **Consistent math**: All operations use the same value range
3. **No cascade conversions**: Direct conversion from source to float, then float to destination
4. **Simplified code**: One processing path instead of three

## Input Normalization

When reading pixels from a texture layer, we normalize based on the detected format:

### 8-bit Input
```cpp
PF_Pixel8* pixel = /* get pixel */;
float r = pixel->red / 255.0f;     // Convert 0-255 to 0.0-1.0
float g = pixel->green / 255.0f;
float b = pixel->blue / 255.0f;
float a = pixel->alpha / 255.0f;
```

### 16-bit Input
```cpp
PF_Pixel16* pixel = /* get pixel */;
float r = pixel->red / (float)PF_MAX_CHAN16;    // Convert 0-32768 to 0.0-1.0
float g = pixel->green / (float)PF_MAX_CHAN16;
float b = pixel->blue / (float)PF_MAX_CHAN16;
float a = pixel->alpha / (float)PF_MAX_CHAN16;
```

### 32-bit Float Input
```cpp
PF_PixelFloat* pixel = /* get pixel */;
float r = pixel->red;     // Already in 0.0-1.0 range
float g = pixel->green;
float b = pixel->blue;
float a = pixel->alpha;
```

## Processing in Float

All color operations happen in normalized float space:

```cpp
// Bilinear interpolation example
float r = (1-fx)*(1-fy)*p00_r + fx*(1-fy)*p10_r +
          (1-fx)*fy*p01_r + fx*fy*p11_r;

// Accumulation for supersampling
float accum_r = 0.0f;
for (each sample) {
    accum_r += sample_r;  // All in 0.0-1.0 range
}
float final_r = accum_r / num_samples;
```

## Output Denormalization

When writing final pixels, we convert from float to the target format.

> **Important:** Always clamp float values to [0.0, 1.0] before converting to integer pixel values to avoid overflow.

### 8-bit Output
```cpp
PF_Pixel8* dst = /* get destination pixel */;
float clamped_r = (float_r < 0.0f) ? 0.0f : (float_r > 1.0f) ? 1.0f : float_r;
float clamped_g = (float_g < 0.0f) ? 0.0f : (float_g > 1.0f) ? 1.0f : float_g;
float clamped_b = (float_b < 0.0f) ? 0.0f : (float_b > 1.0f) ? 1.0f : float_b;
float clamped_a = (float_a < 0.0f) ? 0.0f : (float_a > 1.0f) ? 1.0f : float_a;
dst->red = (unsigned char)(clamped_r * 255.0f + 0.5f);
dst->green = (unsigned char)(clamped_g * 255.0f + 0.5f);
dst->blue = (unsigned char)(clamped_b * 255.0f + 0.5f);
dst->alpha = (unsigned char)(clamped_a * 255.0f + 0.5f);
```

### 16-bit Output
```cpp
PF_Pixel16* dst = /* get destination pixel */;
float clamped_r = (float_r < 0.0f) ? 0.0f : (float_r > 1.0f) ? 1.0f : float_r;
float clamped_g = (float_g < 0.0f) ? 0.0f : (float_g > 1.0f) ? 1.0f : float_g;
float clamped_b = (float_b < 0.0f) ? 0.0f : (float_b > 1.0f) ? 1.0f : float_b;
float clamped_a = (float_a < 0.0f) ? 0.0f : (float_a > 1.0f) ? 1.0f : float_a;
dst->red = (A_u_short)(clamped_r * PF_MAX_CHAN16 + 0.5f);
dst->green = (A_u_short)(clamped_g * PF_MAX_CHAN16 + 0.5f);
dst->blue = (A_u_short)(clamped_b * PF_MAX_CHAN16 + 0.5f);
dst->alpha = (A_u_short)(clamped_a * PF_MAX_CHAN16 + 0.5f);
```

### 32-bit Float Output
```cpp
PF_PixelFloat* dst = /* get destination pixel */;
dst->red = float_r;     // Direct assignment, already in correct range
dst->green = float_g;   // Note: 32-bit float can represent HDR values > 1.0
dst->blue = float_b;
dst->alpha = float_a;
```

## Complete Example: Vanilla Renderer

Here's how a vanilla renderer implements the float workflow for texture sampling:

```cpp
// Detect texture format
bool tex_is_32bit = false;
int tex_bytes_expected_32 = texture_layer->width * sizeof(PF_PixelFloat);
if (texture_layer->rowbytes >= tex_bytes_expected_32 &&
    texture_layer->rowbytes < tex_bytes_expected_32 * 2) {
    tex_is_32bit = true;
}
bool tex_is_16bit = PF_WORLD_IS_DEEP(texture_layer);

float r, g, b, a;

if (tex_is_32bit) {
    // Sample 32-bit float texture
    PF_PixelFloat* p = /* get pixel */;
    r = p->red;  // Already normalized
} else if (tex_is_16bit) {
    // Sample 16-bit texture and normalize
    PF_Pixel16* p = /* get pixel */;
    r = p->red / (float)PF_MAX_CHAN16;
} else {
    // Sample 8-bit texture and normalize
    PF_Pixel8* p = /* get pixel */;
    r = p->red / 255.0f;
}

// Process in float...
r = apply_some_effect(r);

// Write to output (clamp for integer formats)
if (output_is_32bit) {
    dst->red = r;  // Direct
} else if (output_is_16bit) {
    float cr = (r < 0.0f) ? 0.0f : (r > 1.0f) ? 1.0f : r;
    dst->red = (A_u_short)(cr * PF_MAX_CHAN16 + 0.5f);
} else {
    float cr = (r < 0.0f) ? 0.0f : (r > 1.0f) ? 1.0f : r;
    dst->red = (unsigned char)(cr * 255.0f + 0.5f);
}
```

## Buffer Initialization and Alpha Handling

### Clearing Output Buffers

When initializing output buffers, it's critical to use the **same format detection method** for both clearing and writing operations. Mismatched detection can lead to alpha channel corruption.

#### Consistent Format Detection
```cpp
// ALWAYS use this same detection for both clearing AND writing
bool is_32bit = false;
int bytes_expected_32 = output->width * sizeof(PF_PixelFloat);
if (output->rowbytes >= bytes_expected_32 && output->rowbytes < bytes_expected_32 * 2) {
    is_32bit = true;
}
bool is_16bit = PF_WORLD_IS_DEEP(output);

// Clear buffer based on detected format
if (is_32bit) {
    for (int y = 0; y < output->height; y++) {
        for (int x = 0; x < output->width; x++) {
            PF_PixelFloat* pixel = ((PF_PixelFloat*)((char*)output->data + y * output->rowbytes)) + x;
            pixel->red = pixel->green = pixel->blue = pixel->alpha = 0.0f;
        }
    }
} else if (is_16bit) {
    for (int y = 0; y < output->height; y++) {
        for (int x = 0; x < output->width; x++) {
            PF_Pixel16* pixel = ((PF_Pixel16*)((char*)output->data + y * output->rowbytes)) + x;
            pixel->red = pixel->green = pixel->blue = pixel->alpha = 0;
        }
    }
} else {
    for (int y = 0; y < output->height; y++) {
        for (int x = 0; x < output->width; x++) {
            PF_Pixel8* pixel = ((PF_Pixel8*)((char*)output->data + y * output->rowbytes)) + x;
            pixel->red = pixel->green = pixel->blue = pixel->alpha = 0;
        }
    }
}
```

### Why Format Detection Consistency Matters

When the format detection method differs between clearing and writing operations, pixels that should remain transparent can show garbage data or background textures. This is especially problematic in 16-bit and 32-bit modes where uninitialized memory may contain non-zero values.

## Common Pitfalls Avoided

### 1. Accumulating in Wrong Format
**Wrong**: Accumulating in 8-bit range for 16/32-bit output
```cpp
float accum = 0;
accum += pixel_value * 255;  // Wrong for 16/32-bit
// Later trying to scale back up loses precision
```

**Correct**: Always accumulate in normalized float
```cpp
float accum = 0;
accum += normalized_value;  // Always 0.0-1.0
```

### 2. Understanding PF_PixelFormat Enum Values

The `PF_PixelFormat` enum values in the SDK are FourCC codes, which ARE the actual enum values used internally:
```cpp
// These hex values are FourCC codes and are the correct enum values in the SDK:
// PF_PixelFormat_ARGB32  = 'argb' = 0x61726762
// PF_PixelFormat_ARGB64  = 'ae16' = 0x61653136
// PF_PixelFormat_ARGB128 = 'ae32' = 0x61653332
```

The recommended approach for pixel format detection is `PF_WorldSuite2::PF_GetPixelFormat()`:
```cpp
PF_PixelFormat format = PF_PixelFormat_INVALID;
AEFX_SuiteScoper<PF_WorldSuite2> worldSuite(
    in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);
worldSuite->PF_GetPixelFormat(worldP, &format);

switch (format) {
    case PF_PixelFormat_ARGB32:  // 8-bit
        break;
    case PF_PixelFormat_ARGB64:  // 16-bit
        break;
    case PF_PixelFormat_ARGB128: // 32-bit float
        break;
}
```

### 3. Inconsistent Format Detection
**Wrong**: Using different detection methods for clearing vs writing
```cpp
// Clearing with one method
if (some_format_check) {
    // Clear as 32-bit
}

// Later, writing with different method
bool is_32bit = (output->rowbytes >= expected_32);
if (is_32bit) {
    // Write as 32-bit
}
// MISMATCH! May not clear/write the same pixels
```

**Correct**: Use the same detection everywhere
```cpp
// Define once
bool is_32bit = (output->rowbytes >= bytes_expected_32 &&
                 output->rowbytes < bytes_expected_32 * 2);
bool is_16bit = PF_WORLD_IS_DEEP(output);

// Use consistently for both clearing and writing
if (is_32bit) { /* handle 32-bit */ }
else if (is_16bit) { /* handle 16-bit */ }
else { /* handle 8-bit */ }
```

### 4. Cascading Conversions
**Wrong**: Converting through intermediate formats
```cpp
// 16-bit -> 8-bit -> float -> 16-bit (loses precision!)
int value_8bit = pixel16->red * 255 / PF_MAX_CHAN16;
float normalized = value_8bit / 255.0f;
```

**Correct**: Direct conversion
```cpp
// 16-bit -> float -> 16-bit (preserves precision)
float normalized = pixel16->red / (float)PF_MAX_CHAN16;
```

### 5. Mixing Value Ranges
**Wrong**: Mixing different value ranges in calculations
```cpp
if (is_16bit) {
    accum += sample * PF_MAX_CHAN16;  // Now accum is in 16-bit range
} else {
    accum += sample * 255;  // Now accum is in 8-bit range
}
// What range is accum in? Depends on format!
```

**Correct**: Keep everything normalized
```cpp
// Always normalize on input
accum += normalized_sample;  // Always 0.0-1.0
// Convert only on output
```

### 6. world_flags
The key change was ensuring that when SmartRender calls regular
  Render in 32-bit mode, the world_flags and pix_aspect_ratio fields are properly copied from the PF_EffectWorld
  to the PF_LayerDef structures.

  The world_flags field is critical because it contains the bit depth information that PF_COPY needs to properly
  copy pixels between layers. Without this field being set correctly, PF_COPY might treat the 32-bit float data
  incorrectly, causing the half-black screen issue.


## Performance Considerations

The float workflow has minimal performance impact:

1. **Modern CPUs**: Float operations are as fast as integer operations on modern processors
2. **Single conversion**: Each pixel is converted once on read and once on write
3. **SIMD friendly**: Float operations can be vectorized effectively
4. **Cache efficiency**: Consistent data format improves cache usage

## Testing Checklist

To verify correct bit depth handling:

- [ ] 8-bit comp with 8-bit texture -> correct rendering
- [ ] 16-bit comp with 8-bit texture -> texture upsampled correctly
- [ ] 32-bit comp with 8-bit texture -> texture upsampled correctly
- [ ] 8-bit comp with 16-bit texture -> texture downsampled correctly
- [ ] 16-bit comp with 16-bit texture -> correct rendering
- [ ] 32-bit comp with 16-bit texture -> texture upsampled correctly
- [ ] 8-bit comp with 32-bit texture -> texture downsampled correctly
- [ ] 16-bit comp with 32-bit texture -> texture downsampled correctly
- [ ] 32-bit comp with 32-bit texture -> correct rendering

## Summary

The pure float workflow ensures correct color handling by:

1. **Detecting bit depth correctly** using SDK methods, not assumptions
2. **Normalizing all input** to 0.0-1.0 range immediately
3. **Processing everything as float** with no intermediate conversions
4. **Denormalizing only on output** to the target format
5. **Never mixing value ranges** in calculations

This approach guarantees consistent, accurate color reproduction across all bit depths supported by After Effects.
