# Mask After Plugin Implementation

This document explains how the "masks after plugin" feature can be implemented in an After Effects plugin, allowing masks to be applied after the effect rather than before it.

## Solution: Dual-Checkout Strategy

The solution uses a professional technique employed by various third-party plugins: checking out both the masked and unmasked versions of the input layer.

### How It Works

1. **Checkout unmasked input** via `PF_CHECKOUT_PARAM` - returns layer data before masks are applied
2. **Checkout masked input** via `checkout_layer_pixels` - returns layer data after masks are applied
3. **Use unmasked for plugin**
4. **Apply mask alpha after rendering** - multiply output alpha by the mask's alpha channel

## Implementation Details

### Core Components

#### 1. Dual-Checkout in SmartRender

```cpp
// 1. Get UNMASKED input using PF_CHECKOUT_PARAM (full texture data)
PF_ParamDef unmasked_input_param;
AEFX_CLR_STRUCT(unmasked_input_param);
ERR(PF_CHECKOUT_PARAM(in_data, 0, in_data->current_time,
                      in_data->time_step, in_data->time_scale,
                      &unmasked_input_param));

// 2. Get MASKED input for mask bounds/alpha (if available)
PF_EffectWorld *masked_input_worldP = NULL;
PF_Err mask_checkout_err = extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &masked_input_worldP);

// Use unmasked for primary processing (full texture)
input_worldP = &unmasked_input_param.u.ld;
```

**Important:** Inputs must be checked out **before** the output in SmartRender mode, or AE will return error "must checkout at least one input before checking out output".

#### 2. Mask Alpha Application Helper

```cpp
static void ApplyMaskAlphaToOutput(PF_LayerDef* output, PF_LayerDef* mask_layer)
```

This helper function:
- Detects bit depth of both output and mask layer (8-bit, 16-bit, or 32-bit float)
- Reads the alpha channel from the masked input
- Multiplies the output's alpha by the mask's alpha
- Handles all three bit depths correctly


## Key Technical Points

### 1. Checkout Order Matters

In SmartRender mode, **you must checkout at least one input before checking out the output**, otherwise AE returns error 25::244.

Correct order:
```cpp
// 1. Checkout inputs
PF_CHECKOUT_PARAM(...)           // Unmasked
checkout_layer_pixels(...)       // Masked

// 2. Checkout output
checkout_output(...)
```

### 2. PF_CHECKOUT_PARAM vs checkout_layer_pixels

- **PF_CHECKOUT_PARAM(param_index=0)** - Returns layer **before** masks/effects are applied
- **checkout_layer_pixels(param_index=0)** - Returns layer **after** masks are applied (SmartFX only)

This difference is the key to the dual-checkout strategy.

### 3. When Masks Don't Exist

When there's no mask on the layer, both checkout methods return identical data. The mask alpha will be 1.0 (fully opaque) everywhere, so the multiplication has no effect - correct behavior!

### 4. Bit Depth Handling

The mask alpha application must handle all three bit depths:

- **8-bit:** `alpha / 255.0f`
- **16-bit:** `alpha / (float)PF_MAX_CHAN16` where `PF_MAX_CHAN16 = 32768`
- **32-bit float:** `alpha` (already 0.0-1.0 range)

### 5. MFR Compatibility

The implementation is Multi-Frame Rendering (MFR) compatible:

- No modifications to `sequence_data` during render
- No setting of `out_data->out_flags` during SmartRender
- Both inputs are checked out per-frame
- Flag `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` is set

### 6. Global OutFlags

The implementation does **not** require `PF_OutFlag2_DEPENDS_ON_UNREFERENCED_MASKS`. This flag would cause updates when masks change even if not directly used, but since we're using `checkout_layer_pixels` which directly references the mask, it's unnecessary.

> **Note:** The combined hex values for outflags may not add up correctly in all cases; always verify against current SDK headers.

Example outflags2 combination:
- `PF_OutFlag2_FLOAT_COLOR_AWARE`
- `PF_OutFlag2_SUPPORTS_SMART_RENDER`
- `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`
- `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32`
- `PF_OutFlag2_AUTOMATIC_WIDE_TIME_INPUT`
- `PF_OutFlag2_I_MIX_GUID_DEPENDENCIES`

## End-to-End Parameter Flow

This section documents the complete parameter flow from SmartRender through to the renderer, showing exactly what parameters must be passed to make masks work correctly.

### Critical Parameters for Mask Support

For masks to work, the following parameters must flow through the system:

1. **output_origin_x/y** - The buffer's origin in layer space
2. **masked_input_worldP** - The masked input buffer with alpha channel
3. **masked_rect bounds** - The extent_hint from the masked input
4. **output extent_hint** - The output buffer's position in layer space

### Parameter Flow: SmartRender -> Renderer

#### Step 1: Dual-Checkout in SmartRender

```cpp
// 1. Get UNMASKED input using PF_CHECKOUT_PARAM (full texture data)
PF_ParamDef unmasked_input_param;
AEFX_CLR_STRUCT(unmasked_input_param);
ERR(PF_CHECKOUT_PARAM(in_data, 0, in_data->current_time,
                      in_data->time_step, in_data->time_scale,
                      &unmasked_input_param));

// 2. Get MASKED input for mask bounds/alpha (if available)
PF_EffectWorld *masked_input_worldP = NULL;
PF_Err mask_checkout_err = extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &masked_input_worldP);

// Use unmasked for primary processing (full texture)
input_worldP = &unmasked_input_param.u.ld;

// Extract mask bounds from masked input's extent_hint
if (mask_checkout_err == PF_Err_NONE && masked_input_worldP) {
    // The masked input extent_hint tells us WHERE in layer space this cropped buffer sits
    if (masked_input_worldP->extent_hint.left != 0 || masked_input_worldP->extent_hint.top != 0) {
        mask_offset_x = masked_input_worldP->extent_hint.left;
        mask_offset_y = masked_input_worldP->extent_hint.top;
    }
}

// 3. NOW checkout the output (must be after input checkout)
ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));
```

**Key Points:**
- `masked_input_worldP->extent_hint` provides the mask bounds (left, top, right, bottom)
- These bounds tell us where the cropped masked buffer sits in layer space
- The output must be checked out AFTER the inputs (AE requirement)

#### Step 2: Resolve Output Origin

```cpp
// Resolve output origin for UV mapping (handles cropped/masked outputs)
int render_origin_x = 0;
int render_origin_y = 0;
ResolveOutputOriginForMasks(in_data, output_worldP,
                            masked_rect_left, masked_rect_top,
                            masked_rect_right, masked_rect_bottom,
                            &render_origin_x, &render_origin_y);
```

The `ResolveOutputOriginForMasks` function determines the output buffer's origin using this fallback chain:

```cpp
static void ResolveOutputOriginForMasks(const PF_InData* in_data,
                                        const PF_LayerDef* output_worldP,
                                        A_long masked_rect_left,
                                        A_long masked_rect_top,
                                        A_long masked_rect_right,
                                        A_long masked_rect_bottom,
                                        int* out_origin_x,
                                        int* out_origin_y)
{
    // Primary source: in_data->output_origin_x/y (provided by After Effects)
    int origin_x = in_data ? (int)in_data->output_origin_x : 0;
    int origin_y = in_data ? (int)in_data->output_origin_y : 0;

    // Fallback 1: If AE didn't provide origin, try output extent_hint
    if (origin_x == 0 && origin_y == 0 && output_worldP) {
        int extent_left = output_worldP->extent_hint.left;
        int extent_top = output_worldP->extent_hint.top;
        if (extent_left != 0 || extent_top != 0) {
            origin_x = extent_left;
            origin_y = extent_top;
        } else {
            // Fallback 2: Use masked_rect bounds if dimensions match
            int masked_w = (int)(masked_rect_right - masked_rect_left);
            int masked_h = (int)(masked_rect_bottom - masked_rect_top);
            if (masked_w == output_worldP->width && masked_h == output_worldP->height) {
                origin_x = (int)masked_rect_left;
                origin_y = (int)masked_rect_top;
            }
        }
    }

    *out_origin_x = origin_x;
    *out_origin_y = origin_y;
}
```

**Fallback Chain:**
1. **Primary:** `in_data->output_origin_x/y` (most reliable, from AE)
2. **Fallback 1:** `output_worldP->extent_hint.left/top` (if origin is 0,0)
3. **Fallback 2:** `masked_rect_left/top` (if dimensions match and extent_hint is also 0,0)

#### Step 3: Pass Parameters to Renderer

**For CPU Path:**

```cpp
renderUVUnwrapped_generic(output, texture_layer, mesh, view_matrix, proj_matrix,
                        in_data->width, in_data->height,
                        total_film_offset_h, total_film_offset_v, show_diagnostics, supersampling, square_mode,
                        &camera, test_var, render_style,
                        max_render_distance, extreme_coord_threshold, far_geometry_threshold,
                        uv_scale_x, uv_scale_y,
                        (int)in_data->output_origin_x, (int)in_data->output_origin_y);  // <- CRITICAL!
```

**For GPU Path:**

```cpp
// Set up render params structure
RenderParams render_params;
render_params.output_width = output_worldP->width;
render_params.output_height = output_worldP->height;
render_params.comp_width = in_data->width;
render_params.comp_height = in_data->height;

// Resolve and set output origin (handles cropped/masked outputs)
int render_origin_x = 0;
int render_origin_y = 0;
ResolveOutputOriginForMasks(in_data, output_worldP,
                            masked_rect_left, masked_rect_top,
                            masked_rect_right, masked_rect_bottom,
                            &render_origin_x, &render_origin_y);

render_params.output_origin_x = render_origin_x;  // <- CRITICAL!
render_params.output_origin_y = render_origin_y;  // <- CRITICAL!
render_params.uv_scale_x = uv_scale_x;
render_params.uv_scale_y = uv_scale_y;
// ... other parameters ...

// Call GPU renderer
gpu_err = g_gpuRenderer->RenderUnwrap(
    &input_layer, &output_layer,
    gpu_vertices, gpu_triangles,
    view_matrix.m, proj_matrix.m,
    render_params);  // <- All params passed via struct
```

#### Step 4: Coordinate Transformation in Renderer

This is where the magic happens - the output_origin transforms layer coordinates to buffer coordinates:

```cpp
// Map UV to LAYER space coordinates
float layer_x = scaled_u * uv_layer_scale_x + uv_layer_offset_x;
float layer_y = (1.0f - scaled_v) * uv_layer_scale_y + uv_layer_offset_y;

// Convert layer coordinates to output buffer coordinates
// by subtracting the buffer's origin in layer space
uv_pixels_f[i][0] = layer_x - output_origin_x;  // <- CRITICAL!
uv_pixels_f[i][1] = layer_y - output_origin_y;  // <- CRITICAL!

uv_pixels[i][0] = (int)(uv_pixels_f[i][0] + 0.5f);
uv_pixels[i][1] = (int)(uv_pixels_f[i][1] + 0.5f);
```

**Why This Matters:**
- Without masks: `output_origin = (0, 0)`, so `buffer_coord = layer_coord - 0 = layer_coord`
- With masks: `output_origin = mask bounds origin`, so we correctly map into the cropped buffer
- Example: If mask starts at layer position (100, 50), and we're rendering layer pixel (150, 75):
  - `buffer_x = 150 - 100 = 50` (correct position in the cropped buffer)
  - `buffer_y = 75 - 50 = 25` (correct position in the cropped buffer)

#### Step 5: Apply Mask Alpha

After rendering completes, multiply the output alpha by the mask's alpha:

```cpp
// Apply mask alpha if present (both CPU and GPU paths)
if (masked_input_worldP && masked_input_worldP->data) {
    PF_LayerDef mask_layer;
    AEFX_CLR_STRUCT(mask_layer);
    mask_layer.data = masked_input_worldP->data;
    mask_layer.width = masked_input_worldP->width;
    mask_layer.height = masked_input_worldP->height;
    mask_layer.rowbytes = masked_input_worldP->rowbytes;
    mask_layer.world_flags = masked_input_worldP->world_flags;
    mask_layer.extent_hint = masked_input_worldP->extent_hint;

    // Use the same output origin passed to the shader
    int output_origin_x = render_params.output_origin_x;
    int output_origin_y = render_params.output_origin_y;

    ApplyMaskAlphaToOutputPositioned(&output_layer, &mask_layer,
                                    (int)masked_rect_left, (int)masked_rect_top,
                                    output_origin_x, output_origin_y);
}
```

The `ApplyMaskAlphaToOutputPositioned` function:
- Iterates through the output buffer
- For each pixel, calculates its position in layer space
- Finds the corresponding pixel in the masked input buffer
- Multiplies output alpha by mask alpha
- Handles all three bit depths (8-bit, 16-bit, 32-bit float)

### Complete Parameter Checklist

To implement mask support in a plugin, ensure these parameters flow correctly:

**From SmartRender:**
- `in_data->output_origin_x` and `in_data->output_origin_y` (from After Effects)
- `masked_input_worldP` via `checkout_layer_pixels()`
- `masked_input_worldP->extent_hint` (mask bounds: left, top, right, bottom)
- `output_worldP->extent_hint` (output buffer position in layer space)

**To Renderer:**
- `output_origin_x` and `output_origin_y` (resolved via fallback chain)
- `comp_width` and `comp_height` (from `in_data->width/height`)
- `output->width` and `output->height` (actual buffer dimensions)

**In Renderer:**
- Transform: `buffer_coord = layer_coord - output_origin`
- Use this transformation for BOTH reading from input and writing to output

**After Rendering:**
- `ApplyMaskAlphaToOutputPositioned()` with mask bounds and output origin
- Pass the SAME `output_origin_x/y` used during rendering

### GPU Path Implementation

The GPU path uses the same parameters via a render params structure:

```cpp
struct RenderParams {
    int output_width;        // Output buffer dimensions
    int output_height;
    int comp_width;          // Comp dimensions (layer space)
    int comp_height;
    int output_origin_x;     // <- CRITICAL for masks
    int output_origin_y;     // <- CRITICAL for masks
    float uv_scale_x;
    float uv_scale_y;
    // ... other params ...
};
```

The GPU compute shader performs the same coordinate transformation:
```glsl
// In uv_unwrap.comp shader:
vec2 layer_coord = /* ... UV to layer space transform ... */;
vec2 buffer_coord = layer_coord - vec2(output_origin_x, output_origin_y);
```

## Testing

To verify masks work correctly:

1. Apply the plugin to a layer with texture
2. Add a mask to the layer (rectangular, ellipse, etc.)
3. In Unwrap mode, the full texture should be visible in the UV space
4. The mask should only affect the **output** visibility, not clip the texture
5. Move/animate the mask - the effect should re-render correctly

## References

- After Effects SDK: Effect Basics > SmartFX
- After Effects SDK: PF_CHECKOUT_PARAM documentation
- Community discussion: [Adobe forums on mask rendering](https://community.adobe.com)

## Credits

Implementation based on SDK research and industry-standard techniques used by third-party plugins when they need unmasked texture data with masked output.
