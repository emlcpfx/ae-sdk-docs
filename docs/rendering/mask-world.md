# Mask World: PF_MaskWorld and PF_MaskFlags

## Overview

`PF_MaskWorld` is a structure defined in `AE_EffectCB.h` that provides a per-pixel mask for compositing operations within the After Effects SDK. It is used as an optional parameter to the `transfer_rect` and `transform_world` callback functions, enabling effects to perform masked compositing -- where certain areas of the destination are protected from or selectively affected by the source pixels being composited.

This is distinct from After Effects' user-facing mask system (the masks drawn on layers in the timeline). `PF_MaskWorld` is a programmatic tool for effect plugins to control compositing at the pixel level.

## PF_MaskWorld Structure

```c
typedef struct {
    PF_EffectWorld    mask;           // The mask pixel buffer
    PF_Point          offset;         // Offset of the mask relative to the destination
    PF_MaskFlags      what_is_mask;   // How to interpret the mask data
} PF_MaskWorld;
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `mask` | `PF_EffectWorld` | A standard effect world (pixel buffer) containing the mask data. Each pixel in this world contributes mask information according to `what_is_mask`. |
| `offset` | `PF_Point` | The position of the top-left corner of the mask world relative to the destination world. This allows the mask to cover only a sub-region of the destination. |
| `what_is_mask` | `PF_MaskFlags` | Flags that control how the mask pixel data is interpreted. |

## PF_MaskFlags

```c
enum {
    PF_MaskFlag_NONE     = 0,           // Use the alpha channel of the mask
    PF_MaskFlag_INVERTED = 1L << 0,     // Invert the mask result
    PF_MaskFlag_LUMINANCE = 1L << 1     // Use luminance values instead of alpha
};
typedef A_long PF_MaskFlags;
```

### Flag Details

| Flag | Value | Description |
|------|-------|-------------|
| `PF_MaskFlag_NONE` | `0` | The mask world's alpha channel is used as the mask. White (opaque) alpha means fully affected by the composite; black (transparent) alpha means fully protected. |
| `PF_MaskFlag_INVERTED` | `0x01` | Inverts the mask. Areas that would be affected become protected, and vice versa. This is applied after the alpha or luminance value is read. |
| `PF_MaskFlag_LUMINANCE` | `0x02` | Instead of reading the alpha channel, the luminance of the RGB values is used as the mask value. Bright pixels in the mask mean "affected," dark pixels mean "protected." |

Flags can be combined:

```c
// Use luminance values, inverted: dark areas are affected, bright areas are protected
PF_MaskFlags flags = PF_MaskFlag_LUMINANCE | PF_MaskFlag_INVERTED;
```

## How Masks Work with transfer_rect and transform_world

The `PF_MaskWorld` is an optional parameter to two compositing callbacks available through `in_data->utils`:

### transfer_rect

```c
PF_Err (*transfer_rect)(
    PF_ProgPtr              effect_ref,
    PF_Quality              quality,
    PF_ModeFlags            m_flags,
    PF_Field                field,
    const PF_Rect           *src_rec,       // Rectangle within source to transfer
    const PF_EffectWorld    *src_world,     // Source pixel buffer
    const PF_CompositeMode  *comp_mode,     // Transfer mode, opacity, etc.
    const PF_MaskWorld      *mask_world0,   // Optional mask (NULL for no mask)
    A_long                  dest_x,         // Destination X offset
    A_long                  dest_y,         // Destination Y offset
    PF_EffectWorld          *dst_world);    // Destination pixel buffer
```

`transfer_rect` composites a rectangular region from the source world onto the destination world using the specified composite mode. When `mask_world0` is non-NULL, the mask modulates the compositing operation on a per-pixel basis.

### transform_world

```c
PF_Err (*transform_world)(
    PF_ProgPtr              effect_ref,
    PF_Quality              quality,
    PF_ModeFlags            m_flags,
    PF_Field                field,
    const PF_EffectWorld    *src_world,     // Source pixel buffer
    const PF_CompositeMode  *comp_mode,     // Transfer mode, opacity, etc.
    const PF_MaskWorld      *mask_world0,   // Optional mask (NULL for no mask)
    const PF_FloatMatrix    *matrices,      // 3x3 transformation matrix array
    A_long                  num_matrices,   // Number of matrices (for motion blur)
    PF_Boolean              src2dst_matrix, // TRUE if matrix maps src->dst
    const PF_Rect           *dest_rect,     // Destination clipping rect
    PF_EffectWorld          *dst_world);    // Destination pixel buffer
```

`transform_world` is similar to `transfer_rect` but applies a geometric transformation (via a 3x3 matrix) to the source before compositing. The mask applies to the destination space -- it modulates how much of the transformed source actually lands on each destination pixel.

### Per-Pixel Mask Evaluation

During compositing, for each destination pixel at position `(x, y)`:

1. The corresponding mask pixel is looked up at `(x - mask.offset.h, y - mask.offset.v)` in the mask world.
2. If `PF_MaskFlag_LUMINANCE` is set, the luminance of the mask pixel's RGB channels is computed. Otherwise, the alpha channel is read.
3. If `PF_MaskFlag_INVERTED` is set, the mask value is inverted (`255 - value` in 8-bit).
4. The resulting mask value (0.0 to 1.0, scaled to the appropriate bit depth) is multiplied with the composite opacity. A mask value of 0 means the destination pixel is untouched; a mask value of 1.0 means full compositing occurs.

If the destination pixel falls outside the mask world's bounds (accounting for offset), the behavior depends on the implementation but typically treats out-of-bounds areas as unmasked (fully affected).

## The PF_CompositeMode Structure

Since `PF_MaskWorld` is always used alongside `PF_CompositeMode`, here is the composite mode structure for reference:

```c
typedef struct {
    PF_TransferMode  xfer;        // Blend/transfer mode (PF_Xfer_*)
    A_long           rand_seed;   // Random seed for PF_Xfer_DISSOLVE_RANDOMIZED
    A_u_char         opacity;     // 0-255 opacity
    PF_Boolean       rgb_only;    // If true, alpha is not composited (ignored for MULTIPLY_ALPHA modes)
    A_u_short        opacitySu;   // 16-bit opacity for deep color
} PF_CompositeMode;
```

The mask modulates the `opacity` field -- conceptually, the effective opacity at each pixel is `opacity * mask_value`.

## Code Examples

### Basic Masked Composite with transfer_rect

```c
static PF_Err
MaskedComposite(
    PF_InData       *in_data,
    PF_EffectWorld  *src,
    PF_EffectWorld  *mask_buffer,
    PF_EffectWorld  *dst)
{
    PF_Err err = PF_Err_NONE;

    // Set up the composite mode
    PF_CompositeMode comp_mode;
    AEFX_CLR_STRUCT(comp_mode);
    comp_mode.xfer    = PF_Xfer_COPY;
    comp_mode.opacity = 255;            // Full opacity (before mask)
    comp_mode.rgb_only = FALSE;

    // Set up the mask world
    PF_MaskWorld mask_world;
    AEFX_CLR_STRUCT(mask_world);
    mask_world.mask         = *mask_buffer;
    mask_world.offset.h     = 0;    // Mask is aligned with destination
    mask_world.offset.v     = 0;
    mask_world.what_is_mask = PF_MaskFlag_NONE;  // Use alpha channel

    // Define the source rectangle (full source)
    PF_Rect src_rect;
    src_rect.left   = 0;
    src_rect.top    = 0;
    src_rect.right  = src->width;
    src_rect.bottom = src->height;

    // Composite with mask
    err = in_data->utils->transfer_rect(
        in_data->effect_ref,
        PF_Quality_HI,
        PF_MF_Alpha_STRAIGHT,      // mode flags
        PF_Field_FRAME,            // full frame, not fielded
        &src_rect,
        src,
        &comp_mode,
        &mask_world,               // pass the mask
        0, 0,                      // destination offset
        dst);

    return err;
}
```

### Inverted Luminance Mask

```c
static PF_Err
InvertedLuminanceMask(
    PF_InData       *in_data,
    PF_EffectWorld  *src,
    PF_EffectWorld  *luma_mask,
    PF_EffectWorld  *dst)
{
    PF_Err err = PF_Err_NONE;

    PF_CompositeMode comp_mode;
    AEFX_CLR_STRUCT(comp_mode);
    comp_mode.xfer    = PF_Xfer_COPY;
    comp_mode.opacity = 200;    // Slightly less than full

    // Use luminance of the mask pixels, then invert
    PF_MaskWorld mask_world;
    AEFX_CLR_STRUCT(mask_world);
    mask_world.mask         = *luma_mask;
    mask_world.offset.h     = 0;
    mask_world.offset.v     = 0;
    mask_world.what_is_mask = PF_MaskFlag_LUMINANCE | PF_MaskFlag_INVERTED;

    PF_Rect src_rect = { 0, 0, src->width, src->height };

    err = in_data->utils->transfer_rect(
        in_data->effect_ref,
        PF_Quality_HI,
        PF_MF_Alpha_PREMUL,
        PF_Field_FRAME,
        &src_rect,
        src,
        &comp_mode,
        &mask_world,
        0, 0,
        dst);

    return err;
}
```

### Offset Mask for Partial Coverage

```c
static PF_Err
OffsetMaskComposite(
    PF_InData       *in_data,
    PF_EffectWorld  *src,
    PF_EffectWorld  *small_mask,    // Mask smaller than destination
    A_long          mask_x,         // Where to position the mask
    A_long          mask_y,
    PF_EffectWorld  *dst)
{
    PF_Err err = PF_Err_NONE;

    PF_CompositeMode comp_mode;
    AEFX_CLR_STRUCT(comp_mode);
    comp_mode.xfer    = PF_Xfer_MULTIPLY_ALPHA;
    comp_mode.opacity = 255;

    // Position the mask at an offset within the destination
    PF_MaskWorld mask_world;
    AEFX_CLR_STRUCT(mask_world);
    mask_world.mask         = *small_mask;
    mask_world.offset.h     = mask_x;   // Horizontal offset in destination space
    mask_world.offset.v     = mask_y;   // Vertical offset in destination space
    mask_world.what_is_mask = PF_MaskFlag_NONE;

    PF_Rect src_rect = { 0, 0, src->width, src->height };

    err = in_data->utils->transfer_rect(
        in_data->effect_ref,
        PF_Quality_HI,
        PF_MF_Alpha_PREMUL,
        PF_Field_FRAME,
        &src_rect,
        src,
        &comp_mode,
        &mask_world,
        0, 0,
        dst);

    return err;
}
```

### Masked Transform with Motion Blur

```c
static PF_Err
MaskedTransform(
    PF_InData       *in_data,
    PF_EffectWorld  *src,
    PF_EffectWorld  *mask_buffer,
    A_FpLong        rotation_degrees,
    PF_EffectWorld  *dst)
{
    PF_Err err = PF_Err_NONE;

    PF_CompositeMode comp_mode;
    AEFX_CLR_STRUCT(comp_mode);
    comp_mode.xfer    = PF_Xfer_COPY;
    comp_mode.opacity = 255;

    PF_MaskWorld mask_world;
    AEFX_CLR_STRUCT(mask_world);
    mask_world.mask         = *mask_buffer;
    mask_world.offset.h     = 0;
    mask_world.offset.v     = 0;
    mask_world.what_is_mask = PF_MaskFlag_NONE;

    // Build a rotation matrix
    A_FpLong radians = rotation_degrees * PF_RAD_PER_DEGREE;
    A_FpLong cos_val = cos(radians);
    A_FpLong sin_val = sin(radians);

    // Center of the image
    A_FpLong cx = src->width / 2.0;
    A_FpLong cy = src->height / 2.0;

    // Rotation around center: translate to origin, rotate, translate back
    PF_FloatMatrix matrix;
    AEFX_CLR_STRUCT(matrix);
    matrix.mat[0][0] = cos_val;
    matrix.mat[0][1] = -sin_val;
    matrix.mat[0][2] = cx - cos_val * cx + sin_val * cy;
    matrix.mat[1][0] = sin_val;
    matrix.mat[1][1] = cos_val;
    matrix.mat[1][2] = cy - sin_val * cx - cos_val * cy;
    matrix.mat[2][0] = 0;
    matrix.mat[2][1] = 0;
    matrix.mat[2][2] = 1.0;

    PF_Rect dest_rect = { 0, 0, dst->width, dst->height };

    err = in_data->utils->transform_world(
        in_data->effect_ref,
        PF_Quality_HI,
        PF_MF_Alpha_PREMUL,
        PF_Field_FRAME,
        src,
        &comp_mode,
        &mask_world,
        &matrix,
        1,                  // num_matrices (1 = no motion blur)
        TRUE,               // src2dst_matrix
        &dest_rect,
        dst);

    return err;
}
```

### Creating a Procedural Mask

```c
static PF_Err
CreateRadialMask(
    PF_InData       *in_data,
    A_long          width,
    A_long          height,
    A_FpLong        radius,
    PF_EffectWorld  *mask_world)
{
    PF_Err err = PF_Err_NONE;

    // Allocate the mask world
    err = in_data->utils->new_world(
        in_data->effect_ref,
        width, height,
        PF_NewWorldFlag_CLEAR_PIXELS,
        mask_world);

    if (err) return err;

    A_FpLong cx = width / 2.0;
    A_FpLong cy = height / 2.0;

    // Fill with a radial gradient in the alpha channel
    for (A_long y = 0; y < height; y++) {
        PF_Pixel8 *row = (PF_Pixel8 *)((char *)mask_world->data +
                          y * mask_world->rowbytes);
        for (A_long x = 0; x < width; x++) {
            A_FpLong dx = x - cx;
            A_FpLong dy = y - cy;
            A_FpLong dist = sqrt(dx * dx + dy * dy);
            A_FpLong mask_val = 1.0 - (dist / radius);
            if (mask_val < 0.0) mask_val = 0.0;
            if (mask_val > 1.0) mask_val = 1.0;

            row[x].alpha = (A_u_char)(mask_val * 255.0);
            row[x].red   = 0;
            row[x].green = 0;
            row[x].blue  = 0;
        }
    }

    return err;
}
```

## Passing NULL for No Mask

Both `transfer_rect` and `transform_world` accept `NULL` for the `mask_world0` parameter. When NULL, no masking is applied and the composite occurs uniformly across the entire affected region, modulated only by the `opacity` in `PF_CompositeMode`.

```c
// No mask -- simple unmasked composite
err = in_data->utils->transfer_rect(
    in_data->effect_ref,
    PF_Quality_HI,
    PF_MF_Alpha_PREMUL,
    PF_Field_FRAME,
    &src_rect,
    src,
    &comp_mode,
    NULL,           // No mask
    0, 0,
    dst);
```

## Pitfalls and Warnings

> **Pitfall: Mask world size mismatch.** The mask world does not need to be the same size as the source or destination. Use the `offset` field to position it correctly. However, if the mask is smaller than the composited region and does not cover all destination pixels, pixels outside the mask's coverage may behave as if unmasked. Always ensure your mask covers the region you intend to affect.

> **Pitfall: Forgetting to clear the mask world.** If you allocate a mask world and do not initialize it, the contents are undefined. Use `PF_NewWorldFlag_CLEAR_PIXELS` when allocating, or explicitly fill the mask with your desired values before use.

> **Pitfall: Pixel format consistency.** The mask world is a `PF_EffectWorld` and must have the same pixel format (8-bit, 16-bit, or 32-bit) as the source and destination worlds you are compositing. Mixing pixel formats between the mask and the compositing worlds will produce incorrect results or crashes.

> **Pitfall: Disposing the mask world.** If you allocate a mask world with `new_world`, you must dispose of it with `dispose_world` when done. Failing to do so leaks memory.

> **Pitfall: Rowbytes vs. width.** When writing pixels into the mask world, always use `mask_world->rowbytes` to advance between rows, not `mask_world->width * sizeof(PF_Pixel8)`. The row stride may include padding.

## See Also

- `AE_EffectCB.h` -- `PF_MaskWorld`, `PF_MaskFlags`, `transfer_rect`, `transform_world` definitions
- `PF_CompositeMode` in `AE_Effect.h` -- Transfer modes and opacity
- `PF_TransferMode` / `PF_Xfer_*` constants -- Available blend modes for compositing
