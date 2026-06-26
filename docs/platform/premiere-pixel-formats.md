# Premiere Pro Pixel Formats and Rendering

After Effects uses a single pixel layout per bit depth (ARGB at 8, 16, or 32 bpc). Premiere Pro supports a large family of pixel formats covering RGB, YUV, packed, planar, and compressed layouts at multiple bit depths and color spaces. This document provides a complete reference for handling Premiere's pixel format system in cross-host AE-style plugins.

## Channel Order: The Fundamental Difference

| Host | Default 8-bit Format | Channel Order in Memory |
|------|---------------------|------------------------|
| After Effects | `PF_Pixel8` (ARGB) | `A R G B` (alpha first) |
| Premiere Pro | `PrPixelFormat_BGRA_4444_8u` | `B G R A` (alpha last, blue first) |

This is the single most important difference. If you write an AE plugin and run it unmodified in Premiere without registering pixel formats, Premiere will convert its native BGRA to ARGB before calling your render function. This works but incurs a per-frame conversion cost.

## The PF_PixelFormatSuite

To avoid format conversion overhead, register the Premiere pixel formats your plugin can handle natively. This is done via the `PF_PixelFormatSuite` (defined in `PrSDKAESupport.h`):

```cpp
#define kPFPixelFormatSuite         "PF Pixel Format Suite"
#define kPFPixelFormatSuiteVersion1 1
```

### Registering Supported Formats

During `PF_Cmd_GLOBAL_SETUP`, acquire the suite and register formats:

```cpp
static PF_Err GlobalSetup(PF_InData *in_data, PF_OutData *out_data) {
    PF_Err err = PF_Err_NONE;

    if (in_data->appl_id == 'PrMr') {
        // Register Premiere pixel formats
        PF_PixelFormatSuite1 *pfmtSuite = NULL;
        err = AEFX_AcquireSuite(in_data, out_data,
            kPFPixelFormatSuite, kPFPixelFormatSuiteVersion1,
            "Couldn't acquire PF_PixelFormatSuite",
            (void **)&pfmtSuite);

        if (!err && pfmtSuite) {
            // Register 8-bit BGRA (Premiere's most common format)
            pfmtSuite->ClearSupportedPixelFormats(in_data->effect_ref);

            pfmtSuite->AddSupportedPixelFormat(
                in_data->effect_ref, PrPixelFormat_BGRA_4444_8u);

            // Optionally add 32-bit float support
            pfmtSuite->AddSupportedPixelFormat(
                in_data->effect_ref, PrPixelFormat_BGRA_4444_32f);

            // Optionally add YUV support
            pfmtSuite->AddSupportedPixelFormat(
                in_data->effect_ref, PrPixelFormat_VUYA_4444_8u);

            AEFX_ReleaseSuite(in_data, out_data,
                kPFPixelFormatSuite, kPFPixelFormatSuiteVersion1,
                "Couldn't release PF_PixelFormatSuite");
        }
    }

    return err;
}
```

### Detecting the Current Pixel Format at Render Time

When Premiere calls your render function with a non-ARGB format, you need to determine what format you received. Use `GetPixelFormat`:

```cpp
static PF_Err Render(PF_InData *in_data, PF_OutData *out_data,
                     PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;

    if (in_data->appl_id == 'PrMr') {
        PF_PixelFormatSuite1 *pfmtSuite = NULL;
        ERR(AEFX_AcquireSuite(in_data, out_data,
            kPFPixelFormatSuite, kPFPixelFormatSuiteVersion1,
            NULL, (void **)&pfmtSuite));

        PrPixelFormat format = PrPixelFormat_Invalid;
        if (pfmtSuite) {
            pfmtSuite->GetPixelFormat(
                &params[0]->u.ld, &format);  // check input layer
        }

        switch (format) {
            case PrPixelFormat_BGRA_4444_8u:
                err = RenderBGRA8(in_data, params, output);
                break;
            case PrPixelFormat_BGRA_4444_32f:
                err = RenderBGRA32f(in_data, params, output);
                break;
            case PrPixelFormat_VUYA_4444_8u:
                err = RenderVUYA8(in_data, params, output);
                break;
            default:
                // Fall back to AE-style ARGB
                err = RenderARGB8(in_data, params, output);
                break;
        }
    } else {
        // After Effects: standard ARGB rendering
        err = RenderARGB(in_data, params, output);
    }

    return err;
}
```

## Complete PrPixelFormat Reference

All formats are defined in `PrSDKPixelFormat.h` using the `MAKE_PIXEL_FORMAT_FOURCC` macro.

### 8-bit Packed Formats

| Format | Alpha | Premultiplication | Color Space | Description |
|--------|-------|-------------------|-------------|-------------|
| `PrPixelFormat_BGRA_4444_8u` | Straight | No | RGB | Standard Windows 32-bit. Most common. |
| `PrPixelFormat_ARGB_4444_8u` | Straight | No | RGB | AE-compatible format |
| `PrPixelFormat_VUYA_4444_8u` | Yes | No | YUV BT.601 | 4:4:4 YUV |
| `PrPixelFormat_VUYA_4444_8u_709` | Yes | No | YUV BT.709 | 4:4:4 YUV, HD color space |
| `PrPixelFormat_BGRX_4444_8u` | None (X=unused) | N/A | RGB | Opaque RGB |
| `PrPixelFormat_VUYX_4444_8u` | None | N/A | YUV BT.601 | Opaque YUV |
| `PrPixelFormat_VUYX_4444_8u_709` | None | N/A | YUV BT.709 | Opaque YUV HD |
| `PrPixelFormat_XRGB_4444_8u` | None | N/A | RGB | Opaque XRGB |
| `PrPixelFormat_BGRP_4444_8u` | Premultiplied | Yes | RGB | Premultiplied BGRA |
| `PrPixelFormat_VUYP_4444_8u` | Premultiplied | Yes | YUV BT.601 | Premultiplied VUYA |
| `PrPixelFormat_VUYP_4444_8u_709` | Premultiplied | Yes | YUV BT.709 | Premultiplied VUYA HD |
| `PrPixelFormat_PRGB_4444_8u` | Premultiplied | Yes | RGB | Premultiplied ARGB |

### 16-bit Packed Formats

| Format | Alpha | Color Space |
|--------|-------|-------------|
| `PrPixelFormat_BGRA_4444_16u` | Yes | RGB |
| `PrPixelFormat_VUYA_4444_16u` | Yes | YUV |
| `PrPixelFormat_ARGB_4444_16u` | Yes | RGB |
| `PrPixelFormat_BGRX_4444_16u` | None | RGB |
| `PrPixelFormat_XRGB_4444_16u` | None | RGB |
| `PrPixelFormat_BGRP_4444_16u` | Premultiplied | RGB |
| `PrPixelFormat_PRGB_4444_16u` | Premultiplied | RGB |

### 32-bit Float Packed Formats

| Format | Alpha | Color Space | Notes |
|--------|-------|-------------|-------|
| `PrPixelFormat_BGRA_4444_32f` | Yes | RGB | Standard float BGRA |
| `PrPixelFormat_VUYA_4444_32f` | Yes | YUV BT.601 | |
| `PrPixelFormat_VUYA_4444_32f_709` | Yes | YUV BT.709 | |
| `PrPixelFormat_ARGB_4444_32f` | Yes | RGB | AE-compatible float |
| `PrPixelFormat_BGRX_4444_32f` | None | RGB | |
| `PrPixelFormat_VUYX_4444_32f` | None | YUV BT.601 | |
| `PrPixelFormat_VUYX_4444_32f_709` | None | YUV BT.709 | |
| `PrPixelFormat_XRGB_4444_32f` | None | RGB | |
| `PrPixelFormat_BGRP_4444_32f` | Premultiplied | RGB | |
| `PrPixelFormat_VUYP_4444_32f` | Premultiplied | YUV BT.601 | |
| `PrPixelFormat_VUYP_4444_32f_709` | Premultiplied | YUV BT.709 | |
| `PrPixelFormat_PRGB_4444_32f` | Premultiplied | RGB | |

### 32-bit Float Linear Formats

| Format | Alpha | Notes |
|--------|-------|-------|
| `PrPixelFormat_BGRA_4444_32f_Linear` | Yes | Linear light BGRA |
| `PrPixelFormat_BGRP_4444_32f_Linear` | Premultiplied | Linear light premul BGRA |
| `PrPixelFormat_BGRX_4444_32f_Linear` | None | Linear light opaque |
| `PrPixelFormat_ARGB_4444_32f_Linear` | Yes | Linear light ARGB |
| `PrPixelFormat_PRGB_4444_32f_Linear` | Premultiplied | Linear light premul ARGB |
| `PrPixelFormat_XRGB_4444_32f_Linear` | None | Linear light opaque XRGB |

### Subsampled YUV Formats

| Format | Subsampling | Bit Depth | Color Space |
|--------|-------------|-----------|-------------|
| `PrPixelFormat_YUYV_422_8u_601` | 4:2:2 | 8-bit | BT.601 |
| `PrPixelFormat_YUYV_422_8u_709` | 4:2:2 | 8-bit | BT.709 |
| `PrPixelFormat_UYVY_422_8u_601` | 4:2:2 | 8-bit | BT.601 |
| `PrPixelFormat_UYVY_422_8u_709` | 4:2:2 | 8-bit | BT.709 |
| `PrPixelFormat_V210_422_10u_601` | 4:2:2 | 10-bit packed | BT.601 |
| `PrPixelFormat_V210_422_10u_709` | 4:2:2 | 10-bit packed | BT.709 |
| `PrPixelFormat_UYVY_422_32f_601` | 4:2:2 | 32-bit float | BT.601 |
| `PrPixelFormat_UYVY_422_32f_709` | 4:2:2 | 32-bit float | BT.709 |

### Special Formats

| Format | Description |
|--------|-------------|
| `PrPixelFormat_RGB_444_10u` | Full range 10-bit 4:4:4 RGB (little-endian) |
| `PrPixelFormat_RGB_444_12u_PQ_709` | 12-bit PQ curve, BT.709 primaries (HDR) |
| `PrPixelFormat_RGB_444_12u_PQ_P3` | 12-bit PQ curve, P3 primaries (HDR) |
| `PrPixelFormat_RGB_444_12u_PQ_2020` | 12-bit PQ curve, BT.2020 primaries (HDR) |
| `PrPixelFormat_Raw` | Opaque raw data (no row bytes or height) |
| `PrPixelFormat_Invalid` | Error/uninitialized sentinel |
| `PrPixelFormat_Any` | Wildcard (value 0) |

### Planar YUV Formats

Premiere also supports numerous planar 4:2:0 YUV formats for MPEG-2 and MPEG-4 in both BT.601 and BT.709, frame and field picture modes, with limited and full range variants. These are primarily used by importers and hardware decoders, not typically by AE-style effects.

## Pixel Memory Layout

### BGRA 8-bit

```
Byte offset:  0    1    2    3
Channel:      B    G    R    A
Type:         uint8 per channel, range [0, 255]
```

```cpp
struct PrPixelBGRA_8u {
    A_u_char blue;
    A_u_char green;
    A_u_char red;
    A_u_char alpha;
};
```

### ARGB 8-bit (AE native)

```
Byte offset:  0    1    2    3
Channel:      A    R    G    B
```

### VUYA 8-bit

```
Byte offset:  0    1    2    3
Channel:      V    U    Y    A
              Cr   Cb   Luma Alpha
```

Where V = Cr (red difference), U = Cb (blue difference), Y = luma. Range [0, 255] with 128 as the zero point for U and V.

### BGRA 32-bit Float

```
Byte offset:  0-3    4-7    8-11   12-15
Channel:      B      G      R      A
Type:         float per channel, nominal range [0.0, 1.0]
              (values outside this range are valid for HDR)
```

### Premultiplied Variants (P suffix)

In premultiplied formats (`BGRP`, `VUYP`, `PRGB`), color channels are pre-multiplied by alpha:

```
R_stored = R_actual * A
G_stored = G_actual * A
B_stored = B_actual * A
```

The `X` suffix formats (`BGRX`, `VUYX`, `XRGB`) have no alpha channel -- the fourth byte/float is unused padding.

## Handling Both Hosts in a Single Render Function

A practical pattern for a plugin that processes pixels in both AE and Premiere:

```cpp
// Generic pixel processing - works regardless of channel order
template<typename PixelT, int R_IDX, int G_IDX, int B_IDX, int A_IDX>
static void ProcessRow(PixelT *rowP, A_long width, float intensity) {
    for (A_long x = 0; x < width; x++) {
        A_u_char *p = reinterpret_cast<A_u_char*>(&rowP[x]);
        p[R_IDX] = (A_u_char)(p[R_IDX] * intensity);
        p[G_IDX] = (A_u_char)(p[G_IDX] * intensity);
        p[B_IDX] = (A_u_char)(p[B_IDX] * intensity);
        // alpha unchanged
    }
}

static PF_Err Render8(PF_InData *in_data, PF_ParamDef *params[],
                      PF_LayerDef *output, PrPixelFormat format)
{
    float intensity = params[INTENSITY_PARAM]->u.fd.value / 100.0f;
    PF_LayerDef *input = &params[0]->u.ld;

    for (A_long y = 0; y < output->height; y++) {
        char *inRow  = (char *)input->data  + y * input->rowbytes;
        char *outRow = (char *)output->data + y * output->rowbytes;

        // Copy input to output first
        memcpy(outRow, inRow, abs(output->rowbytes));

        // Process based on format
        switch (format) {
            case PrPixelFormat_BGRA_4444_8u:
                // BGRA: B=0, G=1, R=2, A=3
                ProcessRow<PF_Pixel8, 2, 1, 0, 3>(
                    (PF_Pixel8 *)outRow, output->width, intensity);
                break;

            case PrPixelFormat_ARGB_4444_8u:
            default:
                // ARGB: A=0, R=1, G=2, B=3 (AE native)
                ProcessRow<PF_Pixel8, 1, 2, 3, 0>(
                    (PF_Pixel8 *)outRow, output->width, intensity);
                break;
        }
    }

    return PF_Err_NONE;
}
```

### Host-Agnostic Architecture

A cleaner approach is to abstract pixel access behind a format-aware wrapper:

```cpp
// Define channel indices per format
struct ChannelMap {
    int r, g, b, a;
};

static const ChannelMap kARGB = {1, 2, 3, 0};  // AE
static const ChannelMap kBGRA = {2, 1, 0, 3};  // Premiere

static ChannelMap GetChannelMap(PF_InData *in_data, PF_LayerDef *world) {
    if (in_data->appl_id == 'PrMr') {
        PF_PixelFormatSuite1 *pfmtSuite = NULL;
        // acquire suite...
        PrPixelFormat fmt = PrPixelFormat_Invalid;
        pfmtSuite->GetPixelFormat(world, &fmt);

        switch (fmt) {
            case PrPixelFormat_BGRA_4444_8u:
            case PrPixelFormat_BGRA_4444_16u:
            case PrPixelFormat_BGRA_4444_32f:
                return kBGRA;
            case PrPixelFormat_ARGB_4444_8u:
            case PrPixelFormat_ARGB_4444_32f:
            default:
                return kARGB;
        }
    }
    return kARGB;  // AE always uses ARGB
}
```

## YUV Considerations

If your plugin registers VUYA support, you must handle the YUV color space correctly. For effects that operate on luminance only (brightness, contrast, levels), processing the Y channel directly is more efficient than converting to RGB.

For effects that require RGB data (color correction, keying, etc.), convert YUV to RGB, process, then convert back:

```cpp
// BT.709 YUV -> RGB conversion (8-bit, limited range)
static void VUYA_to_RGBA(A_u_char V, A_u_char U, A_u_char Y, A_u_char A,
                         A_u_char *R, A_u_char *G, A_u_char *B, A_u_char *outA)
{
    float y = (Y - 16.0f) / 219.0f;
    float u = (U - 128.0f) / 224.0f;
    float v = (V - 128.0f) / 224.0f;

    float r = y + 1.5748f * v;
    float g = y - 0.1873f * u - 0.4681f * v;
    float b = y + 1.8556f * u;

    *R = (A_u_char)CLAMP(r * 255.0f, 0.0f, 255.0f);
    *G = (A_u_char)CLAMP(g * 255.0f, 0.0f, 255.0f);
    *B = (A_u_char)CLAMP(b * 255.0f, 0.0f, 255.0f);
    *outA = A;
}
```

> **Performance note**: If you only support BGRA and skip VUYA registration, Premiere will convert YUV footage to BGRA before calling your render. This is perfectly acceptable and simpler to implement. Only add VUYA support if you need the performance benefit of native YUV processing.

## Checked-Out Frames

By default, when your Premiere plugin checks out a frame from another layer (via `PF_CHECKOUT_PARAM`), Premiere may deliver it in a different pixel format than the render format. To force checked-out frames to match the render format, use the `PF_UtilitySuite`:

```cpp
PF_UtilitySuite *utilSuite = NULL;
ERR(AEFX_AcquireSuite(in_data, out_data,
    kPFUtilitySuite, kPFUtilitySuiteVersion,
    NULL, (void **)&utilSuite));

if (utilSuite) {
    utilSuite->EffectWantsCheckedOutFramesToMatchRenderPixelFormat(
        in_data->effect_ref);
}
```

This is declared in `PrSDKAESupport.h` and only applies in Premiere.

## Color Utilities

The `PF_PixelFormatSuite` provides helpers for format-aware color values:

```cpp
// Get black in the current pixel format
PF_Pixel8 black;
pfmtSuite->GetBlackForPixelFormat(PrPixelFormat_VUYA_4444_8u, &black);
// Returns V=128, U=128, Y=16, A=0 (BT.601 limited range black)

// Get white
PF_Pixel8 white;
pfmtSuite->GetWhiteForPixelFormat(PrPixelFormat_VUYA_4444_8u, &white);
// Returns V=128, U=128, Y=235, A=255

// Convert arbitrary color to format
float color_data[4];
pfmtSuite->ConvertColorToPixelFormattedData(
    PrPixelFormat_BGRA_4444_8u,
    1.0f,   // alpha
    1.0f,   // red
    0.0f,   // green
    0.0f,   // blue
    color_data);
// Writes format-appropriate bytes for "red" in BGRA layout
```

## Custom Pixel Formats

Third-party plugins can define custom pixel formats using the reserved macro:

```cpp
#define MAKE_THIRD_PARTY_CUSTOM_PIXEL_FORMAT_FOURCC(ch1, ch2, ch3) \
    (static_cast<PrPixelFormat>(MAKE_PIXEL_FORMAT_FOURCC('_', ch1, ch2, ch3)))
```

Custom formats start with `'_'` as the first byte. This is primarily used by importers and exporters that deal with proprietary pixel layouts.

## Recommended Format Support by Plugin Type

### Simple Effects (brightness, blur, color adjust)

Register just BGRA and let Premiere convert everything else:

```cpp
pfmtSuite->AddSupportedPixelFormat(in_data->effect_ref, PrPixelFormat_BGRA_4444_8u);
pfmtSuite->AddSupportedPixelFormat(in_data->effect_ref, PrPixelFormat_BGRA_4444_32f);
```

### Color Grading / Professional Effects

Support the full range for maximum performance:

```cpp
// 8-bit
pfmtSuite->AddSupportedPixelFormat(in_data->effect_ref, PrPixelFormat_BGRA_4444_8u);
pfmtSuite->AddSupportedPixelFormat(in_data->effect_ref, PrPixelFormat_VUYA_4444_8u);
pfmtSuite->AddSupportedPixelFormat(in_data->effect_ref, PrPixelFormat_VUYA_4444_8u_709);
// 32-bit float
pfmtSuite->AddSupportedPixelFormat(in_data->effect_ref, PrPixelFormat_BGRA_4444_32f);
pfmtSuite->AddSupportedPixelFormat(in_data->effect_ref, PrPixelFormat_VUYA_4444_32f);
pfmtSuite->AddSupportedPixelFormat(in_data->effect_ref, PrPixelFormat_VUYA_4444_32f_709);
```

32-bit float is essential for color grading workflows where values outside `[0, 1]` carry meaningful HDR information.

### Minimal / Compatibility Focused

Register nothing. Premiere will convert to ARGB 8-bit for you, matching AE's default format. Simplest but slowest.

## Common Pitfalls

**Forgetting that BGRA is not ARGB.** Swapped channels produce color-shifted output (blue becomes red, etc.). Always check the format before processing pixels.

**Assuming row bytes are positive.** Premiere can provide negative row bytes (bottom-up buffer layout). Use the absolute value of row bytes for `memcpy` lengths, and always compute row pointers arithmetically: `base + y * rowbytes`.

**Ignoring premultiplication state.** The `P` suffix formats (BGRP, VUYP, PRGB) deliver premultiplied pixels. If your effect manipulates alpha, you must un-premultiply, process, and re-premultiply to avoid artifacts.

**Not supporting 32-bit float.** Without 32-bit support, your plugin will not get the "32-bit" badge in Premiere's effects list, and it will force Premiere to down-convert HDR timelines to 8-bit for your effect.

**Using AE iterate suites for non-ARGB data.** The standard `PF_Iterate8Suite` assumes ARGB layout and will produce incorrect results for BGRA or VUYA data. For BGRA 8-bit, use Premiere's dedicated iterate suite. For other formats, write your own iteration.

**Not calling ClearSupportedPixelFormats first.** Always call `ClearSupportedPixelFormats` before `AddSupportedPixelFormat` to reset any previously registered formats. Failure to clear can cause duplicate registrations across GlobalSetup calls.
