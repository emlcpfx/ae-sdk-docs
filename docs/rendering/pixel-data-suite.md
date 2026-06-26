# Pixel Data Access: PF_PixelDataSuite and Typed Pixel Pointers

> How to safely obtain typed pixel pointers (`PF_Pixel8*`, `PF_Pixel16*`, `PF_PixelFloat*`) from a `PF_EffectWorld`, and why the float path is fundamentally different from 8-bit and 16-bit.

## Overview

When After Effects hands your effect a `PF_EffectWorld`, the raw pixel buffer is stored in the `data` field -- but that field is a `PF_PixelPtr`, which is **opaque** when `PF_DEEP_COLOR_AWARE` is defined. You cannot simply cast it to the pixel type you want. Instead, you must use one of the pixel data access mechanisms to obtain a properly typed pointer.

There are **two** access mechanisms:

| Mechanism | 8-bit | 16-bit | 32-bit float | GPU float |
|---|---|---|---|---|
| **Callback macros** (`PF_GET_PIXEL_DATA8`, `PF_GET_PIXEL_DATA16`) | Yes | Yes | **No** | No |
| **PF_PixelDataSuite1 / Suite2** | Yes | Yes | Yes | Suite2 only |

**Critical fact**: There is no `PF_GET_PIXEL_DATA_FLOAT` macro. Float pixel access is *only* available through the suite. This is the single most common compilation error when adding 32-bit support to an existing plugin.

## Pixel Types

From `AE_Effect.h`:

```cpp
// 8-bit: 4 bytes per pixel, channels 0-255
typedef struct {
    A_u_char  alpha, red, green, blue;
} PF_Pixel;       // also known as PF_Pixel8

// 16-bit: 8 bytes per pixel, channels 0-32768 (NOT 0-65535!)
typedef struct {
    A_u_short alpha, red, green, blue;
} PF_Pixel16;

// 32-bit float: 16 bytes per pixel, 1.0 = white, supports overrange
typedef struct {
    PF_FpShort alpha, red, green, blue;  // PF_FpShort is float (32-bit)
} PF_PixelFloat;  // also known as PF_Pixel32
```

Channel order is always **ARGB** for all three types.

## The PF_PixelPtr Problem

When you define `PF_DEEP_COLOR_AWARE` (required for 16-bit and float support), the type of `PF_EffectWorld.data` changes:

```cpp
// Without PF_DEEP_COLOR_AWARE:
typedef PF_Pixel  *PF_PixelPtr;     // direct PF_Pixel8 pointer

// With PF_DEEP_COLOR_AWARE:
typedef PF_PixelOpaquePtr  PF_PixelPtr;  // opaque -- cannot be dereferenced
```

This means `world->data` is opaque and you **cannot** cast it directly. You must ask the host to give you a typed pointer.

## Callback Macros (8-bit and 16-bit Only)

Defined in `AE_EffectCB.h`, these macros go through the utility callback block:

```cpp
#define PF_GET_PIXEL_DATA8(WORLDP, PIXELPTR0, PIXEL8PP) \
    (*in_data->utils->get_pixel_data8)((WORLDP), (PIXELPTR0), (PIXEL8PP))

#define PF_GET_PIXEL_DATA16(WORLDP, PIXELPTR0, PIXEL16PP) \
    (*in_data->utils->get_pixel_data16)((WORLDP), (PIXELPTR0), (PIXEL16PP))
```

### Parameters

| Parameter | Type | Meaning |
|---|---|---|
| `WORLDP` | `PF_EffectWorld*` | The world whose format determines the result |
| `PIXELPTR0` | `PF_PixelPtr` | A pixel pointer to reinterpret, or **NULL** to use `world->data` |
| `PIXEL8PP` / `PIXEL16PP` | `PF_Pixel8**` / `PF_Pixel16**` | Receives the typed pointer, or **NULL** on depth mismatch |

### Usage

```cpp
PF_Pixel8  *pix8  = NULL;
PF_Pixel16 *pix16 = NULL;

// Get base pointer for the world's own data
PF_GET_PIXEL_DATA8(worldP, NULL, &pix8);
if (pix8) {
    // World is 8-bit -- pix8 points to first pixel
}

PF_GET_PIXEL_DATA16(worldP, NULL, &pix16);
if (pix16) {
    // World is 16-bit -- pix16 points to first pixel
}
```

**There is no PF_GET_PIXEL_DATA_FLOAT macro.** Do not search for one, do not try to create one. Float access requires the suite.

## PF_PixelDataSuite1

Defined in `AE_EffectCBSuites.h`. Suite name: `"PF Pixel Data Suite"`, version 1. Frozen in AE 7.0.

```cpp
typedef struct PF_PixelDataSuite1 {
    PF_Err (*get_pixel_data8)(
        PF_EffectWorld  *worldP,
        PF_PixelPtr      pixelsP0,    // NULL to use data in PF_EffectWorld
        PF_Pixel8       **pixPP);     // will return NULL if depth mismatch

    PF_Err (*get_pixel_data16)(
        PF_EffectWorld  *worldP,
        PF_PixelPtr      pixelsP0,    // NULL to use data in PF_EffectWorld
        PF_Pixel16      **pixPP);     // will return NULL if depth mismatch

    PF_Err (*get_pixel_data_float)(
        PF_EffectWorld  *worldP,
        PF_PixelPtr      pixelsP0,    // NULL to use data in PF_EffectWorld
        PF_PixelFloat   **pixPP);     // will return NULL if depth mismatch
} PF_PixelDataSuite1;
```

All three functions follow the same pattern as the macros: pass the world, optionally pass a pixel pointer, and receive a typed pointer back. The pointer is NULL if the world's bit depth does not match the requested type.

## PF_PixelDataSuite2

Suite name: `"PF Pixel Data Suite"`, version 2. Frozen in AE 16.0.

Adds one function for GPU pixel access:

```cpp
typedef struct PF_PixelDataSuite2 {
    // ... same three functions as Suite1 ...

    PF_Err (*get_pixel_data_float_gpu)(
        PF_EffectWorld  *worldP,
        void           **pixPP);      // will return NULL if depth mismatch
} PF_PixelDataSuite2;
```

`get_pixel_data_float_gpu` returns an opaque GPU buffer pointer. This is used during GPU render selectors where the `PF_EffectWorld` wraps a GPU resource rather than CPU memory.

## Complete Example: Detecting Format and Accessing Pixels

```cpp
static PF_Err Render(
    PF_InData     *in_data,
    PF_EffectWorld *input,
    PF_EffectWorld *output)
{
    PF_Err err = PF_Err_NONE;

    // Acquire the suite (needed for float)
    AEFX_SuiteScoper<PF_PixelDataSuite2> pixSuite =
        AEFX_SuiteScoper<PF_PixelDataSuite2>(
            in_data, kPFPixelDataSuite, kPFPixelDataSuiteVersion2,
            "Couldn't acquire PF_PixelDataSuite2");

    PF_Pixel8     *inPix8     = NULL;
    PF_Pixel16    *inPix16    = NULL;
    PF_PixelFloat *inPixFloat = NULL;

    // Try each depth -- exactly one will succeed
    pixSuite->get_pixel_data8(input, NULL, &inPix8);
    pixSuite->get_pixel_data16(input, NULL, &inPix16);
    pixSuite->get_pixel_data_float(input, NULL, &inPixFloat);

    if (inPix8) {
        // Process 8-bit
        err = Process8bit(in_data, inPix8, output);
    } else if (inPix16) {
        // Process 16-bit
        err = Process16bit(in_data, inPix16, output);
    } else if (inPixFloat) {
        // Process 32-bit float
        err = ProcessFloat(in_data, inPixFloat, output);
    }

    return err;
}
```

## Walking Pixel Rows

The `data` pointer (once typed) points to the first pixel of the first row. To advance to the next row, use `rowbytes` -- **not** `width * sizeof(pixel)`, because rows may be padded.

```cpp
// 8-bit row walking
PF_Pixel8 *baseP = NULL;
PF_GET_PIXEL_DATA8(worldP, NULL, &baseP);

for (A_long y = 0; y < worldP->height; y++) {
    PF_Pixel8 *rowP = (PF_Pixel8 *)((char *)baseP + y * worldP->rowbytes);
    for (A_long x = 0; x < worldP->width; x++) {
        PF_Pixel8 *pixP = &rowP[x];
        // Access pixP->alpha, pixP->red, pixP->green, pixP->blue
    }
}

// 16-bit row walking -- same pattern, different type
PF_Pixel16 *baseP16 = NULL;
ERR(pixSuite->get_pixel_data16(worldP, NULL, &baseP16));

for (A_long y = 0; y < worldP->height; y++) {
    PF_Pixel16 *rowP = (PF_Pixel16 *)((char *)baseP16 + y * worldP->rowbytes);
    for (A_long x = 0; x < worldP->width; x++) {
        PF_Pixel16 *pixP = &rowP[x];
        // Channels range 0 to PF_MAX_CHAN16 (32768)
    }
}

// Float row walking
PF_PixelFloat *basePF = NULL;
ERR(pixSuite->get_pixel_data_float(worldP, NULL, &basePF));

for (A_long y = 0; y < worldP->height; y++) {
    PF_PixelFloat *rowP = (PF_PixelFloat *)((char *)basePF + y * worldP->rowbytes);
    for (A_long x = 0; x < worldP->width; x++) {
        PF_PixelFloat *pixP = &rowP[x];
        // 1.0 = white, values can exceed 1.0 (HDR overrange)
    }
}
```

## The PIXELPTR0 Parameter: Reinterpreting Iterate Callbacks

The second parameter (`pixelsP0`) exists for use inside `PF_Iterate` callback functions. When AE calls your pixel function, it gives you a `PF_PixelPtr` (opaque). You use the suite or macro to reinterpret it:

```cpp
static PF_Err MyPixelFunc8(
    void        *refcon,
    A_long       x,
    A_long       y,
    PF_Pixel    *inP,    // Already typed for 8-bit iterate
    PF_Pixel    *outP)
{
    // For 8-bit iterate, inP and outP are already PF_Pixel8*
    outP->red   = PF_MAX_CHAN8 - inP->red;
    outP->green = PF_MAX_CHAN8 - inP->green;
    outP->blue  = PF_MAX_CHAN8 - inP->blue;
    outP->alpha = inP->alpha;
    return PF_Err_NONE;
}

static PF_Err MyPixelFunc16(
    void         *refcon,
    A_long        x,
    A_long        y,
    PF_Pixel16   *inP,
    PF_Pixel16   *outP)
{
    outP->red   = PF_MAX_CHAN16 - inP->red;
    outP->green = PF_MAX_CHAN16 - inP->green;
    outP->blue  = PF_MAX_CHAN16 - inP->blue;
    outP->alpha = inP->alpha;
    return PF_Err_NONE;
}

static PF_Err MyPixelFuncFloat(
    void          *refcon,
    A_long         x,
    A_long         y,
    PF_PixelFloat *inP,
    PF_PixelFloat *outP)
{
    outP->red   = 1.0f - inP->red;
    outP->green = 1.0f - inP->green;
    outP->blue  = 1.0f - inP->blue;
    outP->alpha = inP->alpha;
    return PF_Err_NONE;
}
```

When using the iterate suites (`PF_Iterate8Suite1`, `PF_Iterate16Suite1`, `PF_IterateFloatSuite1`), the pixel function signature already uses the correct typed pointer, so you do not need the pixel data suite inside iterate callbacks.

## Macros vs Suite: When to Use Which

| Scenario | Use macros | Use suite |
|---|---|---|
| 8-bit only plugin | Yes | Either |
| 16-bit support | Yes | Either |
| 32-bit float support | **No** -- macro does not exist | **Must use suite** |
| GPU rendering | No | Suite2 required |
| Inside iterate callback | Not needed (already typed) | Not needed |
| Need to support all depths | Use suite for consistency | Recommended |

**Recommendation**: Always use the suite. It covers all cases and keeps your code consistent across bit depths. The macros exist for legacy compatibility but offer no advantage.

## Common Pitfalls

### 1. Assuming PF_GET_PIXEL_DATA_FLOAT exists

```cpp
// WRONG -- this macro does not exist and will not compile
PF_GET_PIXEL_DATA_FLOAT(worldP, NULL, &pixFloat);

// CORRECT -- must use the suite
AEFX_SuiteScoper<PF_PixelDataSuite1> pds(
    in_data, kPFPixelDataSuite, kPFPixelDataSuiteVersion1, "");
pds->get_pixel_data_float(worldP, NULL, &pixFloat);
```

### 2. Using width instead of rowbytes for row stride

```cpp
// WRONG -- rows may be padded
PF_Pixel8 *nextRow = baseP + worldP->width;

// CORRECT -- always use rowbytes
PF_Pixel8 *nextRow = (PF_Pixel8 *)((char *)baseP + worldP->rowbytes);
```

`rowbytes` is in **bytes**, not pixels. Always cast through `char*` when doing pointer arithmetic with `rowbytes`.

### 3. Forgetting that returned pointer can be NULL

The functions do not return an error on depth mismatch -- they succeed but set the output pointer to NULL. Always check:

```cpp
PF_Pixel8 *pix8 = NULL;
PF_Err err = pixSuite->get_pixel_data8(worldP, NULL, &pix8);
// err may be PF_Err_NONE even though pix8 is NULL (world is 16-bit or float)
if (!err && pix8) {
    // Safe to access pix8
}
```

### 4. Not defining PF_DEEP_COLOR_AWARE

If you do not `#define PF_DEEP_COLOR_AWARE 1` before including SDK headers, `PF_PixelPtr` resolves to `PF_Pixel*` and you lose the ability to handle 16-bit or float worlds correctly. The compiler will not warn you -- it will silently misinterpret pixel data.

Place this define in your precompiled header or at the very top of your main source file:

```cpp
#define PF_DEEP_COLOR_AWARE 1
#include "AE_Effect.h"
```

### 5. Mixing up PF_MAX_CHAN8 and PF_MAX_CHAN16

| Constant | Value | Used for |
|---|---|---|
| `PF_MAX_CHAN8` | 255 | 8-bit white point |
| `PF_HALF_CHAN8` | 128 | 8-bit midpoint |
| `PF_MAX_CHAN16` | 32768 | 16-bit white point (NOT 65535) |
| `PF_HALF_CHAN16` | 16384 | 16-bit midpoint |

AE's 16-bit range is 0 to 32768, not 0 to 65535. This is an AE-specific convention that differs from most other image processing software.

## SDK Header References

- **`AE_EffectCBSuites.h`** -- `PF_PixelDataSuite1`, `PF_PixelDataSuite2`, suite name and version constants
- **`AE_EffectCB.h`** -- `PF_GET_PIXEL_DATA8`, `PF_GET_PIXEL_DATA16` macro definitions, `get_pixel_data8`/`get_pixel_data16` in `_PF_UtilCallbacks`
- **`AE_Effect.h`** -- `PF_Pixel8`, `PF_Pixel16`, `PF_PixelFloat`, `PF_PixelPtr`, `PF_EffectWorld`/`PF_LayerDef`, `PF_WorldFlags`, `PF_MAX_CHAN8`, `PF_MAX_CHAN16`
- **`AEFX_SuiteHandlerTemplate.h`** -- `AEFX_SuiteScoper` template for safe suite acquisition
