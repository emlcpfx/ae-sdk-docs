# AEIO Frame Caching and Memory Management

This document covers how After Effects manages frame caching for AEIO-imported footage, the mechanics of the `AEIO_DrawSparseFrame` callback, memory allocation strategies, and performance considerations for custom file format plugins.

## How AE Caches Imported Frames

After Effects maintains a global frame cache (often called the "image cache" or "RAM cache") that stores rendered frames across all footage items, compositions, and layers. When an AEIO plugin delivers a frame via `AEIO_DrawSparseFrame`, that frame enters this cache and remains available until the cache is purged.

### Cache Lookup Flow

When AE needs a frame from AEIO-imported footage, the following sequence occurs:

```
AE needs frame at time T
  -> Check global frame cache for this footage item + time + quality + scale
     -> Cache HIT: use cached frame, no AEIO callback
     -> Cache MISS:
        -> Allocate a PF_EffectWorld of the appropriate size
        -> Call AEIO_DrawSparseFrame with the world and parameters
        -> Store the returned frame in the cache
        -> Use the frame
```

The cache key includes:
- The footage item identity
- The requested time
- The quality level (`AEIO_Qual_LOW` or `AEIO_Qual_HIGH`)
- The scale factor (`AEIO_RationalScale`)
- The required region (if sparse rendering is active)

This means the same frame at different quality levels or scales will be cached as separate entries.

### When DrawSparseFrame Is Called

`AEIO_DrawSparseFrame` is called in these situations:

- **First access**: The frame has never been rendered before.
- **Cache purge**: The frame was previously cached but has been evicted due to memory pressure.
- **Different parameters**: The frame is needed at a different quality, scale, or region than what is cached.
- **Footage reload**: The user invoked "Reload Footage" or the file changed on disk and `AEIO_SynchInSpec` invalidated cached frames.
- **RAM preview / render**: During continuous playback or final render, frames not yet in cache trigger calls.
- **Thumbnail generation**: The Project panel, composition mini-flowchart, and footage interpretation dialog all request low-quality, scaled-down frames.

Frames are **not** re-requested when:
- The user scrubs to a time whose frame is already cached at the needed quality/scale.
- A composition re-renders but the source footage frame has not changed (upstream cache hit).
- The user switches between compositions that reference the same cached footage frame.

### Cache Eviction

AE evicts cached frames under memory pressure using a least-recently-used (LRU) strategy. The user can also manually purge caches via Edit > Purge. When a frame is evicted, the next request for that frame will trigger another `AEIO_DrawSparseFrame` call.

Your AEIO has no direct control over when frames are evicted. You cannot pin frames in cache or request that specific frames be retained.

## The DrawSparseFrame Parameter Block

```cpp
typedef struct {
    AEIO_Quality        qual;               // LOW or HIGH
    AEIO_RationalScale  rs;                 // scale factor (x/y rationals)
    A_Time              tr;                 // requested time
    A_Time              duration;           // frame duration
    A_LRect             required_region;    // sub-region needed (empty = all)
    AEIO_InterruptFuncs inter;              // abort/progress function pointers
} AEIO_DrawSparseFramePB;
```

### Quality Levels

| Value | Meaning | Typical Use |
|-------|---------|-------------|
| `AEIO_Qual_LOW` | Draft quality | Thumbnails, scrubbing, draft previews |
| `AEIO_Qual_HIGH` | Full quality | Final render, RAM preview at full quality |

When `qual` is `AEIO_Qual_LOW`, you can return a lower-fidelity decode to improve responsiveness. For example, you might skip expensive deblocking filters or use a faster color space conversion.

### Scale Factor

The `rs` field contains rational scale values for both X and Y:

```cpp
typedef struct {
    A_Ratio x;  // .num / .den
    A_Ratio y;
} AEIO_RationalScale;
```

When the user views footage at 50% in the Composition panel, `rs.x` might be `{1, 2}` (one-half). The provided `PF_EffectWorld` will already be sized to match this scale. You should decode at this reduced resolution if your codec supports it.

### Required Region

The `required_region` field is an `A_LRect` specifying the sub-rectangle of the frame that AE actually needs. If this rectangle is empty (all zeros), the entire frame is needed. If it is non-empty, you only need to fill that portion of the world.

> **Note**: Many AEIO implementations ignore the required region and always fill the entire world. This is acceptable but suboptimal for large frames where only a small region is visible.

### Interrupt Functions

The `inter` field provides abort and progress callbacks:

```cpp
typedef struct {
    void                *refcon;
    AEIO_AbortProc      abort0;      // returns non-zero if user cancelled
    AEIO_ProgressProc   progress0;   // report decode progress
} AEIO_InterruptFuncs;
```

Check these periodically during long decodes:

```cpp
if (inter.abort0) {
    A_Err abort_err = inter.abort0(inter.refcon);
    if (abort_err) {
        return abort_err;  // user cancelled
    }
}

if (inter.progress0) {
    inter.progress0(inter.refcon, current_row, total_rows);
}
```

## PF_EffectWorld Allocation for Import Frames

AE allocates the `PF_EffectWorld` before calling `DrawSparseFrame`. The world's pixel format depends on:

1. **The depth you declared** via `AEGP_SetInSpecDepth` in `InitInSpecFromFile`.
2. **The project bit depth setting** (8, 16, or 32 bpc).
3. **Module flags**: `AEIO_MFlag_CAN_DRAW_DEEP` enables 16-bpc worlds, `AEIO_MFlag2_CAN_DRAW_FLOAT` enables 32-bpc float worlds.

The relationship:

| Module declares | Project at 8bpc | Project at 16bpc | Project at 32bpc |
|----------------|-----------------|-------------------|-------------------|
| Neither DEEP nor FLOAT flag | 8-bit world | 8-bit world | 8-bit world |
| `CAN_DRAW_DEEP` only | 8-bit world | 16-bit world | 16-bit world |
| `CAN_DRAW_FLOAT` only | 8-bit world | 8-bit world | 32-bit float world |
| Both flags | 8-bit world | 16-bit world | 32-bit float world |

### Determining the World Format

Check `worldP->world_flags` to determine the pixel format:

```cpp
if (worldP->world_flags & PF_WorldFlag_DEEP) {
    // 16-bit per channel (PF_Pixel16 / PF_PixelFloat for DEEP worlds)
    // Stride: worldP->rowbytes
    PF_Pixel16 *row = (PF_Pixel16 *)((char *)worldP->data + y * worldP->rowbytes);
} else {
    // Check for float (32-bit)
    // In practice, use PF_GET_PIXEL_DATA macros or check world dimensions
    PF_Pixel8 *row = (PF_Pixel8 *)((char *)worldP->data + y * worldP->rowbytes);
}
```

### Memory Layout

The `PF_EffectWorld` has these critical fields:

```cpp
worldP->data      // pointer to first pixel
worldP->rowbytes  // bytes per row (includes padding!)
worldP->width     // width in pixels
worldP->height    // height in pixels
```

**Row bytes will almost always be larger than `width * bytes_per_pixel`.** AE aligns rows for SIMD performance. Always use `rowbytes` for row-to-row pointer arithmetic.

```cpp
// CORRECT: use rowbytes for stride
for (A_long y = 0; y < worldP->height; y++) {
    PF_Pixel8 *pixelP = (PF_Pixel8 *)((char *)worldP->data + y * worldP->rowbytes);
    for (A_long x = 0; x < worldP->width; x++) {
        pixelP[x].alpha = 255;
        pixelP[x].red   = src_red;
        pixelP[x].green = src_green;
        pixelP[x].blue  = src_blue;
    }
}

// WRONG: this will corrupt memory or produce skewed output
// PF_Pixel8 *pixelP = (PF_Pixel8 *)worldP->data + y * worldP->width;
```

## Frame Disposal

Your AEIO does **not** dispose of the `PF_EffectWorld` passed to `DrawSparseFrame`. AE owns this memory. You simply fill it with pixel data and return. AE will dispose of the world when:

- The cached frame is evicted under memory pressure.
- The footage item is removed from the project.
- The user purges the cache.
- The application shuts down.

If you allocate temporary buffers during decode (e.g., for an intermediate decompression step), you are responsible for freeing those. Use RAII patterns or explicit cleanup before returning from `DrawSparseFrame`.

## Sequential vs Random Access Patterns

### Sequential Access

During RAM preview, render queue output, or continuous playback, AE requests frames in order (frame 0, 1, 2, ...). This is the most common pattern and the most codec-friendly, as most video codecs are optimized for sequential decoding.

**Optimization**: Keep your file handle and decoder state open between calls. Store them in the AEIO refcon or in a structure attached to the InSpec options handle. This avoids the overhead of reopening and re-seeking the file for every frame.

```cpp
// Store decoder state in refcon
struct MyIOData {
    FILE            *fileP;
    DecoderContext  *decoder;
    A_long          last_frame_decoded;
    // ...
};
```

### Random Access

During scrubbing, composition editing, or when AE needs non-sequential frames, `DrawSparseFrame` may be called with arbitrary time values. Your AEIO must handle this gracefully.

**For interframe codecs (e.g., H.264, H.265)**: Random access requires seeking to the nearest keyframe and decoding forward to the requested frame. This can be expensive. Strategies:

1. **Maintain a decode cache**: Keep recently decoded reference frames in memory.
2. **Use a seek table**: Build a keyframe index on `InitInSpecFromFile` for fast seeking.
3. **Return AEIO_Qual_LOW quickly**: For low-quality requests during scrubbing, consider returning the nearest keyframe directly rather than decoding all intermediate frames.

**For intraframe codecs (e.g., MJPEG, ProRes, DPX sequences)**: Random access is cheap since each frame is independently decodable. Simply seek to the requested frame and decode it.

### Thumbnail Requests

AE frequently requests small, low-quality frames for the Project panel and composition thumbnails. These arrive as `DrawSparseFrame` calls with:
- `qual = AEIO_Qual_LOW`
- A scaled `rs` value
- A `PF_EffectWorld` that may be significantly smaller than native resolution

> **Pitfall**: Do not assume the world dimensions match the source resolution. A thumbnail world might be 80x45 pixels for a 1920x1080 source. Hardcoding source dimensions in your pixel loop will crash.

## Memory Pressure Handling

### The Idle Callback

If your AEIO has internal caches (decode buffers, file caches, etc.), implement `AEIO_Idle` to respond to memory pressure:

```cpp
A_Err MyIdle(
    AEIO_BasicData        *basic_dataP,
    AEIO_ModuleSignature  sig,
    AEIO_IdleFlags        *idle_flags0)
{
    // Check if AE purged memory
    if (*idle_flags0 & AEIO_IdleFlag_PURGED_MEM) {
        // AE just purged its cache - release your internal caches too
        FreeDecoderCache();
    }

    // You can set flags to indicate you freed memory
    // *idle_flags0 |= AEIO_IdleFlag_ADD_YOUR_OWN;

    return A_Err_NONE;
}
```

### CloseSourceFiles

AE calls `AEIO_CloseSourceFiles` when it needs to reclaim file handles or when the user invokes a file operation that requires exclusive access (such as "Replace Footage" or moving the file). Close all file handles associated with the InSpec, but be prepared to reopen them on the next `DrawSparseFrame` call.

```cpp
A_Err MyCloseSourceFiles(
    AEIO_BasicData *basic_dataP,
    AEIO_InSpecH   seqH)
{
    // Close file handles but keep decoder state
    MyIOData *ioData = GetIODataFromSpec(seqH);
    if (ioData && ioData->fileP) {
        fclose(ioData->fileP);
        ioData->fileP = NULL;
    }
    return A_Err_NONE;
}
```

## Performance Considerations

### Large Media Files

For footage with large frame sizes (4K, 8K, EXR sequences):

1. **Decode directly into the world buffer.** Avoid allocating a separate buffer and then copying. If your codec's output format matches AE's pixel layout (ARGB, 8/16/32 bpc), write directly to `worldP->data` respecting `rowbytes`.

2. **Use memory-mapped I/O for large files.** For image sequences with huge individual frames (e.g., 16-bit EXR at 8K), memory-mapping the file can be faster than buffered reads.

3. **Support the scale factor.** When `rs` indicates a downscale, decode at reduced resolution if your codec supports it. For JPEG 2000, this means decoding fewer resolution levels. For DPX sequences, you might subsample during the read.

### Concurrent Access

AE may call `DrawSparseFrame` from multiple threads during multi-frame rendering (MFR). If your decoder or file I/O is not thread-safe, you must synchronize access:

```cpp
// Thread-safe approach: use a mutex
std::mutex g_decoder_mutex;

A_Err MyDrawSparseFrame(/* ... */) {
    std::lock_guard<std::mutex> lock(g_decoder_mutex);
    // Decode frame...
    return A_Err_NONE;
}
```

A better approach for MFR compatibility is to maintain per-thread decoder instances, avoiding contention entirely.

### Minimizing Decode Overhead

| Strategy | Benefit |
|----------|---------|
| Keep file handles open | Avoids open/seek overhead per frame |
| Maintain a keyframe index | Fast random access for interframe codecs |
| Cache recently decoded reference frames | Reduces re-decoding for GOP-based codecs |
| Decode at requested scale | Less memory and processing for thumbnails |
| Use hardware decode if available | Significant speedup for supported codecs |
| Return `AEIO_Qual_LOW` frames faster | Better interactivity during scrubbing |

### Memory Budget

AE does not tell your AEIO how much memory is available. You need to manage your own internal memory budget. Guidelines:

- Keep internal decode caches modest (a few frames at most).
- Release caches when `AEIO_Idle` indicates memory pressure.
- Do not hold large allocations across `DrawSparseFrame` calls unless necessary for codec state.
- Use the AEGP memory suite (`AEGP_NewMemHandle` / `AEGP_FreeMemHandle`) for allocations that AE should track, or standard allocators for private memory.

## Auxiliary Channels

For formats with extra per-pixel data (depth, object ID, normals), implement the auxiliary channel callbacks:

```cpp
AEIO_GetNumAuxChannels   -> report how many extra channels exist
AEIO_GetAuxChannelDesc   -> describe each channel (name, type, dimensions)
AEIO_DrawAuxChannel      -> fill a PF_ChannelChunk with channel data
AEIO_FreeAuxChannel      -> free the channel chunk data
```

Set `AEIO_MFlag_HAS_AUX_DATA` in your module flags to enable this. Auxiliary channel data follows the same caching principles as primary frame data.

## Summary of Key Points

- AE allocates and owns the `PF_EffectWorld`; you just fill it with pixels.
- Frames are cached by (footage item, time, quality, scale, region).
- `DrawSparseFrame` is only called on cache misses.
- Always use `worldP->rowbytes` for stride, never `width * pixel_size`.
- The world dimensions may be scaled down from source for thumbnails and previews.
- Implement `AEIO_Idle` and `AEIO_CloseSourceFiles` to handle memory pressure gracefully.
- Keep file handles and decoder state alive between calls for performance.
- For MFR compatibility, ensure thread-safe decoder access.
