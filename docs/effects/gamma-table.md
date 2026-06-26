# LUT and Gamma Processing: The Gamma_Table Example

The `Gamma_Table` SDK example demonstrates how to implement efficient per-pixel color correction using a pre-computed lookup table (LUT) rather than calculating expensive math per pixel. Located at `Examples/Effect/Gamma_Table/`, it is one of the simplest yet most instructive examples in the entire SDK.

## What This Example Demonstrates

| Concept | SDK Feature Used |
|---|---|
| Sequence data (persistent per-instance state) | `PF_Cmd_SEQUENCE_SETUP`, `PF_NEW_HANDLE`, `PF_DISPOSE_HANDLE` |
| LUT-based pixel processing | Pre-computed 256-entry `A_u_char` table |
| Pixel iteration callback | `PF_ITERATE` macro |
| Extent hint optimization | `PF_OutFlag_USE_OUTPUT_EXTENT` with `in_data->extent_hint` |
| Fixed-point parameter | `PF_ADD_FIXED` slider |
| Progress reporting | Progress height passed to `PF_ITERATE` |
| Error display | `PF_OutFlag_DISPLAY_ERROR_MESSAGE` |
| MFR compatibility | `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` |

---

## Architecture Overview

The plugin's flow can be summarized as:

```
SEQUENCE_SETUP  -->  Allocate 256-byte LUT + gamma value in sequence_data
       |
    RENDER      -->  If gamma changed, regenerate LUT (once)
       |             Then iterate all pixels using PF_ITERATE + LUT
       v
SEQUENCE_SETDOWN --> Free the handle
```

The key insight is that the expensive `pow()` call happens only 256 times when the gamma parameter changes -- not once per pixel. During actual rendering, each pixel channel is transformed via a single array lookup.

---

## The Gamma_Table Data Structure

The header defines two structures:

```cpp
typedef struct {
    PF_Fixed    gamma_val;    // Cached gamma parameter value (fixed-point)
    A_u_char    lut[256];     // The lookup table: input byte -> output byte
} Gamma_Table;

typedef struct {
    unsigned char    *lut;    // Pointer to the LUT, passed as refcon to iterate
} GammaInfo;
```

`Gamma_Table` is stored in `sequence_data` -- it persists across renders for the same effect instance. `GammaInfo` is a stack-local struct passed as the `refcon` to `PF_ITERATE`, containing just a pointer to the LUT.

---

## Sequence Data Lifecycle

### Allocation (PF_Cmd_SEQUENCE_SETUP)

```cpp
static PF_Err SequenceSetup(
    PF_InData *in_data, PF_OutData *out_data,
    PF_ParamDef *params[], PF_LayerDef *output)
{
    Gamma_Table *g_tableP;
    A_long iL = 0;

    if (out_data->sequence_data) {
        PF_DISPOSE_HANDLE(out_data->sequence_data);
    }
    out_data->sequence_data = PF_NEW_HANDLE(sizeof(Gamma_Table));
    if (!out_data->sequence_data) {
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    // Initialize to identity (gamma = 1.0 in fixed-point)
    g_tableP = *(Gamma_Table**)out_data->sequence_data;
    g_tableP->gamma_val = (1L << 16);  // 1.0 in 16.16 fixed-point

    for (iL = 0; iL <= PF_MAX_CHAN8; iL++) {
        g_tableP->lut[iL] = (A_u_char)iL;  // Identity mapping
    }

    return PF_Err_NONE;
}
```

Key details:

- The handle is double-indirected: `*(Gamma_Table**)out_data->sequence_data` dereferences first the `PF_Handle` then the underlying pointer.
- The initial LUT is an identity table (input = output) corresponding to gamma = 1.0.
- `PF_MAX_CHAN8` is 255, and the loop uses `<=` to fill all 256 entries (indices 0 through 255).

### Teardown (PF_Cmd_SEQUENCE_SETDOWN)

```cpp
static PF_Err SequenceSetdown(
    PF_InData *in_data, PF_OutData *out_data,
    PF_ParamDef *params[], PF_LayerDef *output)
{
    if (in_data->sequence_data) {
        PF_DISPOSE_HANDLE(in_data->sequence_data);
        out_data->sequence_data = NULL;
    }
    return PF_Err_NONE;
}
```

Note the asymmetry: reads from `in_data->sequence_data`, writes `NULL` to `out_data->sequence_data`.

### Resetup (PF_Cmd_SEQUENCE_RESETUP)

```cpp
static PF_Err SequenceResetup(
    PF_InData *in_data, PF_OutData *out_data,
    PF_ParamDef *params[], PF_LayerDef *output)
{
    if (!in_data->sequence_data) {
        return SequenceSetup(in_data, out_data, params, output);
    }
    return PF_Err_NONE;
}
```

`SEQUENCE_RESETUP` is called when a project is re-opened or when the effect is unflatted. If the data is missing, it falls back to full setup.

---

## LUT Generation: The Gamma Math

During `Render`, the plugin checks whether the current gamma parameter differs from the cached value. If so, it regenerates the LUT:

```cpp
if (g_tableP->gamma_val != params[GAMMA_GAMMA]->u.fd.value) {

    double temp, gamma;

    g_tableP->gamma_val = params[GAMMA_GAMMA]->u.fd.value;
    gamma = (PF_FpLong)g_tableP->gamma_val / (double)(1L << 16);
    gamma = 1.0 / gamma;

    for (xL = 0; xL <= PF_MAX_CHAN8; ++xL) {
        temp = PF_POW((PF_FpLong)xL / 255.0, gamma);
        g_tableP->lut[xL] = (A_u_char)(temp * 255.0);
    }
}
```

### The Math Explained

The standard gamma correction formula is:

```
output = input^(1/gamma)
```

Step by step:

1. **Convert parameter from fixed-point**: The `PF_ADD_FIXED` slider stores values in 16.16 fixed-point format. Dividing by `(1L << 16)` (65536) converts to floating-point.
2. **Invert gamma**: `gamma = 1.0 / gamma` produces the exponent for the power function.
3. **Normalize to [0, 1]**: `xL / 255.0` maps the 8-bit input range to the unit interval.
4. **Apply power function**: `PF_POW(normalized, 1/gamma)` performs the actual correction.
5. **Scale back to [0, 255]**: Multiply by 255 and truncate to `A_u_char`.

| Input Gamma | Effect |
|---|---|
| 1.0 | No change (identity) |
| < 1.0 | Darkens midtones |
| > 1.0 | Brightens midtones |
| 2.2 | Typical sRGB gamma encoding |

> **Pitfall**: The cast `(A_u_char)(temp * 255.0)` truncates rather than rounds. For production code, you may want `(A_u_char)(temp * 255.0 + 0.5)` to get proper rounding.

---

## The Pixel Iterator: GammaFunc

The per-pixel callback is minimal -- it simply looks up each channel in the table:

```cpp
static PF_Err GammaFunc(
    void *refcon,
    A_long x, A_long y,
    PF_Pixel *inP, PF_Pixel *outP)
{
    PF_Err    err = PF_Err_NONE;
    GammaInfo *giP = (GammaInfo*)refcon;

    if (giP) {
        outP->alpha = inP->alpha;          // Alpha passes through unchanged
        outP->red   = giP->lut[inP->red];
        outP->green = giP->lut[inP->green];
        outP->blue  = giP->lut[inP->blue];
    } else {
        err = PF_Err_BAD_CALLBACK_PARAM;
    }
    return err;
}
```

This is the function signature required by `PF_ITERATE`:

```
PF_Err (*PF_IteratePixel8Func)(
    void *refcon,    // Your custom data
    A_long x,        // Pixel X coordinate
    A_long y,        // Pixel Y coordinate
    PF_Pixel *inP,   // Source pixel
    PF_Pixel *outP   // Destination pixel
);
```

Key points:

- **Alpha is preserved unchanged.** Gamma correction applies to visible color channels only.
- **Each channel lookup is O(1).** Three array reads per pixel instead of three `pow()` calls.
- The `refcon` pointer carries the LUT into the callback without globals, making this thread-safe and MFR-compatible.

---

## Invoking PF_ITERATE

```cpp
progress_heightL = in_data->extent_hint.top - in_data->extent_hint.bottom;
gamma_info.lut = g_tableP->lut;

ERR(PF_ITERATE(0,
               progress_heightL,
               &params[GAMMA_INPUT]->u.ld,   // Source layer
               &in_data->extent_hint,         // Region to process
               (void*)&gamma_info,            // Refcon (our LUT wrapper)
               GammaFunc,                     // Pixel callback
               output));                      // Destination layer
```

### PF_ITERATE Parameters

| Parameter | Value | Purpose |
|---|---|---|
| `progress_base` | `0` | Starting progress value |
| `progress_final` | `progress_heightL` | Total lines for progress bar |
| `src` | `&params[GAMMA_INPUT]->u.ld` | Source layer data |
| `area` | `&in_data->extent_hint` | Rectangle to iterate over |
| `refcon` | `(void*)&gamma_info` | Custom data for callback |
| `pix_fn` | `GammaFunc` | Per-pixel function |
| `dst` | `output` | Output layer data |

The `PF_ITERATE` macro wraps `in_data->utils->iterate()`. After Effects handles:

- Walking scanlines with proper row byte stride
- Calling the progress callback
- Potentially distributing work across threads (when the plugin opts in via `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`)

---

## Extent Hint Optimization

Before iterating, the example clears any region of the output that lies outside the intersection of input and output extent hints:

```cpp
if (in_data->extent_hint.left   != output->extent_hint.left   ||
    in_data->extent_hint.top    != output->extent_hint.top     ||
    in_data->extent_hint.right  != output->extent_hint.right   ||
    in_data->extent_hint.bottom != output->extent_hint.bottom) {

    ERR(PF_FILL(NULL,
                &output->extent_hint,
                output));
}
```

This is required because `PF_OutFlag_USE_OUTPUT_EXTENT` tells AE that the plugin will only write within the extent hint. Any pixels outside that region must be zeroed.

> **Pitfall**: Forgetting `PF_FILL` when using `PF_OutFlag_USE_OUTPUT_EXTENT` leads to garbage pixels in non-rendered areas, typically visible as flickering artifacts.

---

## The Gamma=1.0 Fast Path

When gamma is exactly 1.0, the plugin skips LUT generation entirely and performs a direct copy:

```cpp
if (params[GAMMA_GAMMA]->u.fd.value == (1L << 16)) {
    ERR(PF_COPY(&params[0]->u.ld,
                output,
                NULL,
                NULL));
}
```

This is a common optimization pattern: check for identity parameter values and short-circuit with a copy. `PF_COPY` is faster than iterating pixel-by-pixel since AE can use optimized `memcpy` paths internally.

---

## Performance: LUT vs Per-Pixel Math

The LUT approach trades a small amount of memory (256 bytes) for a dramatic reduction in per-pixel computation:

| Approach | Cost per pixel | Cost for 1920x1080 |
|---|---|---|
| Per-pixel `pow()` | 3 x `pow()` calls | ~6.2 million `pow()` calls |
| LUT | 3 x array lookups | ~6.2 million array reads (L1 cache) |

### Why LUTs Work for 8-bit

- An 8-bit channel has exactly 256 possible values, so the table is complete.
- 256 bytes fits entirely in L1 cache on any modern CPU.
- Array indexing is a single cycle; `pow()` typically costs 50-100+ cycles.
- The LUT is generated once and reused across all pixels and all frames at the same gamma value.

### When LUTs Do Not Apply

| Bit Depth | LUT Feasibility | Notes |
|---|---|---|
| 8-bit | Excellent (256 entries) | This example |
| 16-bit | Possible (32,769 entries) | Use `AEFX_ChannelDepthTpl.h` for interpolated lookup |
| 32-bit float | Impractical | Continuous range; must compute per pixel |

For 16-bit, the SDK provides `PixelTraits<PF_Pixel16>::LutFunc()` in `AEFX_ChannelDepthTpl.h` which uses a smaller table with linear interpolation between entries, controlled by `PF_TABLE_BITS`.

---

## Global Setup and Flags

```cpp
static PF_Err GlobalSetup(
    PF_InData *in_data, PF_OutData *out_data,
    PF_ParamDef *params[], PF_LayerDef *output)
{
    out_data->my_version = PF_VERSION(MAJOR_VERSION, MINOR_VERSION,
                                       BUG_VERSION, STAGE_VERSION,
                                       BUILD_VERSION);

    out_data->out_flags |= PF_OutFlag_PIX_INDEPENDENT |
                           PF_OutFlag_USE_OUTPUT_EXTENT;

    out_data->out_flags2 = PF_OutFlag2_SUPPORTS_THREADED_RENDERING;

    return PF_Err_NONE;
}
```

| Flag | Hex | Meaning |
|---|---|---|
| `PF_OutFlag_PIX_INDEPENDENT` | `0x40` | Each output pixel depends only on the corresponding input pixel |
| `PF_OutFlag_USE_OUTPUT_EXTENT` | `0x400` | Plugin will respect and use extent hints for optimization |
| `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` | `0x8000000` | Plugin is safe for Multi-Frame Rendering (MFR) |

`PF_OutFlag_PIX_INDEPENDENT` is critical here: it tells AE that this effect does not sample neighboring pixels, enabling internal optimizations.

---

## PiPL Resource

The PiPL at `Gamma_TablePiPL.r` declares:

```
AE_Effect_Global_OutFlags { 1088 }
```

The value 1088 in decimal is `0x440`, which equals `PF_OutFlag_PIX_INDEPENDENT (0x40) | PF_OutFlag_USE_OUTPUT_EXTENT (0x400)`. These must match the flags set in `GlobalSetup`.

```
AE_Effect_Global_OutFlags_2 { 0x8000000 }
```

This is `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`.

---

## Limitations of This Example

1. **8-bit only**: The effect does not declare `PF_OutFlag_DEEP_COLOR_AWARE` and thus only processes 8-bit color. For 16-bit or 32-bit support, you would need additional iterate functions for `PF_Pixel16` and `PF_PixelFloat`.

2. **No SmartFX**: This uses the legacy render path (`PF_Cmd_RENDER`), not `PF_Cmd_SMART_PRE_RENDER` / `PF_Cmd_SMART_RENDER`. A modern implementation would use SmartFX for better caching behavior.

3. **Sequence data under MFR**: While the example sets `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`, note that sequence data shared across threads requires careful handling. In this case, the LUT is effectively read-only during iteration (it is regenerated only when the gamma value changes), so it is safe. However, modifying `g_tableP->gamma_val` and regenerating the LUT is not atomic. In a truly concurrent scenario, you would want `const_sequence_data` or compute the LUT on the stack.

4. **No rounding**: As noted above, the float-to-int conversion truncates rather than rounds.

---

## Adapting This Pattern to Your Plugin

To use a LUT approach in your own effect:

```cpp
// 1. Define your LUT structure
typedef struct {
    PF_FpLong   cached_param;     // Track which parameter generated this LUT
    A_u_char    lut[256];
} MyLUT;

// 2. Regenerate only when the parameter changes
if (my_lut->cached_param != current_param_value) {
    my_lut->cached_param = current_param_value;
    for (int i = 0; i <= PF_MAX_CHAN8; i++) {
        my_lut->lut[i] = your_transfer_function(i, current_param_value);
    }
}

// 3. Use PF_ITERATE with a simple lookup callback
ERR(PF_ITERATE(0, height, src, area, (void*)my_lut, MyPixelFunc, dst));
```

This pattern applies to any point operation: brightness, contrast, levels, curves, color grading, posterization, solarization, and more.

---

## Source Files

| File | Purpose |
|---|---|
| `Gamma_Table.cpp` | All effect logic: setup, render, entry point |
| `Gamma_Table.h` | Structs, enums, constants, entry point declaration |
| `Gamma_TablePiPL.r` | PiPL resource definition |

---

## See Also

- `AEFX_ChannelDepthTpl.h` -- Template-based LUT handling for 8-bit and 16-bit with interpolation
- `PF_ITERATE` / `iterate` callback documentation
- The **Convolutron** example for neighborhood-based (non-LUT) processing
- The **GLator** example for GPU-accelerated alternatives to per-pixel iteration
