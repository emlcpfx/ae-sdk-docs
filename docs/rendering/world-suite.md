# World Management: PF_WorldSuite1 vs PF_WorldSuite2

> How to create, query, and dispose `PF_EffectWorld` buffers at every bit depth, and how to register supported pixel formats.

## Overview

A `PF_EffectWorld` (also known as `PF_LayerDef`) is the SDK's pixel buffer type. Effects receive worlds as input/output, and can also create temporary worlds for intermediate processing. There are two generations of world management suites plus a legacy callback, each with different approaches to specifying bit depth.

| API | How depth is specified | Float support | Format query |
|---|---|---|---|
| Legacy callback (`in_data->utils->new_world`) | `PF_NewWorldFlags` | No | No |
| `PF_WorldSuite1` | `PF_NewWorldFlags` | No | No |
| `PF_WorldSuite2` | `PF_PixelFormat` enum | Yes | Yes (`PF_GetPixelFormat`) |

**Key insight**: `PF_NewWorldFlags` (for creation) and `PF_WorldFlags` (for querying existing worlds) are **different enums** with different values and purposes. Confusing them is a common source of bugs.

## The PF_EffectWorld Structure

From `AE_Effect.h`:

```cpp
typedef struct PF_LayerDef {
    void             *reserved0;
    void             *reserved1;
    PF_WorldFlags     world_flags;    // Query flags (PF_WorldFlag_DEEP, etc.)
    PF_PixelPtr       data;           // Opaque pixel buffer pointer
    A_long            rowbytes;       // Bytes per row (includes padding)
    A_long            width;          // Width in pixels
    A_long            height;         // Height in pixels
    PF_UnionableRect  extent_hint;    // Relevant area rect
    void             *platform_ref;   // Unused since CS5
    A_long            reserved_long1;
    void             *reserved_long4;
    PF_RationalScale  pix_aspect_ratio;
    void             *reserved_long2;
    A_long            origin_x;       // SmartFX checkouts only
    A_long            origin_y;
    A_long            reserved_long3;
    A_long            dephault;       // PF_LayerDefault constant
} PF_LayerDef;

typedef PF_LayerDef  PF_EffectWorld;
```

## PF_WorldFlags vs PF_NewWorldFlags -- DIFFERENT ENUMS

This is the most common point of confusion. These are **separate enums** used in **separate contexts**.

### PF_WorldFlags -- Querying Existing Worlds

Defined in `AE_Effect.h`. Read from `world->world_flags` to inspect a world you have received.

```cpp
enum {
    PF_WorldFlag_DEEP        = 1L << 0,   // World is 16-bit or higher
    PF_WorldFlag_WRITEABLE   = 1L << 1,   // World can be written to
    // Reserved flags 24-31 -- do not use
};
typedef A_long PF_WorldFlags;
```

The convenience macro:

```cpp
#define PF_WORLD_IS_DEEP(W)  ( ((W)->world_flags & PF_WorldFlag_DEEP) != 0 )
```

**Limitation**: `PF_WorldFlag_DEEP` is set for *both* 16-bit and 32-bit float worlds. It cannot distinguish between them. To determine the exact format, you must use `PF_WorldSuite2::PF_GetPixelFormat`.

### PF_NewWorldFlags -- Creating New Worlds

Defined in `AE_EffectCB.h`. Passed to `PF_WorldSuite1::new_world` and the legacy `new_world` callback.

```cpp
enum {
    PF_NewWorldFlag_NONE          = 0,
    PF_NewWorldFlag_CLEAR_PIXELS  = 1L << 0,  // Zero all pixels on creation
    PF_NewWorldFlag_DEEP_PIXELS   = 1L << 1,  // Create 16-bit world
    // Reserved flags 2-3 -- do not use
};
typedef A_long PF_NewWorldFlags;
```

**Limitation**: `PF_NewWorldFlag_DEEP_PIXELS` creates a 16-bit world. There is no flag to create a 32-bit float world through WorldSuite1 or the legacy callback. To create a float world, you must use `PF_WorldSuite2::PF_NewWorld`.

### Summary of the Confusion

| Enum | Header | Used for | Can specify float? |
|---|---|---|---|
| `PF_WorldFlags` | `AE_Effect.h` | Querying `world->world_flags` | Can detect "deep" but not distinguish 16 vs 32 |
| `PF_NewWorldFlags` | `AE_EffectCB.h` | Creating worlds via Suite1/callback | No -- only 8 or 16 |

## PF_PixelFormat Enum

Defined in `AE_EffectPixelFormat.h`. This is the modern, precise way to identify pixel formats.

```cpp
enum {
    // After Effects CPU formats
    PF_PixelFormat_ARGB32     // 8-bit ARGB, channels 0-255
    PF_PixelFormat_ARGB64     // 16-bit ARGB, channels 0-32768
    PF_PixelFormat_ARGB128    // 32-bit float ARGB, 1.0 = white

    // GPU format
    PF_PixelFormat_GPU_BGRA128  // GPU, 32-bit float BGRA (note: BGRA, not ARGB)

    // Reserved
    PF_PixelFormat_RESERVED

    // Premiere-specific (not used in AE)
    PF_PixelFormat_BGRA32     // 8-bit BGRA
    PF_PixelFormat_VUYA32     // 8-bit YUVA

    PF_PixelFormat_INVALID    // Error/initialization sentinel
};
typedef A_long PF_PixelFormat;
```

The values are FOURCC codes:

| Format | FOURCC | Description |
|---|---|---|
| `PF_PixelFormat_ARGB32` | `'argb'` | 8 bits/channel, range 0-255 |
| `PF_PixelFormat_ARGB64` | `'ae16'` | 16 bits/channel, range 0-32768 |
| `PF_PixelFormat_ARGB128` | `'ae32'` | 32-bit float/channel, 1.0 = white |
| `PF_PixelFormat_GPU_BGRA128` | `'@CDA'` | GPU, 32-bit float, BGRA order |
| `PF_PixelFormat_INVALID` | `'badf'` | Sentinel value |

**Important**: GPU format uses **BGRA** channel order, not ARGB. CPU formats all use **ARGB** order.

## PF_WorldSuite1

Suite name: `"PF World Suite"`, version 1. Frozen in AE 5.0.

```cpp
typedef struct PF_WorldSuite1 {
    PF_Err (*new_world)(
        PF_ProgPtr        effect_ref,
        A_long            width,
        A_long            height,
        PF_NewWorldFlags  flags,      // PF_NewWorldFlag_CLEAR_PIXELS | PF_NewWorldFlag_DEEP_PIXELS
        PF_EffectWorld   *world);

    PF_Err (*dispose_world)(
        PF_ProgPtr        effect_ref,
        PF_EffectWorld   *world);
} PF_WorldSuite1;
```

### Creating worlds with Suite1

```cpp
AEFX_SuiteScoper<PF_WorldSuite1> ws1(
    in_data, kPFWorldSuite, kPFWorldSuiteVersion1, "");

PF_EffectWorld tempWorld;

// Create 8-bit world, cleared to zero
ERR(ws1->new_world(in_data->effect_ref,
    width, height,
    PF_NewWorldFlag_CLEAR_PIXELS,
    &tempWorld));

// Create 16-bit world, cleared to zero
ERR(ws1->new_world(in_data->effect_ref,
    width, height,
    PF_NewWorldFlag_CLEAR_PIXELS | PF_NewWorldFlag_DEEP_PIXELS,
    &tempWorld));

// CANNOT create 32-bit float world with Suite1!

// Always dispose when done
ERR(ws1->dispose_world(in_data->effect_ref, &tempWorld));
```

## PF_WorldSuite2

Suite name: `"PF World Suite"`, version 2. Frozen in AE 7.0.

```cpp
typedef struct PF_WorldSuite2 {
    PF_Err (*PF_NewWorld)(
        PF_ProgPtr       effect_ref,
        A_long           widthL,
        A_long           heightL,
        PF_Boolean       clear_pixB,     // TRUE to zero pixels
        PF_PixelFormat   pixel_format,   // Explicit format enum
        PF_EffectWorld  *worldP);

    PF_Err (*PF_DisposeWorld)(
        PF_ProgPtr       effect_ref,
        PF_EffectWorld  *worldP);

    PF_Err (*PF_GetPixelFormat)(
        const PF_EffectWorld  *worldP,
        PF_PixelFormat        *pixel_formatP);   // << OUT
} PF_WorldSuite2;
```

### Creating worlds with Suite2

```cpp
AEFX_SuiteScoper<PF_WorldSuite2> ws2(
    in_data, kPFWorldSuite, kPFWorldSuiteVersion2, "");

PF_EffectWorld tempWorld;

// Create 8-bit world
ERR(ws2->PF_NewWorld(in_data->effect_ref,
    width, height,
    TRUE,                        // clear pixels
    PF_PixelFormat_ARGB32,       // 8-bit
    &tempWorld));

// Create 16-bit world
ERR(ws2->PF_NewWorld(in_data->effect_ref,
    width, height,
    TRUE,
    PF_PixelFormat_ARGB64,       // 16-bit
    &tempWorld));

// Create 32-bit float world -- ONLY possible with Suite2
ERR(ws2->PF_NewWorld(in_data->effect_ref,
    width, height,
    TRUE,
    PF_PixelFormat_ARGB128,      // 32-bit float
    &tempWorld));

ERR(ws2->PF_DisposeWorld(in_data->effect_ref, &tempWorld));
```

### Querying pixel format

```cpp
PF_PixelFormat format = PF_PixelFormat_INVALID;
ERR(ws2->PF_GetPixelFormat(input, &format));

switch (format) {
    case PF_PixelFormat_ARGB32:
        // 8-bit processing
        break;
    case PF_PixelFormat_ARGB64:
        // 16-bit processing
        break;
    case PF_PixelFormat_ARGB128:
        // 32-bit float processing
        break;
    case PF_PixelFormat_GPU_BGRA128:
        // GPU float processing (BGRA order!)
        break;
}
```

This is the **only reliable way** to distinguish 16-bit from 32-bit float. The `PF_WorldFlag_DEEP` flag in `world->world_flags` is set for both.

## Creating Temp Worlds That Match Input Format

A common pattern is to create a temporary world at the same bit depth as the input:

```cpp
static PF_Err CreateMatchingWorld(
    PF_InData      *in_data,
    PF_EffectWorld *reference,
    A_long          width,
    A_long          height,
    PF_EffectWorld *newWorldP)
{
    PF_Err err = PF_Err_NONE;

    AEFX_SuiteScoper<PF_WorldSuite2> ws2(
        in_data, kPFWorldSuite, kPFWorldSuiteVersion2, "");

    PF_PixelFormat format = PF_PixelFormat_INVALID;
    ERR(ws2->PF_GetPixelFormat(reference, &format));

    if (!err) {
        ERR(ws2->PF_NewWorld(in_data->effect_ref,
            width, height,
            TRUE,       // clear
            format,
            newWorldP));
    }

    return err;
}
```

## PF_PixelFormatSuite2 -- Registering Supported Formats

Suite name: `"PF Pixel Format Suite"`, version 2.

```cpp
typedef struct PF_PixelFormatSuite2 {
    PF_Err (*PF_AddSupportedPixelFormat)(
        PF_ProgPtr      effect_ref,
        PF_PixelFormat  pixel_format);

    PF_Err (*PF_ClearSupportedPixelFormats)(
        PF_ProgPtr      effect_ref);
} PF_PixelFormatSuite2;
```

Call `PF_AddSupportedPixelFormat` during `PF_Cmd_GLOBAL_SETUP` to register the pixel formats your effect can handle. This is the modern, explicit way to declare format support and works in conjunction with the out flags.

### Registration example

```cpp
static PF_Err GlobalSetup(
    PF_InData   *in_data,
    PF_OutData  *out_data)
{
    PF_Err err = PF_Err_NONE;

    // Set out_flags for backwards compatibility
    out_data->out_flags  |= PF_OutFlag_DEEP_COLOR_AWARE;
    out_data->out_flags2 |= PF_OutFlag2_FLOAT_COLOR_AWARE |
                            PF_OutFlag2_SUPPORTS_SMART_RENDER;

    // Explicitly register supported formats
    AEFX_SuiteScoper<PF_PixelFormatSuite2> pfs(
        in_data, kPFPixelFormatSuite, kPFPixelFormatSuiteVersion2, "");

    ERR(pfs->PF_AddSupportedPixelFormat(in_data->effect_ref,
        PF_PixelFormat_ARGB32));     // 8-bit

    ERR(pfs->PF_AddSupportedPixelFormat(in_data->effect_ref,
        PF_PixelFormat_ARGB64));     // 16-bit

    ERR(pfs->PF_AddSupportedPixelFormat(in_data->effect_ref,
        PF_PixelFormat_ARGB128));    // 32-bit float

    return err;
}
```

## Relationship Between Out Flags and Pixel Format Registration

Three mechanisms control bit depth support, and they interact:

### PF_OutFlag_DEEP_COLOR_AWARE (out_flags, bit 25)

Set in `PF_Cmd_GLOBAL_SETUP`. Tells AE the effect can handle 16-bit worlds. Without this flag, AE will always convert input to 8-bit before calling your effect.

### PF_OutFlag2_FLOAT_COLOR_AWARE (out_flags2, bit 12)

Set in `PF_Cmd_GLOBAL_SETUP`. Tells AE the effect can handle 32-bit float worlds. Typically requires `PF_OutFlag2_SUPPORTS_SMART_RENDER` as well.

### PF_AddSupportedPixelFormat

The explicit, modern registration. When used, AE will only deliver worlds in formats you have registered.

### How they combine

| Configuration | Effect receives |
|---|---|
| Neither flag set | 8-bit only |
| `DEEP_COLOR_AWARE` only | 8-bit and 16-bit |
| `DEEP_COLOR_AWARE` + `FLOAT_COLOR_AWARE` | 8-bit, 16-bit, and float |
| `DEEP_COLOR_AWARE` + `FLOAT_COLOR_AWARE` + `AddSupportedPixelFormat(ARGB128)` only | Float only (AE converts other depths) |

When you call `PF_AddSupportedPixelFormat`, you are being *more specific* than the flags alone. If you register only `ARGB128`, AE will convert 8-bit and 16-bit compositions to float before calling your effect, even though the flags indicate general deep/float awareness.

**Best practice**: Set both flags AND register all formats you support. The flags ensure backwards compatibility with older AE versions; the format registration gives modern AE precise information.

## GPU Pixel Format

For GPU-accelerated effects, register the GPU format as well:

```cpp
// In GLOBAL_SETUP, if supporting GPU rendering:
out_data->out_flags2 |= PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;

ERR(pfs->PF_AddSupportedPixelFormat(in_data->effect_ref,
    PF_PixelFormat_GPU_BGRA128));
```

Note the GPU format is **BGRA** (blue, green, red, alpha) rather than ARGB. Your GPU shader must account for the different channel order.

## Common Pitfalls

### 1. Confusing PF_WorldFlags with PF_NewWorldFlags

```cpp
// WRONG -- PF_WorldFlag_DEEP is a query flag, not a creation flag
PF_EffectWorld temp;
ws1->new_world(ref, w, h, PF_WorldFlag_DEEP, &temp);  // Undefined behavior!

// CORRECT -- use the creation flag
ws1->new_world(ref, w, h, PF_NewWorldFlag_DEEP_PIXELS, &temp);
```

These enums have different values: `PF_WorldFlag_DEEP` is `1L << 0` (value 1), and `PF_NewWorldFlag_DEEP_PIXELS` is also `1L << 1` (value 2). The compiler will not catch this mistake because both are `A_long`.

### 2. Using PF_WORLD_IS_DEEP to determine float vs 16-bit

```cpp
// INSUFFICIENT -- cannot tell 16-bit from 32-bit float
if (PF_WORLD_IS_DEEP(input)) {
    // This could be 16-bit OR 32-bit float!
}

// CORRECT -- use PF_GetPixelFormat
PF_PixelFormat fmt;
ws2->PF_GetPixelFormat(input, &fmt);
if (fmt == PF_PixelFormat_ARGB128) {
    // Definitely 32-bit float
}
```

### 3. Trying to create float worlds with Suite1

```cpp
// WRONG -- Suite1 has no way to create float worlds
ws1->new_world(ref, w, h,
    PF_NewWorldFlag_DEEP_PIXELS | PF_NewWorldFlag_CLEAR_PIXELS,
    &temp);
// This creates a 16-bit world, not float!

// CORRECT -- use Suite2 for float
ws2->PF_NewWorld(ref, w, h, TRUE, PF_PixelFormat_ARGB128, &temp);
```

### 4. Forgetting to dispose temporary worlds

Every world created with `new_world` / `PF_NewWorld` must be disposed with `dispose_world` / `PF_DisposeWorld`. Failing to dispose causes memory leaks that grow with each frame rendered. Use RAII wrappers in C++:

```cpp
struct WorldGuard {
    PF_InData      *in;
    PF_WorldSuite2 *ws;
    PF_EffectWorld  world;
    bool            valid;

    WorldGuard(PF_InData *in_, PF_WorldSuite2 *ws_,
               A_long w, A_long h, PF_PixelFormat fmt)
        : in(in_), ws(ws_), valid(false) {
        if (!ws->PF_NewWorld(in->effect_ref, w, h, TRUE, fmt, &world))
            valid = true;
    }
    ~WorldGuard() {
        if (valid) ws->PF_DisposeWorld(in->effect_ref, &world);
    }
};
```

### 5. Setting FLOAT_COLOR_AWARE without DEEP_COLOR_AWARE

`PF_OutFlag2_FLOAT_COLOR_AWARE` requires `PF_OutFlag_DEEP_COLOR_AWARE` to be set as well. Setting only the float flag without the deep flag results in 8-bit only operation.

```cpp
// WRONG -- float flag alone is not sufficient
out_data->out_flags2 |= PF_OutFlag2_FLOAT_COLOR_AWARE;

// CORRECT -- both flags required
out_data->out_flags  |= PF_OutFlag_DEEP_COLOR_AWARE;
out_data->out_flags2 |= PF_OutFlag2_FLOAT_COLOR_AWARE;
```

## SDK Header References

- **`AE_EffectCBSuites.h`** -- `PF_WorldSuite1`, `PF_WorldSuite2`, `PF_PixelFormatSuite2`, suite name and version constants
- **`AE_EffectCB.h`** -- `PF_NewWorldFlags` enum, legacy `new_world`/`dispose_world` callbacks in `_PF_UtilCallbacks`
- **`AE_Effect.h`** -- `PF_WorldFlags` enum, `PF_WORLD_IS_DEEP` macro, `PF_LayerDef`/`PF_EffectWorld` structure, `PF_OutFlag_DEEP_COLOR_AWARE`, `PF_OutFlag2_FLOAT_COLOR_AWARE`
- **`AE_EffectPixelFormat.h`** -- `PF_PixelFormat` enum with FOURCC values, `PF_PixelFormat_ARGB32`/`ARGB64`/`ARGB128`/`GPU_BGRA128`
