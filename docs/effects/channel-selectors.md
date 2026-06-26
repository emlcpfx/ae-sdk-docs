# Channel Selectors, Pixel Macros, and Plane Constants

After Effects provides several systems for working with individual pixel channels: the `PF_Plane` bitmask constants for selecting image planes, `PF_PixLong` macros for packed 32-bit pixel manipulation, and the Channel Suite types (`PF_ChannelType`, `PF_DataType`) for accessing auxiliary render channels from 3D renderers. This document covers all three systems.

---

## PF_Plane Constants (Image Plane Selectors)

Defined in `AE_EffectCB.h`, these bitmask constants select subsets of the four ARGB image planes:

```cpp
enum {
    PF_Plane_ALPHA = 1,   // bit 0
    PF_Plane_RED   = 2,   // bit 1
    PF_Plane_GREEN = 4,   // bit 2
    PF_Plane_BLUE  = 8    // bit 3
};
typedef A_u_long PF_Plane;
```

### Usage

These constants are used with callbacks like `convolve` that operate on individual planes:

```cpp
// Convolve only the red and green channels
PF_Plane planes = PF_Plane_RED | PF_Plane_GREEN;
```

### Common Combinations

| Combination | Value | Meaning |
|---|---|---|
| `PF_Plane_ALPHA` | 1 | Alpha only |
| `PF_Plane_RED` | 2 | Red only |
| `PF_Plane_GREEN` | 4 | Green only |
| `PF_Plane_BLUE` | 8 | Blue only |
| `PF_Plane_RED \| PF_Plane_GREEN \| PF_Plane_BLUE` | 14 | RGB (no alpha) |
| All four OR'd together | 15 | All channels |

---

## PF_PixLong Access Macros

`PF_PixLong` is a packed 32-bit unsigned integer representation of an 8-bit ARGB pixel. Defined in `AE_Effect.h`:

```cpp
typedef A_u_long PF_PixLong;
```

The layout is: `[Alpha:8][Red:8][Green:8][Blue:8]` (most significant to least significant byte).

### Read Macros

```cpp
#define PF_PixLong_ALPHA(pl)    ((A_u_char)(0xff & ((pl) >> 24)))
#define PF_PixLong_RED(pl)      ((A_u_char)(0xff & ((pl) >> 16)))
#define PF_PixLong_GREEN(pl)    ((A_u_char)(0xff & ((pl) >> 8)))
#define PF_PixLong_BLUE(pl)     ((A_u_char)(0xff & (pl)))
```

```cpp
PF_PixLong pixel = 0xFF804020;  // A=255, R=128, G=64, B=32

A_u_char a = PF_PixLong_ALPHA(pixel);  // 255
A_u_char r = PF_PixLong_RED(pixel);    // 128
A_u_char g = PF_PixLong_GREEN(pixel);  // 64
A_u_char b = PF_PixLong_BLUE(pixel);   // 32
```

### Write Macros

```cpp
#define PF_SET_PixLong_ALPHA(pl, v)   (pl)&=0x00ffffff, (pl)|=((A_long)(v)<<24)
#define PF_SET_PixLong_RED(pl, v)     (pl)&=0xff00ffff, (pl)|=((A_long)(v)<<16)
#define PF_SET_PixLong_GREEN(pl, v)   (pl)&=0xffff00ff, (pl)|=((A_long)(v)<<8)
#define PF_SET_PixLong_BLUE(pl, v)    (pl)&=0xffffff00, (pl)|=(0xff & (v))
```

> **Note:** The SET macros use the comma operator and modify `pl` in place. They are statements, not expressions. The `v` parameter should be in range 0-255.

```cpp
PF_PixLong pixel = 0;
PF_SET_PixLong_ALPHA(pixel, 255);
PF_SET_PixLong_RED(pixel, 128);
PF_SET_PixLong_GREEN(pixel, 64);
PF_SET_PixLong_BLUE(pixel, 32);
// pixel is now 0xFF804020
```

### Construction Macro

```cpp
#define PF_MAKE_PixLong(a, r, g, b) \
    ((PF_PixLong)(((A_long)(a)<<24) | ((A_long)(r)<<16) | ((A_long)(g)<<8) | (b)))
```

```cpp
PF_PixLong pixel = PF_MAKE_PixLong(255, 128, 64, 32);  // 0xFF804020
```

### PF_PixLong vs PF_Pixel

`PF_PixLong` is a packed integer; `PF_Pixel` is a struct with named fields:

```cpp
typedef struct {
    A_u_char  alpha, red, green, blue;
} PF_Pixel;
```

Use `PF_Pixel` for direct struct access (the standard pattern in iterate callbacks). Use `PF_PixLong` when you need to treat a pixel as a single integer for bitwise operations, hashing, comparison, or storage in integer arrays.

> **PITFALL:** The byte order of `PF_Pixel` struct members in memory depends on the compiler's struct packing. It is NOT guaranteed to match the bit layout of `PF_PixLong`. Do not cast between them; use the macros.

---

## PF_ChannelType Constants (Auxiliary Channels)

These FourCC constants identify the type of auxiliary (non-color) channel data available from 3D renderers and other sources. Defined in `AE_Effect.h`:

```cpp
#define PF_ChannelType_DEPTH         'DPTH'   // Z-depth
#define PF_ChannelType_DEPTHAA       'DPAA'   // Z-depth with anti-aliasing (since AE 16.0)
#define PF_ChannelType_NORMALS       'NRML'   // Surface normals (3D)
#define PF_ChannelType_OBJECTID      'OBID'   // Object/material ID
#define PF_ChannelType_MOTIONVECTOR  'MTVR'   // Motion vectors (2D)
#define PF_ChannelType_BK_COLOR      'BKCR'   // Background color
#define PF_ChannelType_TEXTURE       'TEXR'   // Texture coordinates (UV)
#define PF_ChannelType_COVERAGE      'COVR'   // Coverage/anti-aliasing
#define PF_ChannelType_NODE          'NODE'   // Node ID
#define PF_ChannelType_MATERIAL      'MATR'   // Material ID
#define PF_ChannelType_UNCLAMPED     'UNCP'   // Unclamped color
#define PF_ChannelType_UNKNOWN       'UNKN'   // Unknown channel type

typedef A_long PF_ChannelType;
```

### When These Are Used

These types appear when using the `PF_ChannelSuite1` to access auxiliary channels from 3D layers (Classic 3D renderer, CINEMA 4D renderer, or third-party Artisan renderers). They are NOT used for standard ARGB pixel access.

---

## PF_DataType Constants (Data Format)

These FourCC constants describe the element data type within a channel. Defined in `AE_Effect.h`:

```cpp
#define PF_DataType_FLOAT           'FLT4'   // 4 bytes (float)
#define PF_DataType_DOUBLE          'DBL8'   // 8 bytes (double)
#define PF_DataType_LONG            'LON4'   // 4 bytes (A_long)
#define PF_DataType_SHORT           'SHT2'   // 2 bytes (A_short)
#define PF_DataType_FIXED_16_16     'FIX4'   // 4 bytes (16.16 fixed)
#define PF_DataType_CHAR            'CHR1'   // 1 byte (A_char)
#define PF_DataType_U_BYTE          'UBT1'   // 1 byte (A_u_char)
#define PF_DataType_U_SHORT         'UST2'   // 2 bytes (A_u_short)
#define PF_DataType_U_FIXED_16_16   'UFX4'   // 4 bytes (unsigned 16.16 fixed)
#define PF_DataType_RGB             'RBG '   // 3 bytes (RGB triplet)

typedef A_long PF_DataType;
```

The last character(s) of each FourCC encode the byte size per element (convention only).

### Relationship to PF_ChannelType

A channel has both a **type** (what kind of data: depth, normals, etc.) and a **data type** (how that data is stored: float, short, etc.). Combined with the **dimension** (number of values per pixel), this fully describes the channel format.

Example: `PF_ChannelType_NORMALS` with `PF_DataType_FLOAT` and `dimension = 3` means three float values (nx, ny, nz) per pixel, totaling 12 bytes per pixel.

---

## Channel Suite Usage

### PF_ChannelDesc

```cpp
#define PF_CHANNEL_NAME_LEN  63

typedef struct {
    PF_ChannelType  channel_type;
    A_char          name[PF_CHANNEL_NAME_LEN + 1];
    PF_DataType     data_type;
    A_long          dimension;    // number of data values per pixel (e.g., 3 for normals)
} PF_ChannelDesc;
```

### PF_ChannelChunk

```cpp
typedef struct {
    PF_ChannelRef   channel_ref;
    A_long          widthL;
    A_long          heightL;
    A_long          dimensionL;
    A_long          row_bytesL;
    PF_DataType     data_type;
    PF_Handle       dataH;
    void            *dataPV;    // pointer to dereferenced locked handle
} PF_ChannelChunk;
```

### Example: Reading Z-Depth

```cpp
AEFX_SuiteScoper<PF_ChannelSuite1> ch_suite(
    in_data, kPFChannelSuite1, kPFChannelSuiteVersion1);

// Check for depth channel on the input layer
PF_Boolean found = FALSE;
PF_ChannelRef depth_ref;
PF_ChannelDesc depth_desc;

ERR(ch_suite->PF_GetLayerChannelTypedRefAndDesc(
    in_data->effect_ref,
    0,                          // param_index 0 = input layer
    PF_ChannelType_DEPTH,
    &found,
    &depth_ref,
    &depth_desc));

if (found) {
    PF_ChannelChunk depth_chunk;
    AEFX_CLR_STRUCT(depth_chunk);

    ERR(ch_suite->PF_CheckoutLayerChannel(
        in_data->effect_ref,
        &depth_ref,
        in_data->current_time,
        in_data->time_step,
        in_data->time_scale,
        PF_DataType_FLOAT,      // request data as float
        &depth_chunk));

    if (!err && depth_chunk.dataPV) {
        // Access depth data
        for (A_long y = 0; y < depth_chunk.heightL; y++) {
            A_FpShort* row = NULL;
            PF_GET_CHANNEL_ROW_FLOAT_DATA(depth_chunk, y, row);
            if (row) {
                for (A_long x = 0; x < depth_chunk.widthL; x++) {
                    A_FpShort depth = row[x * depth_chunk.dimensionL];
                    // Use depth value...
                }
            }
        }
    }

    // Must check in the channel
    ERR2(ch_suite->PF_CheckinLayerChannel(
        in_data->effect_ref, &depth_ref, &depth_chunk));
}
```

### Channel Data Access Macros

`AE_ChannelSuites.h` provides typed access macros that validate the data type before returning a pointer:

```cpp
// Flat data access (pointer to start of all data)
PF_GET_CHANNEL_FLOAT_DATA(chunk, float_ptr)
PF_GET_CHANNEL_DOUBLE_DATA(chunk, double_ptr)
PF_GET_CHANNEL_LONG_DATA(chunk, long_ptr)
PF_GET_CHANNEL_SHORT_DATA(chunk, short_ptr)
PF_GET_CHANNEL_CHAR_DATA(chunk, char_ptr)
PF_GET_CHANNEL_U_BYTE_DATA(chunk, byte_ptr)
PF_GET_CHANNEL_U_SHORT_DATA(chunk, ushort_ptr)

// Row access (pointer to start of a specific row)
PF_GET_CHANNEL_ROW_FLOAT_DATA(chunk, row, float_ptr)
PF_GET_CHANNEL_ROW_DOUBLE_DATA(chunk, row, double_ptr)
// ... etc.

// Element access (pointer to a specific pixel's data)
PF_GET_CHANNEL_ROW_COL_FLOAT_DATA(chunk, row, col, float_ptr)
PF_GET_CHANNEL_ROW_COL_DOUBLE_DATA(chunk, row, col, double_ptr)
// ... etc.
```

All macros set the output pointer to NULL if the data type does not match or the row/col indices are out of bounds.

---

## Iterating All Available Channels

```cpp
A_long num_channels = 0;
ERR(ch_suite->PF_GetLayerChannelCount(
    in_data->effect_ref, 0, &num_channels));

for (A_long i = 0; i < num_channels && !err; i++) {
    PF_Boolean found = FALSE;
    PF_ChannelRef ref;
    PF_ChannelDesc desc;

    ERR(ch_suite->PF_GetLayerChannelIndexedRefAndDesc(
        in_data->effect_ref, 0, i, &found, &ref, &desc));

    if (found) {
        // desc.channel_type tells you what kind of channel
        // desc.data_type tells you the native data format
        // desc.dimension tells you values per pixel
        // desc.name gives you a human-readable name
    }
}
```

---

## Pitfalls

**PF_ChannelType values are FourCC integers, not strings.** They are `A_long` values created from four ASCII characters. Compare them directly: `if (desc.channel_type == PF_ChannelType_DEPTH)`.

**Channel data must be checked in after checkout.** Failing to call `PF_CheckinLayerChannel` will leak memory. Use ERR2 to ensure check-in happens even on error.

**Not all layers have auxiliary channels.** Only 3D-rendered layers and layers with certain effects provide channels beyond the standard ARGB. Always check the `found` boolean.

**The data type you request may not match the native format.** When calling `PF_CheckoutLayerChannel`, you specify the desired `PF_DataType`. AE will convert if possible, but check for errors.

**PF_PixLong byte order is platform-defined at the bit level but the macros abstract this.** Always use the PF_PixLong macros rather than casting to `A_u_char*` and assuming a specific byte order. The struct `PF_Pixel` with named fields is safer for direct component access.
