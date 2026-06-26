# The AE Transform Pipeline: From Layer to Comp to Screen

This document explains how coordinates flow through After Effects' transform system, covering every coordinate space a plugin developer encounters and the functions that convert between them.

## Overview of Coordinate Spaces

After Effects uses four distinct coordinate spaces that form a pipeline:

```
Layer Space  -->  Comp Space  -->  View/Frame Space  -->  Screen Space
  (source)       (composition)     (panel pixels)       (OS pixels)
```

| Space | Origin | Units | Used By |
|-------|--------|-------|---------|
| **Layer Space** | Top-left of the full-resolution source | Full-res pixels | Render, pixel processing, effect params |
| **Comp Space** | Top-left of the composition | Comp-resolution pixels | Composition layout, AEGP transforms |
| **View/Frame Space** | Top-left of the panel viewport | Screen pixels (zoomed/panned) | Custom UI drawing, mouse events |
| **Screen Space** | Top-left of the OS window/monitor | Device pixels | Raw OS-level input |

## Layer Space

Layer space is where your effect does its work. When you receive `PF_Cmd_RENDER`, the input buffer (`params[0]`) and output buffer (`output`) are in layer space.

### Key Fields in PF_InData

```c
// Full-resolution dimensions of the source layer (NOT the buffer dimensions)
in_data->width;    // Full-res width
in_data->height;   // Full-res height

// Downsample factors -- the ratio between full-res and actual buffer size
in_data->downsample_x;  // PF_RationalScale (num/den)
in_data->downsample_y;  // PF_RationalScale (num/den)
```

The actual buffer dimensions in `params[0]->u.ld` (the `PF_EffectWorld`) may be smaller than `width`/`height` due to downsampling. The relationship is:

```
buffer_width  = in_data->width  * downsample_x.num / downsample_x.den
buffer_height = in_data->height * downsample_y.num / downsample_y.den
```

### origin_x / origin_y (output_origin)

```c
in_data->output_origin_x;
in_data->output_origin_y;
```

These fields are non-zero only when your effect has set `PF_OutFlag_I_EXPAND_BUFFER` and changed the output size during `PF_Cmd_FRAME_SETUP`. They describe where the top-left corner of the **input** buffer sits within the **output** buffer.

For example, if your effect expands the buffer by 10 pixels on every side, `output_origin_x` and `output_origin_y` will both be 10. The input image starts at pixel (10, 10) within your expanded output buffer.

### pre_effect_source_origin

```c
in_data->pre_effect_source_origin_x;
in_data->pre_effect_source_origin_y;
```

These are non-zero only during frame calls (`PF_Cmd_FRAME_SETUP`, `PF_Cmd_RENDER`, `PF_Cmd_FRAME_SETDOWN`) when a **preceding** effect on the same layer has expanded its output buffer. They tell you where the original source image begins within the input buffer you receive.

> **Pitfall**: If you stack two buffer-expanding effects, the second effect sees `pre_effect_source_origin` reflecting the first effect's expansion, but `output_origin` reflects its own expansion. These are independent offsets.

### extent_hint

```c
in_data->extent_hint;  // PF_Rect
```

The `extent_hint` is the intersection of the visible portions of the input and output layers, expressed in layer-space coordinates (adjusted for downsampling). For effects that do not geometrically distort the image, iterating only over `extent_hint` pixels is sufficient and provides a significant performance gain.

```c
// Efficient render loop using extent_hint
for (int y = in_data->extent_hint.top; y < in_data->extent_hint.bottom; y++) {
    for (int x = in_data->extent_hint.left; x < in_data->extent_hint.right; x++) {
        // Process pixel at (x, y)
    }
}
```

> **Pitfall**: The extent_hint coordinates are in downsampled layer space, matching the buffer dimensions -- not full-resolution layer space.

## Downsample Factors

Downsampling is one of the most error-prone areas for plugin developers. When the user sets the composition viewer to Half, Third, Quarter, or Custom resolution, AE renders at reduced resolution for speed.

### How Downsampling Affects Different Things

| What | Adjusted Automatically? | What You Must Do |
|------|------------------------|-----------------|
| Input/output buffer size | Yes | Nothing |
| Layer param buffers | Yes | Nothing |
| Slider/angle params | **No** | Scale by downsample factor |
| Point params | Yes (values adjusted) | Nothing |
| extent_hint | Yes | Nothing |
| in_data->width/height | **No** (full-res always) | Divide by factor for buffer size |

### Scaling Scalar Parameters

Slider values that represent pixel distances must be manually adjusted for downsampling:

```c
// A blur radius slider value needs downsampling
PF_FpLong radius = params[MY_RADIUS]->u.fs_d.value;

// Scale to match current buffer resolution
PF_FpLong ds_radius = radius * (PF_FpLong)in_data->downsample_x.num
                              / (PF_FpLong)in_data->downsample_x.den;
```

> **Pitfall**: Point parameters (`PF_Param_POINT`) are automatically adjusted by AE to match the downsampled buffer. Slider values are not. If your slider represents a distance in pixels, you must scale it yourself.

### The Downsample Factor Structure

```c
typedef struct {
    A_long  num;  // numerator
    A_long  den;  // denominator
} PF_RationalScale;
```

At full resolution: `num=1, den=1`. At half resolution: `num=1, den=2`. Custom resolutions may produce other values.

## Comp Space

Comp space is the coordinate system of the composition itself. Layer transforms (position, scale, rotation, anchor point) define how a layer maps from layer space into comp space.

### Layer-to-Comp Transform via Event Callbacks

During `PF_Cmd_EVENT`, the `PF_EventCallbacks` structure provides coordinate transform functions:

```c
typedef struct {
    void *refcon;

    // Convert a point from layer space to comp space
    PF_Err (*layer_to_comp)(void *refcon,
                            PF_ContextH context,
                            A_long curr_time,
                            A_long time_scale,
                            PF_FixedPoint *pt);  // in-out

    // Convert a point from comp space to layer space
    PF_Err (*comp_to_layer)(void *refcon,
                            PF_ContextH context,
                            A_long curr_time,
                            A_long time_scale,
                            PF_FixedPoint *pt);  // in-out

    // Get the full comp-to-layer 3x3 matrix
    PF_Err (*get_comp2layer_xform)(void *refcon,
                                   PF_ContextH context,
                                   A_long curr_time,
                                   A_long time_scale,
                                   A_long *exists,
                                   PF_FloatMatrix *c2l);

    // Get the full layer-to-comp 3x3 matrix
    PF_Err (*get_layer2comp_xform)(void *refcon,
                                   PF_ContextH context,
                                   A_long curr_time,
                                   A_long time_scale,
                                   PF_FloatMatrix *l2c);

    // Convert source (layer) coords to frame (view) coords
    PF_Err (*source_to_frame)(void *refcon, PF_ContextH context, PF_FixedPoint *pt);

    // Convert frame (view) coords to source (layer) coords
    PF_Err (*frame_to_source)(void *refcon, PF_ContextH context, PF_FixedPoint *pt);

    // ...
} PF_EventCallbacks;
```

These callbacks are accessed through `event_extraP->cbs`.

### Important Notes on PF_FixedPoint

All coordinate transform callbacks use `PF_FixedPoint` (16.16 fixed-point format):

```c
// Convert integer coordinates to fixed-point
PF_FixedPoint pt;
pt.x = INT2FIX(my_x);  // or FLOAT2FIX(my_x_float)
pt.y = INT2FIX(my_y);

// Call the transform
event_extraP->cbs.layer_to_comp(
    event_extraP->cbs.refcon,
    event_extraP->contextH,
    in_data->current_time,
    in_data->time_scale,
    &pt);

// Convert back
int comp_x = FIX2INT(pt.x);
int comp_y = FIX2INT(pt.y);
// Or for sub-pixel precision:
float comp_xf = FIX_2_FLOAT(pt.x);
```

## View/Frame Space

Frame space (also called view space) represents the actual pixel coordinates within the composition or layer panel as the user sees it. This accounts for zoom level, scroll position, and view transforms.

The `source_to_frame` and `frame_to_source` callbacks convert between layer space and frame space. This is critical for custom UI drawing and hit testing.

### Coordinate Conversion for Comp Window Overlays

When drawing in the comp window, you must chain transforms:

```
Layer Space  --[layer_to_comp]-->  Comp Space  --[source_to_frame]-->  Frame Space
```

The CCU SDK example demonstrates this pattern:

```c
void Source2FrameRect(
    PF_InData       *in_data,
    PF_EventExtra   *event_extraP,
    PF_FixedRect    *fx_frameRP,
    PF_FixedPoint   *bounding_boxFiPtAP)  // Array of 4 corner points
{
    // Set up the four corners in layer space
    bounding_boxFiPtAP[0].x = fx_frameRP->left;
    bounding_boxFiPtAP[0].y = fx_frameRP->top;
    bounding_boxFiPtAP[1].x = fx_frameRP->right;
    bounding_boxFiPtAP[1].y = fx_frameRP->top;
    bounding_boxFiPtAP[2].x = fx_frameRP->right;
    bounding_boxFiPtAP[2].y = fx_frameRP->bottom;
    bounding_boxFiPtAP[3].x = fx_frameRP->left;
    bounding_boxFiPtAP[3].y = fx_frameRP->bottom;

    // If in comp window, first transform layer -> comp
    if (PF_Window_COMP == (*event_extraP->contextH)->w_type) {
        for (int i = 0; i < 4; ++i) {
            event_extraP->cbs.layer_to_comp(
                event_extraP->cbs.refcon,
                event_extraP->contextH,
                in_data->current_time,
                in_data->time_scale,
                &bounding_boxFiPtAP[i]);
        }
    }

    // Then transform to frame (view) coordinates
    for (int j = 0; j < 4; ++j) {
        event_extraP->cbs.source_to_frame(
            event_extraP->cbs.refcon,
            event_extraP->contextH,
            &bounding_boxFiPtAP[j]);
    }
}
```

### Converting Mouse Clicks Back to Layer Space

The reverse pipeline for mouse input:

```
Frame Space  --[frame_to_source]-->  Comp/Layer Space  --[comp_to_layer]-->  Layer Space
```

For the **comp window**:

```c
void CompFrame2Layer(
    PF_InData       *in_data,
    PF_EventExtra   *event_extraP,
    PF_Point        *framePt,
    PF_Point        *lyrPt,
    PF_FixedPoint   *fix_lyrFiPt)
{
    fix_lyrFiPt->x = INT2FIX(framePt->h);
    fix_lyrFiPt->y = INT2FIX(framePt->v);

    // Frame -> source (comp space for comp window)
    event_extraP->cbs.frame_to_source(
        event_extraP->cbs.refcon,
        event_extraP->contextH,
        fix_lyrFiPt);

    // Comp -> layer
    event_extraP->cbs.comp_to_layer(
        event_extraP->cbs.refcon,
        event_extraP->contextH,
        in_data->current_time,
        in_data->time_scale,
        fix_lyrFiPt);

    lyrPt->h = fix_lyrFiPt->x;
    lyrPt->v = fix_lyrFiPt->y;
}
```

For the **layer window**, skip `comp_to_layer` since `frame_to_source` already returns layer-space coordinates:

```c
void LayerFrame2Layer(
    PF_InData       *in_data,
    PF_EventExtra   *event_extraP,
    PF_Point        *framePt,
    PF_Point        *lyrPt,
    PF_FixedPoint   *fix_lyrPtP)
{
    fix_lyrPtP->x = INT2FIX(framePt->h);
    fix_lyrPtP->y = INT2FIX(framePt->v);

    event_extraP->cbs.frame_to_source(
        event_extraP->cbs.refcon,
        event_extraP->contextH,
        fix_lyrPtP);

    lyrPt->h = FIX2INT(fix_lyrPtP->x);
    lyrPt->v = FIX2INT(fix_lyrPtP->y);
}
```

## AEGP Layer-to-World Transform

For AEGP plugins or effects using the AEGP suites, `AEGP_GetLayerToWorldXform` provides a 4x4 matrix that converts from layer space to world (composition) space, including 3D transforms:

```c
AEGP_SuiteHandler suites(in_data->pica_basicP);

A_Matrix4 layer_to_world;
suites.LayerSuite8()->AEGP_GetLayerToWorldXform(
    layer_ref,
    &time,
    &layer_to_world);
```

This matrix accounts for:
- Layer position (including Z)
- Anchor point
- Scale
- Rotation (X, Y, Z, Orientation)
- Parent chain transforms

> **Pitfall**: `PF_PathDataSuite` returns path vertices that are partially transformed -- they include 2D transforms (XY translation, Z rotation, shear from parenting) but NOT 3D transforms (Z translation, Orientation, X/Y Rotation). If you need true layer-space vertices to pass through `AEGP_GetLayerToWorldXform`, you must account for this discrepancy.

## Buffer Expansion and Coordinate Spaces

When an effect sets `PF_OutFlag_I_EXPAND_BUFFER` and adjusts `out_data->width` and `out_data->height` during `PF_Cmd_FRAME_SETUP`, the coordinate spaces shift.

### How Expansion Changes Coordinates

Before expansion, the input and output buffers are the same size and aligned. After expansion:

```
Output buffer (expanded):
+--------------------------------------+
|          expanded region             |
|    +----------------------------+    |
|    |    original input layer    |    |
|    |   (at output_origin_x/y)  |    |
|    +----------------------------+    |
|                                      |
+--------------------------------------+

output_origin_x = expansion_left
output_origin_y = expansion_top
```

### Practical Example: 10px Blur Expansion

```c
static PF_Err FrameSetup(
    PF_InData   *in_data,
    PF_OutData  *out_data)
{
    A_long expand = 10;  // pixels of expansion on each side

    // Request a larger output buffer
    out_data->width  = in_data->width  + 2 * expand;
    out_data->height = in_data->height + 2 * expand;

    // Tell AE where the input sits within the output
    out_data->origin.h = expand;
    out_data->origin.v = expand;

    return PF_Err_NONE;
}
```

During render, you read the origin from `in_data`:

```c
A_long ox = in_data->output_origin_x;  // will be 10
A_long oy = in_data->output_origin_y;  // will be 10

// To find where input pixel (0,0) is in the output buffer:
// output_buffer[oy][ox] == input pixel (0,0)
```

### Stacked Expanding Effects

When multiple expanding effects are stacked on a layer, each one sees:

- `pre_effect_source_origin_x/y`: Where the original source starts in the input buffer (accumulated from all preceding expansions)
- `output_origin_x/y`: Where the input buffer starts in this effect's output buffer (only this effect's expansion)

## Summary: Which Transform When

| Situation | Transform Needed |
|-----------|-----------------|
| Drawing overlay in comp window | `layer_to_comp` then `source_to_frame` |
| Drawing overlay in layer window | `source_to_frame` only |
| Mouse click in comp window to layer coords | `frame_to_source` then `comp_to_layer` |
| Mouse click in layer window to layer coords | `frame_to_source` only |
| Scalar param representing pixel distance | Multiply by `downsample_x.num / downsample_x.den` |
| Accessing pixels after buffer expansion | Offset by `output_origin_x/y` |
| Converting to world space (3D) | Use `AEGP_GetLayerToWorldXform` |
| Getting the transform as a matrix | Use `get_layer2comp_xform` or `get_comp2layer_xform` |

## Common Pitfalls

1. **Forgetting downsample scaling for sliders.** Point params are auto-adjusted; sliders are not. If your 100px radius looks correct at full-res but wrong at half-res, you forgot to scale.

2. **Using the wrong transform chain for the window type.** Always check `(*event_extraP->contextH)->w_type` before choosing your transform path. Comp window needs `layer_to_comp` + `source_to_frame`. Layer window needs only `source_to_frame`.

3. **Confusing full-res dimensions with buffer dimensions.** `in_data->width/height` are always full resolution. The actual buffer you iterate over is `params[0]->u.ld.width/height` (or the `PF_EffectWorld` dimensions).

4. **Ignoring pre_effect_source_origin.** If your effect follows another buffer-expanding effect, the input coordinates are shifted. Always account for `pre_effect_source_origin_x/y` when mapping between the logical layer and the physical buffer.

5. **Fixed-point confusion.** The event callbacks use 16.16 fixed-point (`PF_FixedPoint`). Always use `INT2FIX`/`FIX2INT` or `FLOAT2FIX`/`FIX_2_FLOAT` macros for conversion. Treating fixed-point values as plain integers produces coordinates 65536x too large.

6. **Pixel aspect ratio.** When converting scalar distances, account for non-square pixels with `in_data->pixel_aspect_ratio`. Horizontal distances should be multiplied by `par.den / par.num` to appear correct on screen.
