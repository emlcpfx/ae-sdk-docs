# Iterate Functions: iterate vs iterate_origin vs iterate_origin_non_clip_src

The After Effects SDK provides a family of **iterate** functions that invoke a per-pixel callback across a region of source and destination worlds. Choosing the correct variant is critical for correctness, particularly when your effect expands buffers, reads outside source bounds, or needs to scale to modern multi-core hardware.

This document covers every iterate variant, including the suite-based versions and the `NO_MAX_THREADS` alternatives, with code examples and a decision tree.

---

## Overview of Iterate Variants

| Function | Origin Offset | Clips Source Reads | Per-Pixel Callback | Notes |
|---|---|---|---|---|
| `iterate` | No | Yes | Yes | Simplest form; src and dst must be same size or overlap |
| `iterate_origin` | Yes | Yes | Yes | Handles offset between src and dst coordinate systems |
| `iterate_origin_non_clip_src` | Yes | No | Yes | Allows reading outside source bounds (returns zero/transparent) |
| `iterate_lut` | No | Yes | No (LUT-based) | Fast 256-entry lookup table per channel, 8-bit only |
| `iterate_generic` | N/A | N/A | No (generic) | Thread-parallel generic work distribution, no pixel worlds |

---

## PF_ITERATE (Standard Per-Pixel Processing)

The basic iterate walks every pixel in the intersection of `src` and `dst` (or a specified `area` rect), calling your pixel function once for each pixel.

### Macro

```c
#define PF_ITERATE(PROG_BASE, PROG_FINAL, SRC, RCT_P, REFCON, PIX_FUNC, DST)
```

### Suite Function (PF_Iterate8Suite1 / PF_Iterate8Suite2)

```c
PF_Err (*iterate)(
    PF_InData       *in_data,
    A_long          progress_base,
    A_long          progress_final,
    PF_EffectWorld  *src,
    const PF_Rect   *area,          /* pass NULL for all pixels */
    void            *refcon,
    PF_Err          (*pix_fn)(void* refcon, A_long x, A_long y,
                              PF_Pixel *in, PF_Pixel *out),
    PF_EffectWorld  *dst);
```

### Parameters

| Parameter | Description |
|---|---|
| `in_data` | Automatically passed by the macro; provides the effect reference and progress context |
| `progress_base` | Starting value for the progress bar (0 to begin) |
| `progress_final` | Maximum value for the progress bar. **Pass 0 to disable progress reporting.** |
| `src` | Source world to read from. **Pass NULL to iterate only over dst** (useful for in-place generation) |
| `area` | Rectangle to iterate over. **Pass NULL for all pixels** in the overlap region |
| `refcon` | Opaque pointer passed through to your pixel function (see refcon pattern below) |
| `pix_fn` | Your per-pixel callback function |
| `dst` | Destination world to write to |

### When To Use

Use `PF_ITERATE` when your source and destination worlds have the same dimensions and coordinate system, which is the common case for simple color corrections, adjustments, and effects that do not expand their output buffer.

### Code Example

```c
// Data structure passed via refcon
typedef struct {
    PF_FpLong brightness;  // Must be read-only or synchronized
} BrightnessInfo;

static PF_Err BrightnessPixelFunc(
    void        *refcon,
    A_long      x,
    A_long      y,
    PF_Pixel    *inP,
    PF_Pixel    *outP)
{
    PF_Err err = PF_Err_NONE;
    BrightnessInfo *info = (BrightnessInfo *)refcon;

    A_long r = (A_long)(inP->red   * info->brightness);
    A_long g = (A_long)(inP->green * info->brightness);
    A_long b = (A_long)(inP->blue  * info->brightness);

    outP->alpha = inP->alpha;
    outP->red   = (A_u_char)MIN(r, PF_MAX_CHAN8);
    outP->green = (A_u_char)MIN(g, PF_MAX_CHAN8);
    outP->blue  = (A_u_char)MIN(b, PF_MAX_CHAN8);

    return err;
}

// In your Render function:
static PF_Err Render(PF_InData *in_data, PF_OutData *out_data,
                     PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;
    BrightnessInfo info;
    info.brightness = params[BRIGHTNESS_SLIDER]->u.fd.value / 100.0;

    // Use extent_hint for optimal iteration area
    err = PF_ITERATE(0,
                     output->extent_hint.bottom - output->extent_hint.top,
                     &params[0]->u.ld,   // input layer
                     &output->extent_hint,
                     (void *)&info,
                     BrightnessPixelFunc,
                     output);
    return err;
}
```

---

## PF_ITERATE_ORIGIN (Buffer Expansion with Origin Offset)

When an effect expands its output buffer (by responding to `PF_Cmd_FRAME_SETUP` with larger dimensions, or via SmartFX `PreRender`), the source and destination worlds have **different coordinate systems**. The `origin` parameter tells iterate how to map between them.

### Macro

```c
#define PF_ITERATE_ORIGIN(PROG_BASE, PROG_FINAL, SRC, RCT_P, OR, REFCON, PIX_FUNC, DST)
```

### Suite Function

```c
PF_Err (*iterate_origin)(
    PF_InData       *in_data,
    A_long          progress_base,
    A_long          progress_final,
    PF_EffectWorld  *src,
    const PF_Rect   *area,
    const PF_Point  *origin,        /* offset from dst coords to src coords */
    void            *refcon,
    PF_Err          (*pix_fn)(void* refcon, A_long x, A_long y,
                              PF_Pixel *in, PF_Pixel *out),
    PF_EffectWorld  *dst);
```

### The Origin Parameter

The `origin` point specifies the offset from destination coordinates to source coordinates. If your effect expanded the output by `N` pixels on each side, the origin is typically `{N, N}` -- meaning pixel (0,0) in the destination corresponds to pixel (N,N) in the source... but more precisely, the origin tells the iterator where the source world's (0,0) pixel falls within the destination world's coordinate space.

For SmartFX, the origin values come directly from the `PF_LayerDef` that was checked out:

```c
PF_Point origin;
origin.h = input_worldP->origin_x;
origin.v = input_worldP->origin_y;
```

### When To Use

Use `PF_ITERATE_ORIGIN` when:
- Your effect expands its output buffer in `PF_Cmd_FRAME_SETUP`
- You are using SmartFX and the checked-out input has a different origin than the output
- Source and destination worlds have the same logical content but different buffer offsets

**Important**: This variant still **clips source reads** to the source world bounds. If your pixel function's `(x, y)` maps to a coordinate outside the source world, the `inP` pixel will be clipped to the nearest edge pixel. This is usually fine for effects like drop shadows or borders where out-of-bounds reads should not occur.

### Code Example

```c
// Effect that adds a 10-pixel border (expand in FRAME_SETUP)
static PF_Err Render(PF_InData *in_data, PF_OutData *out_data,
                     PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;
    A_long border = 10;

    PF_Point origin;
    origin.h = border;  // src (0,0) is at dst (border, border)
    origin.v = border;

    err = PF_ITERATE_ORIGIN(0,
                            output->extent_hint.bottom - output->extent_hint.top,
                            &params[0]->u.ld,
                            &output->extent_hint,
                            &origin,
                            NULL,           // no refcon needed
                            CopyPixelFunc,
                            output);
    return err;
}
```

---

## PF_ITERATE_ORIGIN_NON_SRC_CLIP (No Source Clipping)

This is the most permissive iterate variant. It works exactly like `iterate_origin` but does **not** clip source coordinate reads to the source world bounds. When your pixel function receives `(x, y)` coordinates that map outside the source buffer, the `inP` pointer will point to pixels with **zero alpha (transparent black)** rather than clamped edge pixels.

### Macro

```c
#define PF_ITERATE_ORIGIN_NON_SRC_CLIP(PROG_BASE, PROG_FINAL, SRC, RCT_P, OR, REFCON, PIX_FUNC, DST)
```

### Suite Function

```c
PF_Err (*iterate_origin_non_clip_src)(
    PF_InData       *in_data,
    A_long          progress_base,
    A_long          progress_final,
    PF_EffectWorld  *src,
    const PF_Rect   *area,
    const PF_Point  *origin,
    void            *refcon,
    PF_Err          (*pix_fn)(void* refcon, A_long x, A_long y,
                              PF_Pixel *in, PF_Pixel *out),
    PF_EffectWorld  *dst);
```

### When To Use

This variant is **essential** for:
- **Blur effects**: Gaussian blur, box blur, directional blur -- anywhere you need to read neighboring pixels that may fall outside the source buffer
- **Displacement effects**: Where the sampling location is computed dynamically and may land outside the source
- **Convolution effects** with custom kernels where border handling matters
- Any effect where reading a clamped edge pixel would produce visible artifacts (repeated edges, color bleeding)

### Behavior Difference (Critical)

| Variant | Out-of-bounds source read behavior |
|---|---|
| `iterate` | Clips x,y to source bounds; `inP` points to nearest valid pixel |
| `iterate_origin` | Same as `iterate` but with coordinate offset applied first |
| `iterate_origin_non_clip_src` | Returns transparent black `{0, 0, 0, 0}` for out-of-bounds reads |

> **WARNING**: If you use `iterate` or `iterate_origin` for a blur effect with buffer expansion, you will see repeated edge pixels ("edge clamping artifacts") instead of the smooth fade to transparent that users expect.

### Code Example: Blur-style Effect

```c
typedef struct {
    PF_EffectWorld *src;
    A_long radius;
} BlurInfo;

static PF_Err BlurPixelFunc(
    void        *refcon,
    A_long      x,
    A_long      y,
    PF_Pixel    *inP,       // This pixel may be transparent black if outside src
    PF_Pixel    *outP)
{
    // Note: for a real blur, you would NOT use the iterate callback's inP.
    // Instead you would read from the source world directly using the
    // refcon to access the src world pointer, sampling neighbors around (x,y).
    // The non-clip-src variant matters because the x,y coordinates
    // given to you will span the full destination area, even where
    // there is no corresponding source pixel.

    BlurInfo *info = (BlurInfo *)refcon;
    // ... perform box/gaussian blur sampling from info->src ...
    return PF_Err_NONE;
}

static PF_Err Render(PF_InData *in_data, PF_OutData *out_data,
                     PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;
    BlurInfo info;
    info.src = &params[0]->u.ld;
    info.radius = 10;

    PF_Point origin;
    origin.h = info.radius;
    origin.v = info.radius;

    err = PF_ITERATE_ORIGIN_NON_SRC_CLIP(
        0,
        output->extent_hint.bottom - output->extent_hint.top,
        info.src,
        &output->extent_hint,
        &origin,
        (void *)&info,
        BlurPixelFunc,
        output);

    return err;
}
```

---

## PF_ITERATE_LUT (Lookup Table Iteration)

`PF_ITERATE_LUT` is a highly optimized iterate variant that applies four independent 256-entry lookup tables (one per channel: A, R, G, B) without requiring a per-pixel callback function. This is the fastest way to apply color corrections that can be expressed as per-channel curves.

### Macro

```c
#define PF_ITERATE_LUT(PROG_BASE, PROG_FINAL, SRC, RCT_P, A_LUT, R_LUT, G_LUT, B_LUT, DST)
```

### Suite Function

```c
PF_Err (*iterate_lut)(
    PF_InData       *in_data,
    A_long          progress_base,
    A_long          progress_final,
    PF_EffectWorld  *src,
    const PF_Rect   *area,          /* pass NULL for all pixels */
    A_u_char        *a_lut0,        /* pass NULL for identity */
    A_u_char        *r_lut0,        /* pass NULL for identity */
    A_u_char        *g_lut0,        /* pass NULL for identity */
    A_u_char        *b_lut0,        /* pass NULL for identity */
    PF_EffectWorld  *dst);
```

### Key Details

- Each LUT is a 256-entry `A_u_char` array mapping input value to output value.
- Pass `NULL` for any channel's LUT to apply an identity transform (pass-through) for that channel.
- **8-bit only** -- there is no 16-bit or float variant. For higher bit depths, use `iterate` with a per-pixel callback.
- Very fast because the host can optimize the inner loop without function call overhead per pixel.

### Code Example: Gamma Correction via LUT

```c
static PF_Err ApplyGamma(PF_InData *in_data, PF_OutData *out_data,
                         PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;

    A_u_char r_lut[256], g_lut[256], b_lut[256];
    PF_FpLong gamma = params[GAMMA_SLIDER]->u.fd.value;
    PF_FpLong inv_gamma = 1.0 / gamma;

    for (int i = 0; i < 256; i++) {
        A_u_char val = (A_u_char)(pow((double)i / 255.0, inv_gamma) * 255.0 + 0.5);
        r_lut[i] = val;
        g_lut[i] = val;
        b_lut[i] = val;
    }

    err = PF_ITERATE_LUT(0, 0,             // no progress bar
                         &params[0]->u.ld,
                         NULL,              // all pixels
                         NULL,              // alpha: identity (pass through)
                         r_lut, g_lut, b_lut,
                         output);
    return err;
}
```

---

## iterate_generic (Thread-Parallel Generic Work)

`iterate_generic` is fundamentally different from all other iterate variants. It does not operate on pixel worlds at all. Instead, it distributes a range of integer iterations across multiple threads, calling your function with a thread index and iteration index.

### Suite Function

```c
PF_Err (*iterate_generic)(
    A_long  iterationsL,    /* total iterations, or PF_Iterations_ONCE_PER_PROCESSOR */
    void    *refconPV,
    PF_Err  (*fn_func)(
        void    *refconPV,
        A_long  thread_indexL,  // only call abort/progress from thread 0
        A_long  i,              // current iteration index
        A_long  iterationsL));  // total iterations (resolved, never -1)
```

### PF_Iterations_ONCE_PER_PROCESSOR

```c
#define PF_Iterations_ONCE_PER_PROCESSOR  (-1L)
```

When you pass `-1` as `iterationsL`, the host will call your function once per available CPU core. The callback receives the resolved total in its `iterationsL` parameter (it will never see `-1`). This is ideal for effects that want to subdivide work across cores without knowing the core count in advance.

> **Note**: The SDK header also defines the misspelling `PF_Iteratations_ONCE_PER_PROCESSOR` for source compatibility.

### Threading Rules

- **Only thread 0** (`thread_indexL == 0`) may call `PF_ABORT()` or `PF_PROGRESS()`.
- Your refcon data must be **read-only or properly synchronized** because the function may be called simultaneously from multiple threads.
- The callback is responsible for its own pixel access; it receives no `inP`/`outP` pointers.

### Code Example: Parallel Row Processing

```c
typedef struct {
    PF_EffectWorld  *src;
    PF_EffectWorld  *dst;
    A_long          width;
    A_long          height;
    PF_FpLong       intensity;
} ParallelInfo;

static PF_Err ProcessRows(void *refconPV, A_long thread_indexL,
                           A_long i, A_long iterationsL)
{
    ParallelInfo *info = (ParallelInfo *)refconPV;

    // Divide rows among threads
    A_long start_row = (info->height * i) / iterationsL;
    A_long end_row   = (info->height * (i + 1)) / iterationsL;

    for (A_long y = start_row; y < end_row; y++) {
        PF_Pixel *srcRow = (PF_Pixel *)((char *)info->src->data +
                                         y * info->src->rowbytes);
        PF_Pixel *dstRow = (PF_Pixel *)((char *)info->dst->data +
                                         y * info->dst->rowbytes);
        for (A_long x = 0; x < info->width; x++) {
            // Process pixel at (x, y)
            dstRow[x] = srcRow[x];  // placeholder
        }
    }
    return PF_Err_NONE;
}

// Usage:
ParallelInfo info = { src, dst, width, height, 1.5 };
err = iterateSuite->iterate_generic(
    PF_Iterations_ONCE_PER_PROCESSOR,
    &info,
    ProcessRows);
```

---

## Iterate8Suite1 vs Iterate8Suite2 (The 32-Thread Cap)

This is one of the most important distinctions for modern plugin development.

### Suite Versions

| Suite | Version Constant | Frozen In | Thread Limit |
|---|---|---|---|
| `PF_Iterate8Suite1` | `kPFIterate8SuiteVersion1` (1) | AE 5.0 | **Capped at 32 threads** |
| `PF_Iterate8Suite2` | `kPFIterate8SuiteVersion2` (2) | AE 22.0 | **No thread cap** |
| `PF_iterate16Suite1` | `kPFIterate16SuiteVersion1` (1) | AE 5.0 | **Capped at 32 threads** |
| `PF_iterate16Suite2` | `kPFIterate16SuiteVersion2` (2) | AE 22.0 | **No thread cap** |
| `PF_iterateFloatSuite1` | `kPFIterateFloatSuiteVersion1` (1) | AE 7.0 | **Capped at 32 threads** |
| `PF_iterateFloatSuite2` | `kPFIterateFloatSuiteVersion2` (2) | AE 22.0 | **No thread cap** |

### Why This Matters

After Effects 22.0 introduced Multi-Frame Rendering (MFR), which can utilize far more than 32 threads. If you acquire Suite version 1, the host limits your iterate calls to 32 concurrent threads even on machines with 64+ cores. Suite version 2 removes this limitation entirely.

### Suite String Names

```c
#define kPFIterate8Suite       "PF Iterate8 Suite"
#define kPFIterate16Suite      "PF iterate16 Suite"      // note lowercase 'i'
#define kPFIterateFloatSuite   "PF iterateFloat Suite"   // note lowercase 'i'
```

> **PITFALL**: Notice the inconsistent capitalization in the suite name strings. `Iterate8` has a capital I, but `iterate16` and `iterateFloat` have lowercase. These strings must match exactly.

### Acquiring Suite v2 With Fallback

```c
PF_Iterate8Suite2 *iterateSuite2 = NULL;
PF_Iterate8Suite1 *iterateSuite1 = NULL;

PF_Err err = in_data->pica_basicP->AcquireSuite(
    kPFIterate8Suite,
    kPFIterate8SuiteVersion2,
    (const void **)&iterateSuite2);

if (err || !iterateSuite2) {
    // Fall back to v1 on older AE versions
    err = in_data->pica_basicP->AcquireSuite(
        kPFIterate8Suite,
        kPFIterate8SuiteVersion1,
        (const void **)&iterateSuite1);
}
```

Or use the `AEFX_SuiteScoper` template from `AEFX_SuiteHandlerTemplate.h` which handles acquisition and release automatically:

```c
AEFX_SuiteScoper<PF_Iterate8Suite2> iterateSuite(
    in_data,
    kPFIterate8Suite,
    kPFIterate8SuiteVersion2,
    out_data);
```

---

## NO_MAX_THREADS Variants via get_callback_addr

Before Suite version 2 existed, there was another way to bypass the 32-thread cap: the `NO_MAX_THREADS` callback selectors accessible through `get_callback_addr`. These are still available and may be useful if you need to work through the `PF_UtilCallbacks` function pointer table rather than suites.

### Available Selectors

```c
PF_Callback_ITERATE_GENERIC_NO_MAX_THREADS,
PF_Callback_ITERATE_NO_MAX_THREADS,
PF_Callback_ITERATE_ORIGIN_NO_MAX_THREADS,
PF_Callback_ITERATE_ORIGIN_NON_CLIP_SRC_NO_MAX_THREADS,
PF_Callback_ITERATE16_NO_MAX_THREADS,
PF_Callback_ITERATE_ORIGIN16_NO_MAX_THREADS,
PF_Callback_ITERATE_ORIGIN_NON_CLIP_SRC16_NO_MAX_THREADS,
```

### Usage

```c
PF_CallbackFunc fn = NULL;
err = in_data->utils->get_callback_addr(
    in_data->effect_ref,
    in_data->quality,
    PF_MF_Alpha_STRAIGHT,
    PF_Callback_ITERATE_NO_MAX_THREADS,
    &fn);

if (!err && fn) {
    // Cast to the actual iterate function signature and call it
    typedef PF_Err (*IterateFunc)(
        PF_InData*, A_long, A_long, PF_EffectWorld*,
        const PF_Rect*, void*, PF_IteratePixel8Func, PF_EffectWorld*);

    IterateFunc iterate_no_cap = (IterateFunc)fn;
    err = iterate_no_cap(in_data, 0, 0, src, area, refcon, pixFn, dst);
}
```

> **Recommendation**: Prefer Suite version 2 over the `get_callback_addr` approach. It is cleaner, type-safe, and the officially supported path going forward.

---

## The refcon / void* Pattern

All iterate pixel callbacks receive a `void* refcon` that you provide. This is the standard mechanism for passing data into your per-pixel function.

### Thread Safety Rules

Since After Effects may distribute iteration across multiple threads:

1. **Read-only data is safe** -- constants, precomputed LUTs, source world pointers.
2. **Per-pixel output is safe** -- writing to `outP` is fine because each thread processes different pixels.
3. **Shared mutable state is NOT safe** -- accumulators, histograms, or any shared write target requires synchronization (mutex, atomic operations).

### Pattern

```c
typedef struct {
    // Read-only parameters -- safe across threads
    PF_FpLong       intensity;
    PF_FpLong       threshold;
    const A_u_char  *lut;

    // Source world pointer -- safe for reading
    PF_EffectWorld  *src;

    // DO NOT put unsynchronized mutable state here
    // Bad: A_long pixel_count;  // race condition!
} MyEffectInfo;
```

---

## 16-Bit and Float Iterate Suites

The iterate family is replicated for 16-bit and 32-bit float pixel formats.

### 16-Bit

```c
// Suite names
#define kPFIterate16Suite  "PF iterate16 Suite"

// Pixel function signature
PF_Err (*pix_fn)(void* refcon, A_long x, A_long y,
                 PF_Pixel16 *in, PF_Pixel16 *out);

// Macro
PF_ITERATE16(PROG_BASE, PROG_FINAL, SRC, RCT_P, REFCON, PIX_FUNC, DST)
PF_ITERATE_ORIGIN16(PROG_BASE, PROG_FINAL, SRC, RCT_P, OR, REFCON, PIX_FUNC, DST)
```

### 32-Bit Float

```c
// Suite name
#define kPFIterateFloatSuite  "PF iterateFloat Suite"

// Pixel function signature (uses typedef)
typedef PF_Err (*PF_IteratePixelFloatFunc)(
    void* refconP, A_long xL, A_long yL,
    PF_PixelFloat *inP, PF_PixelFloat *outP);
```

There is **no macro** for float iterate in `AE_EffectCB.h`. You must acquire the suite directly.

> **Note**: `iterate_lut` and `iterate_generic` do NOT have 16-bit or float-specific variants. `iterate_lut` is 8-bit only. `iterate_generic` is pixel-format-agnostic (it operates on integer indices, not pixel data).

---

## Decision Tree: Which Iterate To Use

```
Start
  |
  +-- Do you need per-pixel processing?
  |     |
  |     NO --> Is it a per-channel 8-bit LUT?
  |     |       |
  |     |       YES --> PF_ITERATE_LUT
  |     |       |
  |     |       NO --> iterate_generic (parallel work distribution)
  |     |
  |     YES --> Does src and dst have the same coordinate system?
  |               |
  |               YES --> PF_ITERATE / PF_ITERATE16
  |               |
  |               NO --> Does your effect need to read outside source bounds?
  |                       |
  |                       NO --> PF_ITERATE_ORIGIN / PF_ITERATE_ORIGIN16
  |                       |
  |                       YES --> PF_ITERATE_ORIGIN_NON_SRC_CLIP
  |                               (blur, displacement, convolution, etc.)
  |
  +-- Are you on AE 22.0+ and need >32 threads?
        |
        YES --> Use Suite version 2 (Iterate8Suite2, etc.)
        |
        NO --> Suite version 1 is fine
```

---

## Common Pitfalls

### 1. Using iterate instead of iterate_origin_non_clip_src for blurs

If your blur effect expands the output buffer but uses `PF_ITERATE_ORIGIN`, out-of-bounds source reads will return clamped edge pixels. The result is bright or colored edges instead of a smooth fade to transparency.

### 2. Forgetting that progress_final of 0 disables progress

Passing `0` for `progress_final` turns off the progress bar entirely. If you want the progress bar, pass the height of the iteration area.

### 3. Writing to shared state from the pixel callback

The host parallelizes iterate calls across threads. Writing to a shared counter or accumulator without synchronization will cause data races. Either use atomics, or accumulate per-thread and merge after iteration.

### 4. Using the PF_ITERATE macro for float worlds

There is no `PF_ITERATE_FLOAT` macro. You must acquire `PF_iterateFloatSuite1` or `PF_iterateFloatSuite2` and call the suite function directly.

### 5. Suite name capitalization

The iterate suite strings have inconsistent capitalization:
- `"PF Iterate8 Suite"` -- capital I
- `"PF iterate16 Suite"` -- lowercase i
- `"PF iterateFloat Suite"` -- lowercase i

Getting this wrong results in `AcquireSuite` silently returning an error.

### 6. Not handling the src == NULL case

When `src` is `NULL`, iterate calls your pixel function with only the `outP` pointer valid. The `inP` is undefined. This is used for procedural generation where you write pixels without reading a source.

---

## Summary Table

| Variant | Macro Available | 8-bit | 16-bit | Float | Origin | Non-Clip | Suite v2 |
|---|---|---|---|---|---|---|---|
| `iterate` | Yes | Yes | Yes | Yes (suite only) | No | No | Yes |
| `iterate_origin` | Yes | Yes | Yes | Yes (suite only) | Yes | No | Yes |
| `iterate_origin_non_clip_src` | Yes (8-bit) | Yes | Yes | Yes (suite only) | Yes | Yes | Yes |
| `iterate_lut` | Yes | Yes | No | No | No | No | Yes |
| `iterate_generic` | No | N/A | N/A | N/A | N/A | N/A | Yes |
