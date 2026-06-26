# Using SDK Transfer Modes with PF_MaskWorld

This document explains how to use After Effects SDK's `transfer_rect` function with `PF_MaskWorld` for masked compositing operations.

## Overview

The `transfer_rect` callback is a powerful SDK function that composites one `PF_EffectWorld` onto another using various blending modes, with optional masking support via `PF_MaskWorld`.

## Key Data Structures

### PF_CompositeMode

Defines how pixels are composited:

```cpp
typedef struct {
    PF_TransferMode     xfer;       // Blending mode (see PF_Xfer_* constants)
    A_long              rand_seed;  // For PF_Xfer_DISSOLVE_RANDOMIZED
    A_u_char            opacity;    // 0-255 for 8-bit
    PF_Boolean          rgb_only;   // Ignored for PF_Xfer_MULTIPLY_ALPHA modes
    A_u_short           opacitySu;  // For deep color (16-bit and 32-bit)
} PF_CompositeMode;
```

### PF_MaskWorld

Provides an alpha mask for the transfer operation:

```cpp
typedef struct {
    PF_EffectWorld      mask;           // World whose alpha is used as mask
    PF_Point            offset;         // Offset of mask relative to dest
    PF_MaskFlags        what_is_mask;   // How to interpret the mask
} PF_MaskWorld;
```

### PF_MaskFlags

```cpp
enum {
    PF_MaskFlag_NONE = 0,           // Use alpha channel directly
    PF_MaskFlag_INVERTED = 1L << 0, // Invert the mask result
    PF_MaskFlag_LUMINANCE = 1L << 1 // Use luminance instead of alpha
};
```

## Transfer Modes (PF_Xfer_*)

Common transfer modes from `AE_EffectCB.h`:

| Mode | Description |
|------|-------------|
| `PF_Xfer_COPY` | Simple copy (replaces destination) |
| `PF_Xfer_BEHIND` | Composite behind destination (useful for layer stacking) |
| `PF_Xfer_IN_FRONT` | Composite in front of destination (standard OVER) |
| `PF_Xfer_DISSOLVE` | Random dissolve based on opacity |
| `PF_Xfer_ADD` | Additive blend |
| `PF_Xfer_MULTIPLY` | Multiply blend |
| `PF_Xfer_SCREEN` | Screen blend |
| `PF_Xfer_OVERLAY` | Overlay blend |
| `PF_Xfer_DIFFERENCE` | Difference blend |
| `PF_Xfer_MULTIPLY_ALPHA` | dest alpha *= src alpha |
| `PF_Xfer_MULTIPLY_NOT_ALPHA` | dest alpha *= ~(src alpha) |

## Function Signature

```cpp
PF_Err transfer_rect(
    PF_ProgPtr              effect_ref,     // from in_data->effect_ref
    PF_Quality              quality,        // PF_Quality_LO or PF_Quality_HI
    PF_ModeFlags            m_flags,        // PF_MF_Alpha_PREMUL or PF_MF_Alpha_STRAIGHT
    PF_Field                field,          // PF_Field_FRAME, _UPPER, or _LOWER
    const PF_Rect           *src_rec,       // Source rectangle (NULL for whole world)
    const PF_EffectWorld    *src_world,     // Source pixels
    const PF_CompositeMode  *comp_mode,     // How to composite
    const PF_MaskWorld      *mask_world0,   // Optional mask (NULL for no mask)
    A_long                  dest_x,         // Destination X offset
    A_long                  dest_y,         // Destination Y offset
    PF_EffectWorld          *dst_world      // Destination buffer
);
```

## Real-World Usage Examples

These examples demonstrate `transfer_rect` usage for edge extension compositing.

### Pattern 1: Layer Stacking with BEHIND Mode

Used to build up blur layers where more blurred content goes under less blurred:

```cpp
// Subsequent iterations (more blurred) - composite UNDER the accumulator
PF_CompositeMode comp_mode;
AEFX_CLR_STRUCT(comp_mode);
comp_mode.xfer = PF_Xfer_BEHIND;  // More blurred goes UNDER
comp_mode.rand_seed = 0;
comp_mode.opacity = 255;
comp_mode.rgb_only = FALSE;
comp_mode.opacitySu = PF_MAX_CHAN16;

ERR(wtransformSuite->transfer_rect(
    in_data->effect_ref,
    PF_Quality_HI,
    PF_MF_Alpha_PREMUL,
    PF_Field_FRAME,
    &temp1_worldP->extent_hint,
    temp1_worldP,       // Source: current blur iteration
    &comp_mode,
    NULL,               // No mask
    0, 0,
    temp3_worldP));     // Dest: accumulator
```

### Pattern 2: Compositing Original ON TOP (IN_FRONT Mode)

Used to put the sharp original image over the blurred extension:

```cpp
// Composite original ON TOP of blur_stack
PF_CompositeMode comp_mode_overlay;
AEFX_CLR_STRUCT(comp_mode_overlay);
comp_mode_overlay.xfer = PF_Xfer_IN_FRONT;  // Original ON TOP
comp_mode_overlay.rand_seed = 0;
comp_mode_overlay.opacity = 255;
comp_mode_overlay.rgb_only = FALSE;
comp_mode_overlay.opacitySu = PF_MAX_CHAN16;

ERR(wtransformSuite->transfer_rect(
    in_data->effect_ref,
    PF_Quality_HI,
    PF_MF_Alpha_PREMUL,
    PF_Field_FRAME,
    &original_worldP->extent_hint,
    original_worldP,             // Source: original
    &comp_mode_overlay,
    NULL,                        // No mask
    0, 0,
    temp2_worldP));              // Dest: blur_stack (becomes result)
```

### Pattern 3: Masked Transfer with PF_MaskWorld

The most complex pattern - using a separate buffer's alpha as a mask:

```cpp
// First, prepare the mask world with blurred alpha values
int temp3_stride = temp3_worldP->rowbytes / sizeof(PF_PixelFloat);
PF_PixelFloat* mask_data = (PF_PixelFloat*)temp3_worldP->data;

for (int y = 0; y < height; y++) {
    PF_PixelFloat* mask_row = mask_data + y * temp3_stride;
    for (int x = 0; x < width; x++) {
        float blur_a = blurred_alpha[y * width + x];
        // Set alpha to blurred value (RGB can be set for debugging)
        mask_row[x].alpha = blur_a;
        mask_row[x].red = blur_a;
        mask_row[x].green = blur_a;
        mask_row[x].blue = blur_a;
    }
}

// Clear output first
PF_PixelFloat clear_color = {0.0f, 0.0f, 0.0f, 0.0f};
ERR(fillMatteSuite->fill_float(in_data->effect_ref, &clear_color, NULL, output_worldP));

// Set up the mask world structure
PF_MaskWorld mask_world;
AEFX_CLR_STRUCT(mask_world);
mask_world.mask = *temp3_worldP;        // Use temp3 as the mask
mask_world.offset.h = 0;                // No horizontal offset
mask_world.offset.v = 0;                // No vertical offset
mask_world.what_is_mask = PF_MaskFlag_NONE;  // Use alpha channel as mask

// Transfer with mask
PF_CompositeMode comp_mode_masked;
AEFX_CLR_STRUCT(comp_mode_masked);
comp_mode_masked.xfer = PF_Xfer_COPY;   // Just copy, mask does the work
comp_mode_masked.rand_seed = 0;
comp_mode_masked.opacity = 255;
comp_mode_masked.rgb_only = FALSE;
// Note: opacitySu is A_u_short (unsigned 16-bit), not float.
// Use PF_MAX_CHAN16 (32768) for full opacity.
comp_mode_masked.opacitySu = PF_MAX_CHAN16;

ERR(wtransformSuite->transfer_rect(
    in_data->effect_ref,
    PF_Quality_HI,
    PF_MF_Alpha_PREMUL,
    PF_Field_FRAME,
    &temp2_worldP->extent_hint,
    temp2_worldP,           // Source: composited result
    &comp_mode_masked,
    &mask_world,            // Mask: blurred alpha controls visibility
    0, 0,
    output_worldP));        // Dest: final output
```

## Accessing the Suite

Use `AEFX_SuiteScoper` to safely acquire the WorldTransformSuite:

```cpp
AEFX_SuiteScoper<PF_WorldTransformSuite1> wtransformSuite(
    in_data,
    kPFWorldTransformSuite,
    kPFWorldTransformSuiteVersion1,
    out_data);

if (!wtransformSuite.get()) {
    return PF_Err_INTERNAL_STRUCT_DAMAGED;
}

// Now use wtransformSuite->transfer_rect(...)
```

## Important Considerations

### 1. Premultiplied vs Straight Alpha

Always pass the correct `PF_ModeFlags`:
- `PF_MF_Alpha_PREMUL` - Source is premultiplied (RGB scaled by alpha)
- `PF_MF_Alpha_STRAIGHT` - Source is straight alpha (RGB unscaled)

After Effects typically works in premultiplied space internally.

### 2. Quality Setting

- `PF_Quality_HI` - Uses high-quality compositing (anti-aliased)
- `PF_Quality_LO` - Uses faster, lower-quality compositing

### 3. Field Rendering

- `PF_Field_FRAME` - Process all scanlines
- `PF_Field_UPPER` - Process only upper field (odd scanlines)
- `PF_Field_LOWER` - Process only lower field (even scanlines)

### 4. Opacity for Different Bit Depths

```cpp
// 8-bit
comp_mode.opacity = 255;  // A_u_char max

// 16-bit
comp_mode.opacitySu = PF_MAX_CHAN16;  // 32768

// 32-bit float: opacitySu is still A_u_short, not float.
// Use PF_MAX_CHAN16 for full opacity.
comp_mode.opacitySu = PF_MAX_CHAN16;
```

### 5. Mask World Offset

The `offset` field allows positioning the mask relative to the destination:

```cpp
mask_world.offset.h = 10;  // Mask shifted 10 pixels right
mask_world.offset.v = 5;   // Mask shifted 5 pixels down
```

### 6. Inverted Masks

To use an inverted mask:

```cpp
mask_world.what_is_mask = PF_MaskFlag_INVERTED;
// Now white areas in mask become transparent, black becomes opaque
```

### 7. Luminance-Based Masks

To use luminance instead of alpha for masking:

> **Note:** AE uses weighted luminance (Rec.709), not simple (R+G+B)/3.

```cpp
mask_world.what_is_mask = PF_MaskFlag_LUMINANCE;
// Uses luminance of the mask pixels instead of the alpha channel
```

## Common Pitfalls

1. **Forgetting to clear destination** - For `PF_Xfer_COPY` with a mask, the destination should be cleared first or unexpected pixels may appear in unmasked areas.

2. **Mismatched buffer sizes** - The mask world should typically be the same size as the source and destination, or you need to use offsets correctly.

3. **Wrong alpha state** - Passing `PF_MF_Alpha_STRAIGHT` when data is actually premultiplied will cause color fringing.

4. **Not initializing comp_mode** - Always use `AEFX_CLR_STRUCT(comp_mode)` before setting fields.

5. **Deep color opacity** - For 16-bit and 32-bit, you must set `opacitySu` in addition to `opacity`. Remember that `opacitySu` is `A_u_short`, not float.

## SDK Example: Transformer.cpp

The SDK's Transformer sample shows basic `transfer_rect` usage:

```cpp
// From AfterEffectsSDK/Examples/Effect/Transformer/Transformer.cpp
PF_CompositeMode mode;
AEFX_CLR_STRUCT(mode);

mode.xfer       = PF_Xfer_DIFFERENCE;
mode.opacity    = static_cast<A_u_char>(FIX2INT_ROUND(params[XFORM_LAYERBLEND]->u.fd.value / PF_MAX_CHAN8));
mode.rgb_only   = TRUE;

PF_FpLong opacityF = FIX_2_FLOAT(params[XFORM_LAYERBLEND]->u.fd.value);
mode.opacity    = static_cast<A_u_char>((opacityF * PF_MAX_CHAN8) + 0.5);
mode.opacitySu  = static_cast<A_u_short>((opacityF * PF_MAX_CHAN16) + 0.5);

if (mode.opacitySu > PF_MAX_CHAN16) {
    mode.opacitySu = PF_MAX_CHAN16;
}

ERR(suites.WorldTransformSuite1()->transfer_rect(
    in_data->effect_ref,
    in_data->quality,
    PF_MF_Alpha_STRAIGHT,
    in_data->field,
    &color_world.extent_hint,
    &color_world,
    &mode,
    NULL,  // No mask
    (A_long)suites.ANSICallbacksSuite1()->fabs(output->extent_hint.right - color_world.extent_hint.right) / 2,
    (A_long)suites.ANSICallbacksSuite1()->fabs(output->extent_hint.bottom - color_world.extent_hint.bottom) / 2,
    output));
```

## See Also

- `AE_EffectCB.h` - Transfer mode definitions and `PF_TRANSFER_RECT` macro
- `AE_EffectCBSuites.h` - `PF_WorldTransformSuite1` definition
- `AE_Effect.h` - `PF_CompositeMode` structure definition
- `Transformer.cpp` - SDK example using transfer_rect
