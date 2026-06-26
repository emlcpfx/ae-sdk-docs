# Color Space Conversion: RGB, HLS, YIQ Callbacks and Suites

> How to convert between RGB, HLS, and YIQ color spaces at 8-bit, 16-bit, and 32-bit float, and the critical range differences between them.

## Overview

The AE SDK provides color space conversion in three tiers, one per bit depth. Each tier has different output ranges for the same conceptual values. Getting these ranges wrong is the single most common bug when porting an 8-bit color effect to higher bit depths.

| Tier | API | Pixel type | Available since |
|---|---|---|---|
| 8-bit callbacks | Macros (`PF_RGB_TO_HLS`, etc.) or `PF_ColorCallbacksSuite1` | `PF_Pixel` (8-bit) | AE 1.0 / Suite frozen AE 5.0 |
| 16-bit suite | `PF_ColorCallbacks16Suite1` | `PF_Pixel16` | Frozen AE 5.0 |
| Float suite | `PF_ColorCallbacksFloatSuite1` | `PF_PixelFloat` | Frozen AE 7.0 |

All three tiers provide the same set of operations:

- `RGBtoHLS` / `HLStoRGB` -- full color space conversion
- `RGBtoYIQ` / `YIQtoRGB` -- full color space conversion
- `Luminance` -- single-value luminance extraction
- `Hue` -- single-value hue extraction
- `Lightness` -- single-value lightness extraction
- `Saturation` -- single-value saturation extraction

## Intermediate Types

HLS and YIQ values are stored in fixed-point arrays, regardless of the input pixel bit depth:

```cpp
typedef PF_Fixed  PF_HLS_Pixel[3];   // array of 3 PF_Fixed values
typedef PF_Fixed  PF_YIQ_Pixel[3];   // array of 3 PF_Fixed values
```

`PF_Fixed` is a 16.16 fixed-point signed 32-bit integer (see the fixed-point documentation for details). The individual component extraction functions (Luminance, Hue, etc.) return `A_long` for 8-bit and 16-bit, and `float` for the float suite.

## 8-bit Callbacks

### Macros (from AE_EffectCB.h)

These macros go through the `in_data->utils->colorCB` callback block. They require `in_data` to be in scope.

```cpp
// Full conversions
PF_RGB_TO_HLS(rgb, hls)     // PF_Pixel* -> PF_HLS_Pixel
PF_HLS_TO_RGB(hls, rgb)     // PF_HLS_Pixel -> PF_Pixel*
PF_RGB_TO_YIQ(rgb, yiq)     // PF_Pixel* -> PF_YIQ_Pixel
PF_YIQ_TO_RGB(yiq, rgb)     // PF_YIQ_Pixel -> PF_Pixel*

// Individual component extraction
PF_LUMINANCE(rgb, lum100)   // PF_Pixel* -> A_long* (100 * luminance)
PF_HUE(rgb, hue)            // PF_Pixel* -> A_long* (0-255, maps to 0-360 degrees)
PF_LIGHTNESS(rgb, light)    // PF_Pixel* -> A_long* (0-255)
PF_SATURATION(rgb, sat)     // PF_Pixel* -> A_long* (0-255)
```

### Macro definitions

```cpp
#define PF_RGB_TO_HLS(RGB, HLS) \
    (*in_data->utils->colorCB.RGBtoHLS)(in_data->effect_ref, (RGB), (HLS))

#define PF_HLS_TO_RGB(HLS, RGB) \
    (*in_data->utils->colorCB.HLStoRGB)(in_data->effect_ref, (HLS), (RGB))

#define PF_LUMINANCE(RGB, LUM100) \
    (*in_data->utils->colorCB.Luminance)(in_data->effect_ref, (RGB), (LUM100))

// ... and so on for each function
```

### Suite form (PF_ColorCallbacksSuite1)

Suite name: `"PF Color Suite"`, version 1. Frozen in AE 5.0.

```cpp
typedef struct PF_ColorCallbacksSuite1 {
    PF_Err (*RGBtoHLS)(PF_ProgPtr effect_ref, PF_Pixel *rgb, PF_HLS_Pixel hls);
    PF_Err (*HLStoRGB)(PF_ProgPtr effect_ref, PF_HLS_Pixel hls, PF_Pixel *rgb);
    PF_Err (*RGBtoYIQ)(PF_ProgPtr effect_ref, PF_Pixel *rgb, PF_YIQ_Pixel yiq);
    PF_Err (*YIQtoRGB)(PF_ProgPtr effect_ref, PF_HLS_Pixel yiq, PF_Pixel *rgb);
    PF_Err (*Luminance)(PF_ProgPtr effect_ref, PF_Pixel *rgb, A_long *lum100);
    PF_Err (*Hue)(PF_ProgPtr effect_ref, PF_Pixel *rgb, A_long *hue);
    PF_Err (*Lightness)(PF_ProgPtr effect_ref, PF_Pixel *rgb, A_long *lightness);
    PF_Err (*Saturation)(PF_ProgPtr effect_ref, PF_Pixel *rgb, A_long *saturation);
} PF_ColorCallbacksSuite1;
```

### 8-bit example

```cpp
static PF_Err AdjustHue8(
    PF_InData  *in_data,
    PF_Pixel   *inP,
    PF_Pixel   *outP,
    A_long      hue_shift)   // 0-255 range shift
{
    PF_Err err = PF_Err_NONE;
    PF_HLS_Pixel hls;

    // Convert RGB to HLS
    ERR(PF_RGB_TO_HLS(inP, hls));

    if (!err) {
        // hls[0] = Hue (PF_Fixed, but 8-bit callbacks use 0-255 mapped to 0-360)
        // hls[1] = Lightness
        // hls[2] = Saturation
        // Shift hue -- wraps around via fixed-point overflow
        hls[0] += INT2FIX(hue_shift);

        // Convert back
        ERR(PF_HLS_TO_RGB(hls, outP));
    }

    return err;
}
```

### 8-bit individual component example

```cpp
PF_Pixel pixel = {255, 200, 100, 50};  // A, R, G, B
A_long lum100, hue, lightness, saturation;

PF_LUMINANCE(&pixel, &lum100);
// lum100 = 100 * luminance value (e.g., 12345 means luminance of 123.45)

PF_HUE(&pixel, &hue);
// hue = 0 to 255, mapping to 0 to 360 degrees
// Undefined hue (achromatic) returns PF_HUE_UNDEFINED (0x80000000)

PF_LIGHTNESS(&pixel, &lightness);
// lightness = 0 to 255

PF_SATURATION(&pixel, &saturation);
// saturation = 0 to 255
```

## 16-bit Suite (PF_ColorCallbacks16Suite1)

Suite name: `"PF Color16 Suite"`, version 1. Frozen in AE 5.0.

```cpp
typedef struct PF_ColorCallbacks16Suite1 {
    PF_Err (*RGBtoHLS)(PF_ProgPtr effect_ref, PF_Pixel16 *rgb, PF_HLS_Pixel hls);
    PF_Err (*HLStoRGB)(PF_ProgPtr effect_ref, PF_HLS_Pixel hls, PF_Pixel16 *rgb);
    PF_Err (*RGBtoYIQ)(PF_ProgPtr effect_ref, PF_Pixel16 *rgb, PF_YIQ_Pixel yiq);
    PF_Err (*YIQtoRGB)(PF_ProgPtr effect_ref, PF_HLS_Pixel yiq, PF_Pixel16 *rgb);
    PF_Err (*Luminance)(PF_ProgPtr effect_ref, PF_Pixel16 *rgb, A_long *lum100);
    PF_Err (*Hue)(PF_ProgPtr effect_ref, PF_Pixel16 *rgb, A_long *hue);
    PF_Err (*Lightness)(PF_ProgPtr effect_ref, PF_Pixel16 *rgb, A_long *lightness);
    PF_Err (*Saturation)(PF_ProgPtr effect_ref, PF_Pixel16 *rgb, A_long *saturation);
} PF_ColorCallbacks16Suite1;
```

### 16-bit range differences

The critical differences from 8-bit, directly from the SDK header comments:

- **Luminance**: Returns `100 * luminance` (same convention as 8-bit)
- **Hue**: Returns `0-255` mapped to `0-360 degrees` (same convention as 8-bit)
- **Lightness**: Returns `0-32768` (NOT 0-255!)
- **Saturation**: Returns `0-32768` (NOT 0-255!)

### 16-bit example

```cpp
static PF_Err GetBrightness16(
    PF_InData    *in_data,
    PF_Pixel16   *pixP,
    PF_FpLong    *brightnessP)   // normalized 0.0-1.0
{
    PF_Err err = PF_Err_NONE;

    AEFX_SuiteScoper<PF_ColorCallbacks16Suite1> cs16(
        in_data, kPFColorCallbacks16Suite, kPFColorCallbacks16SuiteVersion1, "");

    A_long lightness;
    ERR(cs16->Lightness(in_data->effect_ref, pixP, &lightness));

    if (!err) {
        // 16-bit lightness range is 0-32768, NOT 0-255
        *brightnessP = (PF_FpLong)lightness / 32768.0;
    }

    return err;
}
```

## Float Suite (PF_ColorCallbacksFloatSuite1)

Suite name: `"PF ColorFloat Suite"`, version 1. Frozen in AE 7.0.

```cpp
typedef struct PF_ColorCallbacksFloatSuite1 {
    PF_Err (*RGBtoHLS)(PF_ProgPtr effect_ref, PF_PixelFloat *rgb, PF_HLS_Pixel hls);
    PF_Err (*HLStoRGB)(PF_ProgPtr effect_ref, PF_HLS_Pixel hls, PF_PixelFloat *rgb);
    PF_Err (*RGBtoYIQ)(PF_ProgPtr effect_ref, PF_PixelFloat *rgb, PF_YIQ_Pixel yiq);
    PF_Err (*YIQtoRGB)(PF_ProgPtr effect_ref, PF_HLS_Pixel yiq, PF_PixelFloat *rgb);
    PF_Err (*Luminance)(PF_ProgPtr effect_ref, PF_PixelFloat *rgb, float *lumP);
    PF_Err (*Hue)(PF_ProgPtr effect_ref, PF_PixelFloat *rgb, float *hue);
    PF_Err (*Lightness)(PF_ProgPtr effect_ref, PF_PixelFloat *rgb, float *lightness);
    PF_Err (*Saturation)(PF_ProgPtr effect_ref, PF_PixelFloat *rgb, float *saturation);
} PF_ColorCallbacksFloatSuite1;
```

### Float range differences

The float suite uses completely different output types and ranges:

- **Luminance**: Returns raw `float` luminance (NOT multiplied by 100)
- **Hue**: Returns `0.0-360.0` float degrees (NOT 0-255)
- **Lightness**: Returns raw `float` (0.0-1.0 range, can exceed for HDR)
- **Saturation**: Returns raw `float` (0.0-1.0 range)

Note that the output type changes from `A_long*` to `float*` for all individual component functions.

### Float example

```cpp
static PF_Err DesaturateFloat(
    PF_InData     *in_data,
    PF_PixelFloat *inP,
    PF_PixelFloat *outP,
    float          amount)   // 0.0 = original, 1.0 = fully desaturated
{
    PF_Err err = PF_Err_NONE;

    AEFX_SuiteScoper<PF_ColorCallbacksFloatSuite1> csf(
        in_data, kPFColorCallbacksFloatSuite,
        kPFColorCallbacksFloatSuiteVersion1, "");

    PF_HLS_Pixel hls;
    ERR(csf->RGBtoHLS(in_data->effect_ref, inP, hls));

    if (!err) {
        // hls values are PF_Fixed (16.16)
        // Reduce saturation component
        PF_FpLong sat = FIX_2_FLOAT(hls[2]);
        sat *= (1.0 - amount);
        hls[2] = FLOAT2FIX(sat);

        ERR(csf->HLStoRGB(in_data->effect_ref, hls, outP));
    }

    return err;
}
```

## Range Differences Table

This is the most important reference in this document. Print it. Tape it to your monitor.

### Individual Component Functions

| Function | 8-bit output | 16-bit output | Float output |
|---|---|---|---|
| **Luminance** | `A_long`: 100 * luminance | `A_long`: 100 * luminance | `float`: raw luminance (NOT * 100) |
| **Hue** | `A_long`: 0-255 (maps to 0-360) | `A_long`: 0-255 (maps to 0-360) | `float`: 0.0-360.0 degrees |
| **Lightness** | `A_long`: 0-255 | `A_long`: 0-32768 | `float`: 0.0-1.0 |
| **Saturation** | `A_long`: 0-255 | `A_long`: 0-32768 | `float`: 0.0-1.0 |

### Normalizing to 0.0-1.0

To write bit-depth-independent logic, normalize all values to floating point:

```cpp
// Luminance normalization
double lum_8bit   = (double)lum100 / (100.0 * 255.0);   // lum100 is 100*lum, max pixel is 255
double lum_16bit  = (double)lum100 / (100.0 * 32768.0); // lum100 is 100*lum, max pixel is 32768
double lum_float  = (double)lumF;                         // already normalized

// Hue normalization (to 0-360)
double hue_8bit   = (double)hue * 360.0 / 255.0;
double hue_16bit  = (double)hue * 360.0 / 255.0;  // same range as 8-bit!
double hue_float  = (double)hueF;                   // already in degrees

// Lightness normalization (to 0.0-1.0)
double light_8bit  = (double)lightness / 255.0;
double light_16bit = (double)lightness / 32768.0;   // different range!
double light_float = (double)lightnessF;             // already normalized

// Saturation normalization (to 0.0-1.0)
double sat_8bit  = (double)saturation / 255.0;
double sat_16bit = (double)saturation / 32768.0;     // different range!
double sat_float = (double)saturationF;              // already normalized
```

### HLS/YIQ Full Conversion

The `PF_HLS_Pixel` and `PF_YIQ_Pixel` arrays always use `PF_Fixed` (16.16 fixed-point), regardless of which suite you use. The array layout is:

| Index | HLS meaning | YIQ meaning |
|---|---|---|
| `[0]` | Hue | Y (luminance) |
| `[1]` | Lightness | I (in-phase) |
| `[2]` | Saturation | Q (quadrature) |

Use `FIX_2_FLOAT()` and `FLOAT2FIX()` to convert these values.

## Hue Undefined Value

When a pixel is achromatic (zero saturation -- a pure gray), the hue is undefined. The SDK uses a sentinel value:

```cpp
#define PF_HUE_UNDEFINED  0x80000000
```

Always check for this before using hue values:

```cpp
A_long hue;
PF_HUE(&pixel, &hue);

if (hue == PF_HUE_UNDEFINED) {
    // Pixel is achromatic -- hue is meaningless
} else {
    // Safe to use hue value
}
```

This sentinel applies to 8-bit and 16-bit. For the float suite, check the header comments -- an achromatic pixel may return 0.0 for hue.

## Complete Multi-Depth Example

A function that computes a luminance threshold at any bit depth:

```cpp
static PF_Err LuminanceThreshold(
    PF_InData      *in_data,
    PF_EffectWorld *input,
    PF_EffectWorld *output,
    PF_FpLong       threshold)  // 0.0-1.0 normalized
{
    PF_Err err = PF_Err_NONE;

    // Determine format
    AEFX_SuiteScoper<PF_WorldSuite2> ws2(
        in_data, kPFWorldSuite, kPFWorldSuiteVersion2, "");

    PF_PixelFormat format;
    ERR(ws2->PF_GetPixelFormat(input, &format));

    if (!err) {
        switch (format) {
            case PF_PixelFormat_ARGB32: {
                // 8-bit: Luminance returns 100 * lum
                // For a pixel with max channels (255), max lum100 ~ 25500
                A_long thresh100 = (A_long)(threshold * 255.0 * 100.0);

                // Use PF_Iterate8Suite with per-pixel callback that calls:
                //   PF_LUMINANCE(inP, &lum100);
                //   if (lum100 >= thresh100) { white } else { black }
                break;
            }
            case PF_PixelFormat_ARGB64: {
                // 16-bit: Luminance returns 100 * lum
                // For a pixel with max channels (32768), max lum100 ~ 3276800
                A_long thresh100 = (A_long)(threshold * 32768.0 * 100.0);

                // Use PF_Iterate16Suite with per-pixel callback that calls:
                //   cs16->Luminance(ref, inP, &lum100);
                //   if (lum100 >= thresh100) { white } else { black }
                break;
            }
            case PF_PixelFormat_ARGB128: {
                // Float: Luminance returns raw float, NOT multiplied by 100
                float threshF = (float)threshold;

                // Use PF_IterateFloatSuite with per-pixel callback that calls:
                //   csf->Luminance(ref, inP, &lumF);
                //   if (lumF >= threshF) { white } else { black }
                break;
            }
        }
    }

    return err;
}
```

## Common Pitfalls

### 1. Using 8-bit range assumptions with 16-bit callbacks

This is the most common bug when adding 16-bit support:

```cpp
// WRONG -- assumes lightness range is 0-255 at 16-bit
A_long lightness;
cs16->Lightness(ref, pix16, &lightness);
double normalized = (double)lightness / 255.0;  // Values above 255 give > 1.0!

// CORRECT -- 16-bit lightness range is 0-32768
double normalized = (double)lightness / 32768.0;
```

### 2. Forgetting the 100x multiplier on Luminance (8-bit and 16-bit)

```cpp
// WRONG -- treats lum100 as direct luminance
A_long lum100;
PF_LUMINANCE(&pixel, &lum100);
A_u_char gray = (A_u_char)lum100;  // Way too large!

// CORRECT -- divide by 100
A_u_char gray = (A_u_char)(lum100 / 100);
```

### 3. Assuming float Luminance is also multiplied by 100

```cpp
// WRONG -- float luminance is NOT multiplied by 100
float lumF;
csf->Luminance(ref, pixFloat, &lumF);
float normalized = lumF / 100.0f;  // Far too small!

// CORRECT -- float luminance is the raw value
float normalized = lumF;  // Already in natural range
```

The SDK header comment is explicit: `/* << luminance -- note *not* 100*lum */`

### 4. Assuming 16-bit hue range differs from 8-bit

Hue range is `0-255` for **both** 8-bit and 16-bit. It does NOT scale to 0-32768 like lightness and saturation do. Only lightness and saturation change range at 16-bit.

```cpp
// At both 8-bit and 16-bit:
A_long hue;
// hue is 0-255, mapping to 0-360 degrees
double degrees = (double)hue * 360.0 / 255.0;

// Only at float:
float hueF;
// hueF is already 0.0-360.0 degrees
double degrees = (double)hueF;
```

### 5. Passing wrong pixel type to wrong suite

The compiler may not catch this if you use casts or if the types happen to have the same size:

```cpp
// WRONG -- passing PF_Pixel16 to 8-bit suite
PF_Pixel16 pix16 = ...;
PF_RGB_TO_HLS((PF_Pixel*)&pix16, hls);  // Reads wrong data!

// CORRECT -- use the 16-bit suite for PF_Pixel16
cs16->RGBtoHLS(ref, &pix16, hls);
```

### 6. Not acquiring the suite for 16-bit or float

The 8-bit color callbacks are available through macros (via `in_data->utils->colorCB`). But 16-bit and float conversions are **only** available through their respective suites. There are no macros for them.

```cpp
// 8-bit: can use macro (no suite needed)
PF_RGB_TO_HLS(rgb8, hls);

// 16-bit: MUST acquire suite
AEFX_SuiteScoper<PF_ColorCallbacks16Suite1> cs16(
    in_data, kPFColorCallbacks16Suite, kPFColorCallbacks16SuiteVersion1, "");
cs16->RGBtoHLS(in_data->effect_ref, rgb16, hls);

// Float: MUST acquire suite
AEFX_SuiteScoper<PF_ColorCallbacksFloatSuite1> csf(
    in_data, kPFColorCallbacksFloatSuite, kPFColorCallbacksFloatSuiteVersion1, "");
csf->RGBtoHLS(in_data->effect_ref, rgbFloat, hls);
```

## Suite Name Constants

From `AE_EffectCBSuites.h`:

```cpp
#define kPFColorCallbacksSuite           "PF Color Suite"
#define kPFColorCallbacksSuiteVersion1   1    // frozen in AE 5.0

#define kPFColorCallbacks16Suite         "PF Color16 Suite"
#define kPFColorCallbacks16SuiteVersion1 1    // frozen in AE 5.0

#define kPFColorCallbacksFloatSuite              "PF ColorFloat Suite"
#define kPFColorCallbacksFloatSuiteVersion1      1    // frozen in AE 7.0
```

## SDK Header References

- **`AE_EffectCBSuites.h`** -- `PF_ColorCallbacksSuite1`, `PF_ColorCallbacks16Suite1`, `PF_ColorCallbacksFloatSuite1` suite definitions with range comments
- **`AE_EffectCB.h`** -- `PF_ColorCallbacks` struct (8-bit callback block), all 8-bit macros (`PF_RGB_TO_HLS`, `PF_LUMINANCE`, etc.)
- **`AE_Effect.h`** -- `PF_HLS_Pixel`, `PF_YIQ_Pixel`, `PF_RGB_Pixel` type definitions, `PF_HUE_UNDEFINED` constant, `PF_Fixed` type
