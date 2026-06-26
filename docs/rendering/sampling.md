# Pixel Sampling: subpixel_sample, area_sample, and PF_SampPB

After Effects provides built-in callbacks for sampling pixels at non-integer coordinates. These are essential for distortion effects, transforms, and any effect that reads from arbitrary source positions. This document covers the sampling parameter block, the sampling lifecycle, all sampling variants, and practical code examples.

---

## Overview

There are two fundamental sampling operations:

- **subpixel_sample** -- Interpolates a single pixel value at a non-integer (x, y) position. Uses bilinear interpolation at high quality, nearest-neighbor at low quality.
- **area_sample** -- Computes the weighted average of an axis-aligned rectangular area. Used when a source pixel maps to a region larger than one pixel (downsampling/minification).

Both operate through `PF_SampPB`, a parameter block that describes the source world and sampling behavior.

---

## PF_SampPB Structure

Defined in `AE_EffectCB.h`:

```cpp
typedef struct {
    // Required for all sampling
    PF_Fixed            x_radius;       // radius for area_sample, 0 for point sample
    PF_Fixed            y_radius;
    PF_Fixed            area;           // must fit in a PF_Fixed; must be correct
    PF_EffectWorld      *src;           // the world to sample from
    PF_SampleEdgeBehav  samp_behave;    // edge behavior
    A_long              allow_asynch;   // OK if result not available until end_sampling

    // For batch sampling & compositing (advanced)
    A_long              motion_blur;
    PF_CompositeMode    comp_mode;
    PF_PixelPtr         mask0;          // per-pixel masking before xfer mode
    A_u_char            *fcm_table;
    A_u_char            *fcd_table;
    A_long              reserved[8];    // set to zero at begin_sampling
} PF_SampPB;
```

### Fields You Must Set

| Field | For subpixel_sample | For area_sample |
|---|---|---|
| `src` | Required -- the source world | Required -- the source world |
| `x_radius` | Set to 0 | Set to half-width of sample area (PF_Fixed) |
| `y_radius` | Set to 0 | Set to half-height of sample area (PF_Fixed) |
| `area` | Ignored (set to 0) | `x_radius * y_radius` (must be correct, must fit in PF_Fixed) |
| `samp_behave` | Usually `PF_SampleEdgeBehav_ZERO` | Usually `PF_SampleEdgeBehav_ZERO` |
| `allow_asynch` | Usually 0 (synchronous) | Usually 0 |

### Edge Behavior

```cpp
enum {
    PF_SampleEdgeBehav_ZERO = 0L   // Pixels outside image treated as alpha=0 (transparent black)
};
```

Only `PF_SampleEdgeBehav_ZERO` is supported. `REPEAT` and `WRAP` are defined in comments but not implemented.

---

## The Sampling Lifecycle

### 1. begin_sampling

Call before performing any sampling. This allows AE to set up internal acceleration structures (scanline index tables, etc.).

```cpp
PF_Err err = PF_Err_NONE;
PF_SampPB samp_pb;
AEFX_CLR_STRUCT(samp_pb);

samp_pb.src = &input_world;
samp_pb.samp_behave = PF_SampleEdgeBehav_ZERO;

ERR(PF_BEGIN_SAMPLING(in_data->quality, &samp_pb));
```

The macro:
```cpp
#define PF_BEGIN_SAMPLING(QUALITY, PARAMS) \
    (*in_data->utils->begin_sampling)( \
        in_data->effect_ref, (QUALITY), PF_MF_Alpha_STRAIGHT, (PARAMS))
```

### 2. Sample (in your pixel loop)

Call `subpixel_sample` or `area_sample` for each output pixel.

### 3. end_sampling

Call when done sampling. Frees any internal resources allocated by begin_sampling.

```cpp
ERR(PF_END_SAMPLING(in_data->quality, &samp_pb));
```

> **PITFALL:** Always call end_sampling, even if sampling encountered errors. Use ERR2 for this.

---

## subpixel_sample

Samples a single interpolated pixel at a non-integer position.

### Coordinates Are PF_Fixed, NOT Integer Pixels

Coordinates are 16.16 fixed-point numbers. Use `INT2FIX()` to convert integer pixel coordinates, or `FLOAT2FIX()` for floating-point coordinates.

```cpp
// Sample at pixel (10.5, 20.75)
PF_Fixed fx = FLOAT2FIX(10.5);
PF_Fixed fy = FLOAT2FIX(20.75);

PF_Pixel sample;
ERR(PF_SUBPIXEL_SAMPLE(fx, fy, &samp_pb, &sample));
```

### The Macro

```cpp
#define PF_SUBPIXEL_SAMPLE(X, Y, PARAMS, DST_PXL) \
    (*in_data->utils->subpixel_sample)( \
        in_data->effect_ref, (X), (Y), (PARAMS), (DST_PXL))
```

### Direct Function Signature

```cpp
PF_Err (*subpixel_sample)(
    PF_ProgPtr      effect_ref,     // in_data->effect_ref
    PF_Fixed        x,              // 16.16 fixed-point X
    PF_Fixed        y,              // 16.16 fixed-point Y
    const PF_SampPB *params,        // sampling parameter block
    PF_Pixel        *dst_pixel);    // output: interpolated 8-bit pixel
```

### Quality Behavior

- **High quality (PF_Quality_HI):** Bilinear interpolation between surrounding pixels
- **Low quality (PF_Quality_LO):** Nearest-neighbor (snaps to closest pixel)

---

## area_sample

Computes the average color of an axis-aligned rectangular area.

### Radius Meaning

`x_radius` and `y_radius` define half the width and half the height of the sampling rectangle, in PF_Fixed units. The sampled area is centered at (x, y) and extends `x_radius` in each horizontal direction and `y_radius` in each vertical direction.

```
Sample area = (x - x_radius, y - y_radius) to (x + x_radius, y + y_radius)
```

### Area Field

The `area` field must equal `x_radius * y_radius` and must fit in a `PF_Fixed`. Due to overflow constraints, the maximum sample area is approximately 256 x 256 pixels (radius < 128 pixels in each direction).

### Usage

```cpp
PF_SampPB area_pb;
AEFX_CLR_STRUCT(area_pb);
area_pb.src = &input_world;
area_pb.samp_behave = PF_SampleEdgeBehav_ZERO;

// Sample a 4x4 pixel area centered at (50, 50)
area_pb.x_radius = INT2FIX(2);   // half-width = 2 pixels
area_pb.y_radius = INT2FIX(2);   // half-height = 2 pixels
area_pb.area = INT2FIX(2) * INT2FIX(2);  // Careful: this overflows for large radii

PF_Pixel sample;
ERR(PF_AREA_SAMPLE(INT2FIX(50), INT2FIX(50), &area_pb, &sample));
```

> **PITFALL:** Multiplying two PF_Fixed values with INT2FIX multiplication gives incorrect results because PF_Fixed multiplication requires shifting. For the `area` field, compute it as: `area_pb.area = (A_long)((double)x_radius * y_radius / 65536.0);` or use fixed-point multiply properly.

### The Macro

```cpp
#define PF_AREA_SAMPLE(X, Y, PARAMS, DST_PXL) \
    (*in_data->utils->area_sample)( \
        in_data->effect_ref, (X), (Y), (PARAMS), (DST_PXL))
```

---

## 16-bit Sampling Variants

For 16-bit worlds, use the 16-bit callbacks available directly on `PF_UtilCallbacks`:

```cpp
PF_Pixel16 sample16;

// Via direct callback (no convenience macro in SDK)
ERR((*in_data->utils->subpixel_sample16)(
    in_data->effect_ref, fx, fy, &samp_pb, &sample16));

ERR((*in_data->utils->area_sample16)(
    in_data->effect_ref, fx, fy, &samp_pb, &sample16));
```

Or use the `PF_Sampling16Suite1`:

```cpp
AEFX_SuiteScoper<PF_Sampling16Suite1> samp16(
    in_data, kPFSampling16Suite, kPFSampling16SuiteVersion1);

PF_Pixel16 sample16;
ERR(samp16->subpixel_sample16(in_data->effect_ref, fx, fy, &samp_pb, &sample16));
ERR(samp16->area_sample16(in_data->effect_ref, fx, fy, &samp_pb, &sample16));
```

The suite also provides `nn_sample16` for explicit nearest-neighbor sampling regardless of quality setting.

---

## Float (32-bit) Sampling Variants

For 32-bit float worlds, use `PF_SamplingFloatSuite1`:

```cpp
AEFX_SuiteScoper<PF_SamplingFloatSuite1> sampF(
    in_data, kPFSamplingFloatSuite, kPFSamplingFloatSuiteVersion1);

PF_PixelFloat sampleF;
ERR(sampF->subpixel_sample_float(in_data->effect_ref, fx, fy, &samp_pb, &sampleF));
ERR(sampF->area_sample_float(in_data->effect_ref, fx, fy, &samp_pb, &sampleF));
```

Suite functions:
- `nn_sample_float` -- nearest-neighbor
- `subpixel_sample_float` -- bilinear interpolation
- `area_sample_float` -- area average

---

## Sampling Suite Reference

| Suite | Name Constant | Version | Bit Depth |
|---|---|---|---|
| `PF_Sampling8Suite1` | `kPFSampling8Suite` | `kPFSampling8SuiteVersion1` (1) | 8-bit |
| `PF_Sampling16Suite1` | `kPFSampling16Suite` | `kPFSampling16SuiteVersion1` (1) | 16-bit |
| `PF_SamplingFloatSuite1` | `kPFSamplingFloatSuite` | `kPFSamplingFloatSuiteVersion1` (1) | 32-bit float |

All three suites provide the same three functions (nn_sample, subpixel_sample, area_sample) with appropriate pixel types.

---

## Quality Override via get_callback_addr

By default, the sampling functions in `in_data->utils` are automatically set to the current quality level's implementation. If you need high-quality sampling even when AE is rendering at low quality (or vice versa), use `get_callback_addr`:

```cpp
PF_CallbackFunc hi_qual_sample_fn;
ERR((*in_data->utils->get_callback_addr)(
    in_data->effect_ref,
    PF_Quality_HI,              // Force high quality
    PF_MF_Alpha_STRAIGHT,
    PF_Callback_SUBPIXEL_SAMPLE,
    &hi_qual_sample_fn));

// Cast to the correct function pointer type
typedef PF_Err (*SubpixelSampleFunc)(PF_ProgPtr, PF_Fixed, PF_Fixed, const PF_SampPB*, PF_Pixel*);
SubpixelSampleFunc hi_sample = (SubpixelSampleFunc)hi_qual_sample_fn;

PF_Pixel result;
ERR(hi_sample(in_data->effect_ref, fx, fy, &samp_pb, &result));
```

### Available Callback Selectors

```cpp
PF_Callback_BEGIN_SAMPLING
PF_Callback_SUBPIXEL_SAMPLE
PF_Callback_AREA_SAMPLE
PF_Callback_END_SAMPLING
PF_Callback_SUBPIXEL_SAMPLE16
PF_Callback_AREA_SAMPLE16
```

---

## Complete Distortion Effect Example

```cpp
static PF_Err MyDistortFunc(
    void* refcon,
    A_long x,
    A_long y,
    PF_Pixel* inP,
    PF_Pixel* outP)
{
    PF_Err err = PF_Err_NONE;
    MyRefcon* ref = (MyRefcon*)refcon;

    // Compute source position with some distortion
    double src_x = x + ref->amplitude * sin(y * ref->frequency);
    double src_y = y + ref->amplitude * cos(x * ref->frequency);

    // Convert to PF_Fixed
    PF_Fixed fx = FLOAT2FIX(src_x);
    PF_Fixed fy = FLOAT2FIX(src_y);

    // Sample from source
    err = ref->sample_fn(
        ref->effect_ref,
        fx, fy,
        ref->samp_pb,
        outP);

    return err;
}

static PF_Err Render(PF_InData* in_data, PF_OutData* out_data,
                     PF_ParamDef* params[], PF_LayerDef* output)
{
    PF_Err err = PF_Err_NONE;
    PF_Err err2 = PF_Err_NONE;

    PF_EffectWorld* input = &params[0]->u.ld;

    // Set up sampling
    PF_SampPB samp_pb;
    AEFX_CLR_STRUCT(samp_pb);
    samp_pb.src = input;
    samp_pb.samp_behave = PF_SampleEdgeBehav_ZERO;

    ERR(PF_BEGIN_SAMPLING(in_data->quality, &samp_pb));

    // Prepare refcon for iterate callback
    MyRefcon refcon;
    refcon.samp_pb = &samp_pb;
    refcon.effect_ref = in_data->effect_ref;
    refcon.sample_fn = in_data->utils->subpixel_sample;
    refcon.amplitude = FIX_2_FLOAT(params[PARAM_AMPLITUDE]->u.sd.value);
    refcon.frequency = FIX_2_FLOAT(params[PARAM_FREQUENCY]->u.sd.value);

    ERR(PF_ITERATE(0, output->height, input, NULL, &refcon, MyDistortFunc, output));

    // Always end sampling, even on error
    ERR2(PF_END_SAMPLING(in_data->quality, &samp_pb));

    return err;
}
```

---

## Pitfalls

**Coordinates are PF_Fixed, not integers.** Passing raw integer pixel coordinates without INT2FIX will sample from positions near the origin (the integer value is interpreted as a 16.16 fixed-point number, making it a tiny fraction).

**Always call begin_sampling before and end_sampling after.** Skipping begin_sampling may cause crashes or incorrect results. Skipping end_sampling leaks resources.

**area_sample is limited to ~256x256 pixel areas.** The area must fit in a PF_Fixed value. For larger areas, downsample in stages or use a different approach.

**The SampPB.src must remain valid for the entire sampling session.** Do not dispose the source world between begin_sampling and end_sampling.

**For performance-critical inner loops, cache the function pointer.** Dereferencing `in_data->utils->subpixel_sample` on every pixel is slower than caching the function pointer in your refcon.
