# Layer Checkout Patterns

This document provides a structured guide to checking out layer parameters in After Effects plugins, covering both the standard `PF_CHECKOUT_PARAM` path and the SmartFX `checkout_layer` / `checkout_layer_pixels` path, with emphasis on temporal effects, coordinate systems, downsample factors, and common patterns.

> For Q&A-style troubleshooting, see also [layer-checkout.md](layer-checkout.md) and [checkout-layer.md](checkout-layer.md).

---

## Table of Contents

- [Two Checkout Systems](#two-checkout-systems)
- [Standard Checkout: PF_CHECKOUT_PARAM](#standard-checkout-pf_checkout_param)
- [SmartFX Checkout: PreRender + SmartRender](#smartfx-checkout-prerender--smartrender)
- [Checking Out at Different Times (Temporal Effects)](#checking-out-at-different-times-temporal-effects)
- [Layer World Dimensions vs Comp Dimensions](#layer-world-dimensions-vs-comp-dimensions)
- [Downsample Factors](#downsample-factors)
- [The Relationship Between Checked-Out Layers and the Render Pipeline](#the-relationship-between-checked-out-layers-and-the-render-pipeline)
- [Checking Out Non-Layer Parameters](#checking-out-non-layer-parameters)
- [Origin and Coordinate Systems](#origin-and-coordinate-systems)
- [Common Patterns](#common-patterns)
- [Pitfalls and Warnings](#pitfalls-and-warnings)

---

## Two Checkout Systems

After Effects provides two fundamentally different layer checkout mechanisms:

| Feature | `PF_CHECKOUT_PARAM` (Standard) | `checkout_layer` / `checkout_layer_pixels` (SmartFX) |
|---|---|---|
| Render path | `PF_Cmd_RENDER` | `PF_Cmd_PRE_RENDER` + `PF_Cmd_SMART_RENDER` |
| Returns | Raw layer pixels (before effects on that layer) | Layer pixels with all prior effects applied |
| Buffer size control | No (AE decides) | Yes (via `PF_RenderRequest`) |
| GPU support | No | Yes (pixel data may be on GPU) |
| Temporal checkout | Yes (specify time) | Yes (specify time) |
| Requires `PF_OutFlag2_SUPPORTS_SMART_RENDER` | No | Yes |

---

## Standard Checkout: PF_CHECKOUT_PARAM

The `PF_CHECKOUT_PARAM` macro checks out a parameter value at a specific time. For layer parameters, this returns a `PF_EffectWorld` (pixel buffer) of the **raw, pre-effects layer**.

### Signature

```cpp
#define PF_CHECKOUT_PARAM(IN_DATA, INDEX, TIME, STEP, SCALE, PARAM)
```

| Parameter | Type | Description |
|---|---|---|
| `IN_DATA` | `PF_InData*` | The `in_data` pointer from your command handler |
| `INDEX` | `A_long` | Parameter index (disk ID from `PF_Cmd_PARAMS_SETUP`) |
| `TIME` | `A_long` | Time to checkout at (in `time_scale` units) |
| `STEP` | `A_long` | Time step (usually `in_data->time_step`) |
| `SCALE` | `A_u_long` | Time scale (usually `in_data->time_scale`) |
| `PARAM` | `PF_ParamDef*` | Output -- filled with the parameter value |

### Basic Usage

```cpp
PF_ParamDef checkout;
AEFX_CLR_STRUCT(checkout);

ERR(PF_CHECKOUT_PARAM(
    in_data,
    PARAM_LAYER,                // your layer parameter index
    in_data->current_time,      // current frame time
    in_data->time_step,
    in_data->time_scale,
    &checkout));

if (!err) {
    PF_EffectWorld *layer_pixels = &checkout.u.ld;

    // Access pixel data
    A_long width  = layer_pixels->width;
    A_long height = layer_pixels->height;
    PF_PixelPtr data = layer_pixels->data;
    A_long rowbytes = layer_pixels->rowbytes;

    // ... process pixels ...
}

// ALWAYS check in, even if there was an error
ERR2(PF_CHECKIN_PARAM(in_data, &checkout));
```

### The ERR2 Pattern

Use `ERR2` (not `ERR`) for checkin so that a checkin error does not mask an earlier processing error:

```cpp
// ERR2 macro: sets err only if err was previously A_Err_NONE
#define ERR2(FUNC)  { A_Err err2 = (FUNC); if (!err) err = err2; }
```

---

## SmartFX Checkout: PreRender + SmartRender

SmartFX provides a two-phase checkout where you declare what you need in PreRender and retrieve pixel data in SmartRender.

### Phase 1: PreRender -- Declare Dependencies

```cpp
static PF_Err PreRender(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_PreRenderExtra *extra)
{
    PF_Err err = PF_Err_NONE;

    PF_RenderRequest req = extra->input->output_request;
    PF_CheckoutResult in_result;

    // Checkout the input layer (index 0)
    ERR(extra->cb->checkout_layer(
        in_data->effect_ref,
        0,                      // param index: 0 = input layer
        CHECKOUT_ID_INPUT,      // unique ID you define (must be >= 0)
        &req,                   // render request (rect, field, etc.)
        in_data->current_time,
        in_data->time_step,
        in_data->time_scale,
        &in_result));

    // Tell AE what we will output
    if (!err) {
        extra->output->result_rect = in_result.result_rect;
        extra->output->max_result_rect = in_result.max_result_rect;
    }

    return err;
}
```

### Phase 2: SmartRender -- Retrieve Pixels

```cpp
static PF_Err SmartRender(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_SmartRenderExtra *extra)
{
    PF_Err err = PF_Err_NONE;
    PF_EffectWorld *input_worldP = NULL;
    PF_EffectWorld *output_worldP = NULL;

    // Get the actual pixel buffers using the checkout IDs from PreRender
    ERR(extra->cb->checkout_layer_pixels(
        in_data->effect_ref,
        CHECKOUT_ID_INPUT,
        &input_worldP));

    ERR(extra->cb->checkout_output(
        in_data->effect_ref,
        &output_worldP));

    if (!err && input_worldP && output_worldP) {
        // Process pixels...
    }

    // Optional early checkin to free memory
    ERR(extra->cb->checkin_layer_pixels(
        in_data->effect_ref,
        CHECKOUT_ID_INPUT));

    return err;
}
```

### Checkout IDs

Each checkout requires a unique ID (`A_long`, must be >= 0). Define them as constants:

```cpp
enum {
    CHECKOUT_ID_INPUT = 0,
    CHECKOUT_ID_MATTE = 1,
    CHECKOUT_ID_LAYER2 = 2,
    // etc.
};
```

> **Warning**: Checkout IDs must be unique across ALL checkouts within a single PreRender/SmartRender pair, including checkouts of the same layer at different times. If you checkout the same layer at multiple times, each needs a distinct ID.

---

## Checking Out at Different Times (Temporal Effects)

Temporal effects (frame blending, echo, time displacement) need to access frames other than the current time.

### Standard Render: Temporal Checkout

```cpp
// Check out previous frame
A_long prev_time = in_data->current_time - in_data->time_step;

PF_ParamDef prev_frame;
AEFX_CLR_STRUCT(prev_frame);
ERR(PF_CHECKOUT_PARAM(
    in_data,
    PARAM_INPUT,        // 0 for the input layer
    prev_time,
    in_data->time_step,
    in_data->time_scale,
    &prev_frame));

// Use prev_frame.u.ld for the pixel data...

ERR2(PF_CHECKIN_PARAM(in_data, &prev_frame));
```

### SmartFX: Temporal Checkout

In PreRender, checkout the same layer at multiple times:

```cpp
// Current frame
ERR(extra->cb->checkout_layer(
    in_data->effect_ref, 0, CHECKOUT_ID_CURRENT,
    &req, in_data->current_time,
    in_data->time_step, in_data->time_scale, &result_current));

// Previous frame
ERR(extra->cb->checkout_layer(
    in_data->effect_ref, 0, CHECKOUT_ID_PREV,
    &req, in_data->current_time - in_data->time_step,
    in_data->time_step, in_data->time_scale, &result_prev));

// Next frame
ERR(extra->cb->checkout_layer(
    in_data->effect_ref, 0, CHECKOUT_ID_NEXT,
    &req, in_data->current_time + in_data->time_step,
    in_data->time_step, in_data->time_scale, &result_next));
```

Then in SmartRender, retrieve each:

```cpp
PF_EffectWorld *current_worldP = NULL, *prev_worldP = NULL, *next_worldP = NULL;
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, CHECKOUT_ID_CURRENT, &current_worldP));
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, CHECKOUT_ID_PREV, &prev_worldP));
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, CHECKOUT_ID_NEXT, &next_worldP));
```

### Temporal Checkout Limits

There is a practical limit to how many frames you can checkout simultaneously. The system is not designed for batch-checking hundreds or thousands of frames. For effects that need many frames (e.g., optical flow), consider:

- Caching results in sequence data across render calls
- Using the Compute Cache system (`AE_ComputeCacheSuite.h`) in newer AE versions
- Processing a limited window of frames

> **Warning**: Checking out more than approximately 10-12 frames simultaneously can exhaust the checkout system, returning errors or NULL data. Design temporal effects to work within this limit.

### Setting PF_OutFlag_WIDE_TIME_INPUT

For temporal effects, you MUST set this flag in `PF_Cmd_GLOBAL_SETUP`:

```cpp
out_data->out_flags |= PF_OutFlag_WIDE_TIME_INPUT;
```

This tells AE that your effect reads frames other than the current time, which affects caching and pre-render behavior.

---

## Layer World Dimensions vs Comp Dimensions

A common source of confusion is the relationship between different dimension values.

### in_data Dimensions

```cpp
in_data->width   // Full-resolution width of the SOURCE LAYER
in_data->height  // Full-resolution height of the SOURCE LAYER
```

These are the full-resolution dimensions of the layer your effect is applied to, BEFORE downsample factors are applied.

### Checked-Out Layer Dimensions

```cpp
PF_EffectWorld *world = &checkout.u.ld;
world->width    // Actual pixel width of the buffer (after downsampling)
world->height   // Actual pixel height of the buffer (after downsampling)
```

These reflect the actual pixel buffer size, which accounts for the current downsample/resolution setting.

### Comp Dimensions

Comp dimensions are not directly available in `PF_InData`. To get them, use AEGP:

```cpp
AEGP_CompH compH;
A_long comp_width, comp_height;
ERR(suites.CompSuite()->AEGP_GetItemFromComp(compH, &itemH));
ERR(suites.ItemSuite()->AEGP_GetItemDimensions(itemH, &comp_width, &comp_height));
```

### PF_CheckoutResult (SmartFX)

In SmartFX, `PF_CheckoutResult` provides additional sizing information:

```cpp
PF_CheckoutResult result;
// After checkout_layer:
result.result_rect;     // Actual available rect
result.max_result_rect; // Maximum possible rect if full frame was requested
result.ref_width;       // Original layer width, pre-effects, without DSF
result.ref_height;      // Original layer height
result.par;             // Pixel aspect ratio
```

---

## Downsample Factors

When the user sets the comp viewer to Half, Third, Quarter, etc., AE applies a downsample factor (DSF). This affects all layer checkout buffers.

### Reading the DSF

```cpp
PF_RationalScale dsf_x = in_data->downsample_x;  // e.g., {1, 2} for half
PF_RationalScale dsf_y = in_data->downsample_y;
```

The DSF is a rational number (numerator/denominator). Common values:

| Resolution | `downsample_x` | `downsample_y` |
|---|---|---|
| Full | 1/1 | 1/1 |
| Half | 1/2 | 1/2 |
| Third | 1/3 | 1/3 |
| Quarter | 1/4 | 1/4 |

### Impact on Pixel Coordinates

Scalar parameters that represent pixel distances must be adjusted:

```cpp
A_FpLong radius_pixels = params[PARAM_RADIUS]->u.fs_d.value;

// Scale by DSF for actual buffer coordinates
A_FpLong scaled_radius = radius_pixels *
    (A_FpLong)in_data->downsample_x.num / (A_FpLong)in_data->downsample_x.den;
```

### Impact on Checked-Out Layers

Checked-out layer buffers are automatically downsampled. The `width` and `height` fields of the `PF_EffectWorld` reflect the downsampled size. You do not need to downsample the buffer yourself.

> **SmartFX Note**: In SmartFX, the downsample factor is already incorporated into the `PF_RenderRequest` rectangles. You do not need to manually apply DSF when working with SmartFX checkout results.

---

## The Relationship Between Checked-Out Layers and the Render Pipeline

### Standard Render (`PF_CHECKOUT_PARAM`)

```
Source footage --> [No effects applied] --> Your checkout receives this
```

`PF_CHECKOUT_PARAM` returns the layer BEFORE any effects are applied. This is true even if the layer has other effects on it. You get the raw source.

For the input layer (index 0 / `PARAM_INPUT`), `params[0]->u.ld` already contains the input with effects from layers below applied. But if you checkout a LAYER PARAMETER (a user-selected layer), you get raw pixels.

### SmartFX Render (`checkout_layer_pixels`)

```
Source footage --> [Prior effects on this layer applied] --> Your checkout receives this
```

`checkout_layer_pixels` returns the layer WITH all prior effects (effects above yours in the stack) applied. This is the key advantage of SmartFX for effect stacking.

### Input Layer (Index 0) vs Layer Parameters

| Checkout Target | Standard Render | SmartFX |
|---|---|---|
| Input layer (index 0) | Available in `params[0]->u.ld` (pre-effects) | `checkout_layer_pixels` (with prior effects) |
| Layer parameter | `PF_CHECKOUT_PARAM` (always raw, pre-effects) | `checkout_layer` + `checkout_layer_pixels` (raw, pre-effects for non-input layers) |

> **Important**: For layer parameters (not the input), both systems return the raw layer. Only the input layer (index 0) differs between standard and SmartFX rendering.

---

## Checking Out Non-Layer Parameters

`PF_CHECKOUT_PARAM` also works for scalar parameters, not just layers. This is useful for getting parameter values at times other than the current frame:

```cpp
PF_ParamDef slider_at_prev;
AEFX_CLR_STRUCT(slider_at_prev);

ERR(PF_CHECKOUT_PARAM(
    in_data,
    PARAM_SLIDER,
    in_data->current_time - in_data->time_step,  // previous frame
    in_data->time_step,
    in_data->time_scale,
    &slider_at_prev));

A_FpLong prev_value = slider_at_prev.u.fs_d.value;

ERR2(PF_CHECKIN_PARAM(in_data, &slider_at_prev));
```

---

## Origin and Coordinate Systems

### Standard Render

In standard render, the output buffer origin is at (0,0) by default. If a prior effect expanded the buffer, `in_data->output_origin_x/y` indicates where the input buffer sits within the output buffer, and `in_data->pre_effect_source_origin_x/y` indicates the origin of the original source within the input buffer.

### SmartFX

In SmartFX, `PF_EffectWorld` has `origin_x` and `origin_y` fields:

```cpp
PF_EffectWorld *world;
// world->origin_x, world->origin_y
// These indicate the position of the top-left corner of the buffer
// in layer coordinate space. Only meaningful for SmartFX checkouts.
```

The `PF_RenderRequest` specifies rectangles in layer coordinates:

```cpp
typedef struct {
    PF_LRect        rect;               // requested output rect in layer coords
    PF_Field        field;              // field to render
    PF_ChannelMask  channel_mask;       // which channels to render
    PF_Boolean      preserve_rgb_of_zero_alpha;
    char            unused[3];
    A_long          reserved[4];
} PF_RenderRequest;
```

---

## Common Patterns

### Pattern: Difference Between Frames

```cpp
// In SmartFX PreRender
ERR(extra->cb->checkout_layer(in_data->effect_ref, 0, ID_CURR,
    &req, in_data->current_time, in_data->time_step, in_data->time_scale, &res_curr));
ERR(extra->cb->checkout_layer(in_data->effect_ref, 0, ID_PREV,
    &req, in_data->current_time - in_data->time_step,
    in_data->time_step, in_data->time_scale, &res_prev));

// Union the result rects
UnionLRect(&res_prev.result_rect, &extra->output->result_rect);

// In SmartRender
PF_EffectWorld *curr = NULL, *prev = NULL;
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, ID_CURR, &curr));
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, ID_PREV, &prev));
// Compute per-pixel difference...
```

### Pattern: Matte Layer Checkout

```cpp
// Checkout a user-selected layer as a matte
PF_ParamDef matte_param;
AEFX_CLR_STRUCT(matte_param);

ERR(PF_CHECKOUT_PARAM(in_data, PARAM_MATTE_LAYER,
    in_data->current_time, in_data->time_step, in_data->time_scale,
    &matte_param));

if (!err && matte_param.u.ld.data) {
    // Matte is available
    PF_EffectWorld *matte = &matte_param.u.ld;
    // Note: matte dimensions may differ from input layer dimensions
    // if the matte source has different dimensions
}

ERR2(PF_CHECKIN_PARAM(in_data, &matte_param));
```

### Pattern: Validating Checkout Results

Always validate checkout results before using them:

```cpp
PF_ParamDef checkout;
AEFX_CLR_STRUCT(checkout);

ERR(PF_CHECKOUT_PARAM(in_data, PARAM_LAYER, ...));

if (!err) {
    PF_EffectWorld *world = &checkout.u.ld;

    // Validate the world is usable
    if (world->data != NULL &&
        world->width > 0 &&
        world->height > 0 &&
        world->rowbytes > 0)
    {
        // Safe to process
    }
}

ERR2(PF_CHECKIN_PARAM(in_data, &checkout));
```

---

## Pitfalls and Warnings

1. **Always check in what you check out.** Every `PF_CHECKOUT_PARAM` must have a matching `PF_CHECKIN_PARAM`. Failure to check in leaks memory and can cause AE instability. Use `ERR2` for the checkin call.

2. **Do not cache checked-out pixel pointers across commands.** The pixel data is only valid for the duration of the current command. After the command returns, the pointer is invalid.

3. **Checkout during SEQUENCE_SETUP/RESETUP/SETDOWN is illegal.** Layer data is not available during these commands. Attempting to checkout will fail or return garbage.

4. **In GPU mode, `data` may be NULL.** When SmartFX runs on the GPU (CUDA, Metal, OpenCL), `PF_EffectWorld.data` is NULL because pixels live on the GPU. You must use GPU-specific APIs to access pixel data.

5. **Layer parameter checkout returns raw (pre-effects) pixels.** Only the input layer (index 0) in SmartFX returns post-effects pixels via `checkout_layer_pixels`. User-selected layer parameters always return raw footage.

6. **Checked-out layers from different sources may have different dimensions.** If the user selects a layer with a different resolution than the comp, the checked-out buffer will have that layer's dimensions (scaled by DSF), not the comp dimensions.

7. **Checking out the same parameter multiple times at the same time is wasteful but not illegal.** However, checking out the same layer at many different times simultaneously has practical limits (approximately 10-12 simultaneous checkouts).

8. **`extent_hint` is informational.** For input layers, `extent_hint` describes the bounding box of opaque pixels. For output, it describes the area that needs rendering. You can use it for optimization but should not rely on it for correctness.

9. **Rowbytes may differ from width * bytes_per_pixel.** Always use `rowbytes` for pointer arithmetic, never compute stride from width. Buffers may have padding for alignment.

10. **SmartFX checkout_layer_pixels can fail if PreRender did not checkout_layer.** You must first declare your intent in PreRender via `checkout_layer` before requesting pixels in SmartRender via `checkout_layer_pixels`.
