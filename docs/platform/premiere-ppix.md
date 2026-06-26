# PPix: Premiere Pro's Pixel Buffer System

This document covers Premiere Pro's pixel buffer system -- PPix -- referencing the SDK 26.0 headers (`PrSDKPPixSuite.h`, `PrSDKPPix2Suite.h`, `PrSDKPPixCreatorSuite.h`, `PrSDKPPixCreator2Suite.h`, `PrSDKPPixCacheSuite.h`, `PrSDKPixelFormat.h`). If you know After Effects' `PF_EffectWorld`, PPix is the Premiere equivalent, but with significant architectural differences.

## PPix vs PF_EffectWorld

| Property | AE PF_EffectWorld | Premiere PPix |
|---|---|---|
| Access | Direct struct pointer (`world->data`) | Handle-based (`PPixHand`), accessed through suites |
| Pixel data | `PF_Pixel8*` / `PF_Pixel16*` / `PF_PixelFloat*` | `char*` obtained from `GetPixels()` |
| Dimensions | `world->width`, `world->height` | `GetBounds()` returns `prRect` |
| Stride | `world->rowbytes` | `GetRowBytes()` (may be negative!) |
| Pixel format | Implicit from bit depth context | Explicit `PrPixelFormat` fourCC |
| Channel order | ARGB (8/16/32) | BGRA, ARGB, VUYA, YUV, and many more |
| Ownership | Host-managed | Reference counted; you must call `Dispose()` |
| Creation | `PF_NewWorld()` / `PF_NewWorldOfBitDepth()` | Creator suites |
| Caching | SmartFX / sequence data | `PrSDKPPixCacheSuite` |

The most important difference: **you never dereference a PPixHand directly**. All access goes through suite functions.

---

## PrSDKPPixSuite (Version 1)

The core suite for reading PPix properties:

```cpp
#define kPrSDKPPixSuite     "Premiere PPix Suite"
#define kPrSDKPPixSuiteVersion  1
```

### Suite Functions

#### Dispose

```cpp
prSuiteError (*Dispose)(PPixHand inPPixHand);
```

Frees the PPix. The handle is invalid after this call. You must dispose every PPix you own.

#### GetPixels

```cpp
prSuiteError (*GetPixels)(
    PPixHand            inPPixHand,
    PrPPixBufferAccess  inRequestedAccess,
    char**              outPixelAddress);
```

Returns a pointer to the raw pixel data. The access mode must match how the PPix was created:

| Access Mode | Value | Description |
|---|---|---|
| `PrPPixBufferAccess_ReadOnly` | 0 | Read pixels only |
| `PrPPixBufferAccess_WriteOnly` | 1 | Write pixels only |
| `PrPPixBufferAccess_ReadWrite` | 2 | Both read and write |

If the requested access is not supported (e.g., requesting write on a read-only PPix), `outPixelAddress` will be NULL.

#### GetBounds

```cpp
prSuiteError (*GetBounds)(
    PPixHand  inPPixHand,
    prRect*   inoutBoundingRect);
```

Returns the bounding rectangle. The `prRect` uses `{top, left, bottom, right}` ordering:
- Width = `right - left`
- Height = `bottom - top`

#### GetRowBytes

```cpp
prSuiteError (*GetRowBytes)(
    PPixHand        inPPixHand,
    csSDK_int32*    outRowBytes);
```

Returns the number of bytes to advance from one scanline to the next. **This value may be negative**, indicating a bottom-up pixel layout. Always use this value, never compute stride from `width * bytesPerPixel`.

#### GetPixelAspectRatio

```cpp
prSuiteError (*GetPixelAspectRatio)(
    PPixHand        inPPixHand,
    csSDK_uint32*   outPixelAspectRatioNumerator,
    csSDK_uint32*   outPixelAspectRatioDenominator);
```

#### GetPixelFormat

```cpp
prSuiteError (*GetPixelFormat)(
    PPixHand         inPPixHand,
    PrPixelFormat*   outPixelFormat);
```

Returns the fourCC pixel format. See the pixel format table below.

#### GetUniqueKey / GetUniqueKeySize

```cpp
prSuiteError (*GetUniqueKey)(
    PPixHand        inPPixHand,
    unsigned char*  inoutKeyBuffer,
    size_t          inKeyBufferSize);

prSuiteError (*GetUniqueKeySize)(
    size_t*  outKeyBufferSize);
```

Each PPix has a unique key that identifies its content. The key size is constant for the application session. Use this for custom caching logic.

#### GetRenderTime

```cpp
prSuiteError (*GetRenderTime)(
    PPixHand        inPPixHand,
    csSDK_int32*    outRenderMilliseconds);
```

Returns how long the frame took to render. Returns 0 if the frame came from cache.

---

## PrSDKPPix2Suite (Version 3)

Extended functions added in later versions:

```cpp
#define kPrSDKPPix2Suite         "Premiere PPix 2 Suite"
#define kPrSDKPPix2SuiteVersion  kPrSDKPPix2SuiteVersion3  // 3
```

### GetSize

```cpp
prSuiteError (*GetSize)(
    PPixHand  inPPixHand,
    size_t*   outSize);
```

Returns the total memory size of the PPix in bytes.

### GetYUV420PlanarBuffers

```cpp
prSuiteError (*GetYUV420PlanarBuffers)(
    PPixHand            inPPixHand,
    PrPPixBufferAccess  inRequestedAccess,
    char**              out_Y_PixelAddress,
    csSDK_uint32*       out_Y_RowBytes,
    char**              out_U_PixelAddress,
    csSDK_uint32*       out_U_RowBytes,
    char**              out_V_PixelAddress,
    csSDK_uint32*       out_V_RowBytes);
```

For planar YUV formats only. Returns separate pointers and rowbytes for Y, U, and V planes.

### GetOrigin

```cpp
prSuiteError (*GetOrigin)(
    PPixHand        inPPixHand,
    csSDK_int32*    outOriginX,
    csSDK_int32*    outOriginY);
```

Returns the pixel offset from the buffer start to the logical top-left corner of the image. Important when using `VideoSegmentRenderSuite`, which does not normalize to the output rectangle. Rules:
- Negative origin: never set; image should be centered in output
- Zero origin: explicitly set; top-left corner aligns with output top-left
- Positive origin: padding (e.g., blur soft edges)

### GetFieldOrder

```cpp
prSuiteError (*GetFieldOrder)(
    PPixHand      inPPixHand,
    prFieldType*  outFieldType);
```

Returns the actual field type of the PPix, which may differ from the clip or sequence field type. Edge cases where this can be progressive even on an interlaced timeline:
- Still image with no time-varying effects
- Quarter-resolution decompress of interlaced format
- Time remap hold frame (both fields are the same time)

---

## PrSDKPPixCreatorSuite (Version 1)

Basic PPix creation:

```cpp
#define kPrSDKPPixCreatorSuite         "Premiere PPix Creator Suite"
#define kPrSDKPPixCreatorSuiteVersion  1
```

### CreatePPix

```cpp
prSuiteError (*CreatePPix)(
    PPixHand*           outPPixHand,
    PrPPixBufferAccess  inRequestedAccess,    // ReadOnly not allowed
    PrPixelFormat       inPixelFormat,
    const prRect*       inBoundingRect);
```

Creates a new PPix with the specified format and dimensions. The bounding rect defines the size:

```cpp
prRect bounds = {0, 0, 1080, 1920};  // top, left, bottom, right
PPixHand newFrame = NULL;
creatorSuite->CreatePPix(&newFrame,
                          PrPPixBufferAccess_WriteOnly,
                          PrPixelFormat_BGRA_4444_8u,
                          &bounds);
```

### ClonePPix

```cpp
prSuiteError (*ClonePPix)(
    PPixHand            inPPixToClone,
    PPixHand*           outPPixHand,
    PrPPixBufferAccess  inRequestedAccess);  // Only ReadOnly currently
```

Clones a PPix. If both source and destination are read-only, this reference-counts rather than copying pixel data.

---

## PrSDKPPixCreator2Suite (Version 4)

Extended creation with more control over geometry and color management:

```cpp
#define kPrSDKPPixCreator2Suite         "Premiere PPix Creator 2 Suite"
#define kPrSDKPPixCreator2SuiteVersion  kPrSDKPPixCreator2SuiteVersion4  // 4
```

### CreatePPix (Extended)

```cpp
prSuiteError (*CreatePPix)(
    PPixHand*           outPPixHand,
    PrPPixBufferAccess  inRequestedAccess,
    PrPixelFormat       inPixelFormat,
    int                 inWidth,
    int                 inHeight,
    bool                inUseFields,
    int                 inFieldNumber,     // 0=first field, 1=second
    int                 inPARNumerator,
    int                 inPARDenominator);
```

### CreateRawPPix

```cpp
prSuiteError (*CreateRawPPix)(
    PPixHand*           outPPixHand,
    PrPPixBufferAccess  inRequestedAccess,
    int                 inSize,           // Size in bytes
    int                 inAlignment);     // Memory alignment
```

For opaque/compressed data that does not have standard pixel dimensions.

### CreateCustomPPix

```cpp
prSuiteError (*CreateCustomPPix)(
    PPixHand*           outPPixHand,
    PrPPixBufferAccess  inRequestedAccess,
    PrPixelFormat       inPixelFormat,
    int                 inWidth,
    int                 inHeight,
    int                 inPARNumerator,
    int                 inPARDenominator,
    int                 inDataBufferSize);  // Explicit buffer size
```

For custom pixel formats where the buffer size cannot be derived from width/height/format.

### CreateDiskAlignedPPix

```cpp
prSuiteError (*CreateDiskAlignedPPix)(
    PPixHand*       outPPixHand,
    PrPixelFormat   inPixelFormat,
    int             inWidth,
    int             inHeight,
    int             inPARNumerator,
    int             inPARDenominator,
    int             inMemoryAlignment,     // 0 = align to sector size
    int             inSectorSize,          // 0 = no padding
    int             inOffsetToPixelDataFromFirstSectorStart);
```

Creates a PPix optimized for direct disk I/O. Memory is aligned to sector boundaries for unbuffered reads.

### Color-Managed Variants (Pr 14.0+)

Three additional functions append a `PrSDKColorSpaceID` parameter:

- `CreateColorManagedPPix` -- same as `CreatePPix` + color space tag
- `CreateColorManagedCustomPPix` -- same as `CreateCustomPPix` + color space tag
- `CreateColorManagedDiskAlignedPPix` -- same as `CreateDiskAlignedPPix` + color space tag

```cpp
prSuiteError (*CreateColorManagedPPix)(
    PPixHand*           outPPixHand,
    PrPPixBufferAccess  inRequestedAccess,
    PrPixelFormat       inPixelFormat,
    int                 inWidth,
    int                 inHeight,
    bool                inUseFields,
    int                 inFieldNumber,
    int                 inPARNumerator,
    int                 inPARDenominator,
    PrSDKColorSpaceID   inColorSpaceID);
```

---

## PrSDKPPixCacheSuite (Version 8)

The cache suite allows importers to store and retrieve rendered frames:

```cpp
#define kPrSDKPPixCacheSuite         "Premiere PPix Cache Suite"
#define kPrSDKPPixCacheSuiteVersion  kPrSDKPPixCacheSuiteVersion8  // 8
```

### Basic Frame Cache

```cpp
// Add a frame to cache
prSuiteError (*AddFrameToCache)(
    csSDK_uint32   inImporterID,
    csSDK_int32    inStreamIndex,
    PPixHand       inPPixHand,
    csSDK_int32    inFrameNumber,
    void*          inPreferences,        // NULL if no prefs
    csSDK_int32    inPreferencesLength);

// Retrieve a frame from cache
prSuiteError (*GetFrameFromCache)(
    csSDK_uint32   inImporterID,
    csSDK_int32    inStreamIndex,
    csSDK_int32    inFrameNumber,
    csSDK_int32    inNumFormats,
    imFrameFormat* inFormats,
    PPixHand*      outPPixHand,
    void*          inPreferences,
    csSDK_int32    inPreferencesLength);
```

### Raw PPix Cache

For non-frame data (e.g., lookup tables, headers):

```cpp
prSuiteError (*AddRawPPixToCache)(
    csSDK_uint32   inImporterID,
    PPixHand       inPPixHand,
    csSDK_int32    inKey);

prSuiteError (*GetRawPPixFromCache)(
    csSDK_uint32   inImporterID,
    csSDK_int32    inKey,
    PPixHand*      outPPixHand);
```

### Named PPix Cache

GUID-based caching for cross-plugin sharing:

```cpp
prSuiteError (*AddNamedPPixToCache)(
    const prPluginID*  inPPixIdentifier,
    PPixHand           inPPix);

prSuiteError (*GetNamedPPixFromCache)(
    const prPluginID*  inPPixIdentifier,
    PPixHand*          outPPixHand);

prSuiteError (*RegisterDependencyOnNamedPPix)(
    const prPluginID*  inPPixIdentifier);

prSuiteError (*UnregisterDependencyOnNamedPPix)(
    const prPluginID*  inPPixIdentifier);

prSuiteError (*ExpireNamedPPixFromCache)(
    const prPluginID*  inPPixIdentifier);

prSuiteError (*ExpireAllPPixesFromCache)();
```

Dependencies prevent cache eviction. You **must** call `UnregisterDependencyOnNamedPPix` for every successful `RegisterDependencyOnNamedPPix` call, or you will leak memory.

### Color-Managed Cache (Pr 13.0+)

```cpp
prSuiteError (*AddFrameToCacheWithColorSpace)(
    csSDK_uint32       inImporterID,
    csSDK_int32        inStreamIndex,
    PPixHand           inPPixHand,
    csSDK_int32        inFrameNumber,
    PrRenderQuality    inQuality,
    void*              inPreferences,
    csSDK_int32        inPreferencesLength,
    PrSDKColorSpaceID* inPrColorSpaceID);

prSuiteError (*GetFrameFromCacheWithColorSpace)(
    csSDK_uint32       inImporterID,
    csSDK_int32        inStreamIndex,
    csSDK_int32        inFrameNumber,
    csSDK_int32        inNumFormats,
    imFrameFormat*     inFormats,
    PPixHand*          outPPixHand,
    PrRenderQuality    inQuality,
    void*              inPreferences,
    csSDK_int32        inPreferencesLength,
    PrSDKColorSpaceID* inPrColorSpaceID);
```

---

## Pixel Formats

Premiere supports a much wider range of pixel formats than After Effects. All are defined in `PrSDKPixelFormat.h` as fourCC values.

### Packed RGBA/BGRA Formats

| Format | Bit Depth | Channel Order | Notes |
|---|---|---|---|
| `PrPixelFormat_BGRA_4444_8u` | 8-bit | BGRA | Windows standard, most common |
| `PrPixelFormat_ARGB_4444_8u` | 8-bit | ARGB | AE native format |
| `PrPixelFormat_BGRA_4444_16u` | 16-bit int | BGRA | |
| `PrPixelFormat_ARGB_4444_16u` | 16-bit int | ARGB | |
| `PrPixelFormat_BGRA_4444_32f` | 32-bit float | BGRA | |
| `PrPixelFormat_ARGB_4444_32f` | 32-bit float | ARGB | |

### Premultiplied Variants

| Format | Bit Depth | Notes |
|---|---|---|
| `PrPixelFormat_BGRP_4444_8u` | 8-bit | Premultiplied BGRA |
| `PrPixelFormat_PRGB_4444_8u` | 8-bit | Premultiplied ARGB |
| `PrPixelFormat_BGRP_4444_16u` | 16-bit | |
| `PrPixelFormat_PRGB_4444_16u` | 16-bit | |
| `PrPixelFormat_BGRP_4444_32f` | 32-bit float | |
| `PrPixelFormat_PRGB_4444_32f` | 32-bit float | |

### No-Alpha Variants

| Format | Bit Depth | Notes |
|---|---|---|
| `PrPixelFormat_BGRX_4444_8u` | 8-bit | Alpha channel ignored |
| `PrPixelFormat_XRGB_4444_8u` | 8-bit | |
| `PrPixelFormat_BGRX_4444_16u` | 16-bit | |
| `PrPixelFormat_BGRX_4444_32f` | 32-bit float | |

### Linear Light Variants (32f only)

| Format | Notes |
|---|---|
| `PrPixelFormat_BGRA_4444_32f_Linear` | Linear BGRA |
| `PrPixelFormat_BGRP_4444_32f_Linear` | Linear premultiplied BGRA |
| `PrPixelFormat_BGRX_4444_32f_Linear` | Linear BGRX |
| `PrPixelFormat_ARGB_4444_32f_Linear` | Linear ARGB |
| `PrPixelFormat_PRGB_4444_32f_Linear` | Linear premultiplied ARGB |
| `PrPixelFormat_XRGB_4444_32f_Linear` | Linear XRGB |

### YUV Packed Formats

| Format | Subsampling | Bit Depth | Color Space |
|---|---|---|---|
| `PrPixelFormat_VUYA_4444_8u` | 4:4:4 | 8-bit | 601 |
| `PrPixelFormat_VUYA_4444_8u_709` | 4:4:4 | 8-bit | 709 |
| `PrPixelFormat_YUYV_422_8u_601` | 4:2:2 | 8-bit | 601 |
| `PrPixelFormat_YUYV_422_8u_709` | 4:2:2 | 8-bit | 709 |
| `PrPixelFormat_UYVY_422_8u_601` | 4:2:2 | 8-bit | 601 |
| `PrPixelFormat_V210_422_10u_601` | 4:2:2 | 10-bit packed | 601 |
| `PrPixelFormat_V210_422_10u_709` | 4:2:2 | 10-bit packed | 709 |

### YUV Planar Formats

Accessed through `PrSDKPPix2Suite::GetYUV420PlanarBuffers()`:

| Format | Picture Type | Bit Depth | Color Space |
|---|---|---|---|
| `PrPixelFormat_YUV_420_MPEG2_FRAME_PICTURE_PLANAR_8u_601` | Frame | 8-bit | 601 |
| `PrPixelFormat_YUV_420_MPEG2_FIELD_PICTURE_PLANAR_8u_601` | Field | 8-bit | 601 |
| `PrPixelFormat_YUV_420_MPEG2_FRAME_PICTURE_PLANAR_8u_709` | Frame | 8-bit | 709 |
| `PrPixelFormat_YUV_420_MPEG4_FRAME_PICTURE_PLANAR_8u_709` | Frame | 8-bit | 709 |

Plus many more NV12, P010, biplanar, and HDR variants.

### HDR Formats

| Format | Description |
|---|---|
| `PrPixelFormat_RGB_444_12u_PQ_709` | 12-bit PQ curve, Rec.709 |
| `PrPixelFormat_RGB_444_12u_PQ_P3` | 12-bit PQ curve, DCI-P3 |
| `PrPixelFormat_RGB_444_12u_PQ_2020` | 12-bit PQ curve, Rec.2020 |
| `PrPixelFormat_RGB_444_10u_HLG` | 10-bit HLG, Rec.2020 |
| `PrPixelFormat_RGB_444_12u_HLG` | 12-bit HLG, Rec.2020 |

### Special Formats

| Format | Description |
|---|---|
| `PrPixelFormat_Raw` | Opaque data blob, no geometry |
| `PrPixelFormat_Invalid` | Error/initialization sentinel |
| `PrPixelFormat_Any` | Accept any format (value 0) |

### Custom Pixel Formats

Third-party plugins can define custom formats:

```cpp
#define MAKE_THIRD_PARTY_CUSTOM_PIXEL_FORMAT_FOURCC(ch1, ch2, ch3) \
    (static_cast<PrPixelFormat>(MAKE_PIXEL_FORMAT_FOURCC('_', ch1, ch2, ch3)))

// Example:
const PrPixelFormat MyCustomFormat =
    MAKE_THIRD_PARTY_CUSTOM_PIXEL_FORMAT_FOURCC('M', 'y', 'F');
```

Custom formats always start with `_` as the first byte. Adobe internal formats start with `@`.

---

## Complete Workflow: Receiving, Processing, and Returning PPix Buffers

### In an Importer (imGetSourceVideo)

```cpp
// 1. Create a PPix for output
PPixHand outFrame = NULL;
ppixCreator2Suite->CreatePPix(
    &outFrame,
    PrPPixBufferAccess_WriteOnly,
    PrPixelFormat_BGRA_4444_8u,
    width, height,
    false,    // not fields
    0,        // field number
    1, 1);    // square pixels

// 2. Get writable pixel pointer
char* pixels = NULL;
ppixSuite->GetPixels(outFrame, PrPPixBufferAccess_WriteOnly, &pixels);

csSDK_int32 rowBytes = 0;
ppixSuite->GetRowBytes(outFrame, &rowBytes);

// 3. Fill pixels (respecting rowbytes)
for (int y = 0; y < height; y++)
{
    uint32_t* scanline = reinterpret_cast<uint32_t*>(pixels + y * rowBytes);
    for (int x = 0; x < width; x++)
    {
        // BGRA order for PrPixelFormat_BGRA_4444_8u
        uint8_t B = /* ... */;
        uint8_t G = /* ... */;
        uint8_t R = /* ... */;
        uint8_t A = 255;
        scanline[x] = (A << 24) | (R << 16) | (G << 8) | B;
    }
}

// 4. Optionally cache the frame
ppixCacheSuite->AddFrameToCache(
    importerID, 0, outFrame, frameNumber, NULL, 0);

// 5. Return to host
*srcVideoRec->outFrame = outFrame;
```

### In an Effect (Processing Input PPix)

```cpp
// 1. Receive input PPix from host (via EffectRecord or VideoRecord)
PPixHand inputFrame = videoRec->source;

// 2. Query properties
PrPixelFormat format;
ppixSuite->GetPixelFormat(inputFrame, &format);

prRect bounds;
ppixSuite->GetBounds(inputFrame, &bounds);
int width = bounds.right - bounds.left;
int height = bounds.bottom - bounds.top;

csSDK_int32 inputRowBytes;
ppixSuite->GetRowBytes(inputFrame, &inputRowBytes);

// 3. Get read-only pixel access
char* inputPixels = NULL;
ppixSuite->GetPixels(inputFrame, PrPPixBufferAccess_ReadOnly, &inputPixels);

// 4. Get writable access to destination
PPixHand outputFrame = videoRec->destination;
char* outputPixels = NULL;
ppixSuite->GetPixels(outputFrame, PrPPixBufferAccess_WriteOnly, &outputPixels);

csSDK_int32 outputRowBytes;
ppixSuite->GetRowBytes(outputFrame, &outputRowBytes);

// 5. Process
for (int y = 0; y < height; y++)
{
    const uint8_t* srcRow = reinterpret_cast<const uint8_t*>(
        inputPixels + y * inputRowBytes);
    uint8_t* dstRow = reinterpret_cast<uint8_t*>(
        outputPixels + y * outputRowBytes);

    ProcessScanline(srcRow, dstRow, width, format);
}
```

### Handling Negative Rowbytes

```cpp
csSDK_int32 rowBytes;
ppixSuite->GetRowBytes(frame, &rowBytes);

char* pixels;
ppixSuite->GetPixels(frame, PrPPixBufferAccess_ReadOnly, &pixels);

// If rowBytes is negative, the first pixel pointer is actually
// the LAST scanline. Iterate correctly:
if (rowBytes < 0)
{
    // pixels points to the last row. First row is at:
    // pixels + (height - 1) * rowBytes  (rowBytes is negative, so this goes up)
}

// Safest approach: always use rowBytes as-is for pointer arithmetic
for (int y = 0; y < height; y++)
{
    char* row = pixels + (ptrdiff_t)y * rowBytes;
    // Process row...
}
```

---

## BGRA vs ARGB Channel Order

The single most common source of color-swap bugs when porting from AE to Premiere.

**After Effects** uses ARGB channel order everywhere:
```
byte 0: Alpha
byte 1: Red
byte 2: Green
byte 3: Blue
```

**Premiere's native format** (`PrPixelFormat_BGRA_4444_8u`) uses BGRA:
```
byte 0: Blue
byte 1: Green
byte 2: Red
byte 3: Alpha
```

### Strategy for cross-host plugins

If your plugin must work in both AE and Premiere:

1. **Declare support for both formats** via `imGetIndPixelFormat` or `esGetPixelFormatsSupported`
2. **Check the actual format at render time** using `GetPixelFormat()`
3. **Use format-specific pixel access**:

```cpp
PrPixelFormat format;
ppixSuite->GetPixelFormat(frame, &format);

if (format == PrPixelFormat_BGRA_4444_8u)
{
    struct BGRAPixel { uint8_t b, g, r, a; };
    BGRAPixel* pixel = reinterpret_cast<BGRAPixel*>(row + x * 4);
}
else if (format == PrPixelFormat_ARGB_4444_8u)
{
    struct ARGBPixel { uint8_t a, r, g, b; };
    ARGBPixel* pixel = reinterpret_cast<ARGBPixel*>(row + x * 4);
}
```

---

## Pitfalls

1. **Never dereference PPixHand directly.** Always use the suite functions. The handle is opaque.

2. **Always Dispose PPix handles you own.** If you created it (via creator suites) or received it from cache, you must dispose it. If you returned it to the host (e.g., `*outFrame = myPPix`), the host takes ownership.

3. **Rowbytes includes padding.** The stride may be larger than `width * bytesPerPixel` due to alignment. Some Premiere pixel buffers are 16-byte or 32-byte aligned per row.

4. **Negative rowbytes is valid.** This means bottom-up storage. Your scanline loop must handle this correctly.

5. **Cache dependencies are reference-counted.** Every `RegisterDependencyOnNamedPPix` must have a matching `UnregisterDependencyOnNamedPPix`, or the PPix will never be evicted from cache, causing a memory leak.

6. **GetPixels can return NULL.** If you request write access on a read-only PPix, or if the PPix is in a GPU-only format, the pixel pointer will be NULL. Always check.

7. **Planar formats need PPix2Suite.** For YUV 4:2:0 planar formats, `GetPixels()` returns a pointer to the Y plane only. Use `GetYUV420PlanarBuffers()` to access all three planes.

8. **PPix origin matters for compositing.** When receiving PPix from the `VideoSegmentRenderSuite`, the origin may not be (0,0). Check `GetOrigin()` from PPix2Suite and offset accordingly.

9. **Field-based PPix has half height.** If `inUseFields` was true when creating the PPix, the height is the field height (half the frame height). Check `GetFieldOrder()` to know which field you have.

10. **Color-managed PPix creation requires version 4 of PPixCreator2Suite.** If you need to tag frames with a color space, you must acquire `kPrSDKPPixCreator2SuiteVersion4` and use the `CreateColorManaged*` variants.
