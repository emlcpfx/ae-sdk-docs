# AE_Hook.h: Plugin Hooks and Blit Operations

## Overview

`AE_Hook.h` defines a specialized plugin interface for hooking into After Effects' display pipeline. Hook plugins are **not** standard effect or AEGP plugins. They are registered as `'AEgp'` type plugins and provide callback functions that AE invokes when it needs to display frames, handle cursor events, or manage the plugin lifecycle. The primary use case is intercepting the final pixel blit to the composition or layer viewer window.

Hook plugins are rare in third-party development. They are primarily used by internal Adobe code and by specialized external display pipelines -- for example, custom hardware preview monitors, external recording devices, or alternate rendering backends. Understanding this header is valuable for anyone building custom viewer integrations or debugging display pipeline issues.

---

## Plugin Registration

```c
#define AE_HOOK_PLUGIN_TYPE     'AEgp'
#define AE_HOOK_MAJOR_VERSION   3
#define AE_HOOK_MINOR_VERSION   0
```

A hook plugin identifies itself with the `'AEgp'` plugin type. The current API is version 3.0. The plugin's entry point must conform to the `AE_HookPluginEntryFunc` signature.

### Entry Point

```c
typedef PF_Err (*AE_HookPluginEntryFunc)(
    A_long          major_version,   // >> AE_HOOK_MAJOR_VERSION
    A_long          minor_version,   // >> AE_HOOK_MINOR_VERSION
    AE_FileSpecH   file_specH,      // >> Plugin file specification
    AE_FileSpecH   res_specH,       // >> Resource file specification
    AE_Hooks        *hooksP);        // << Fill in your callbacks
```

When AE loads a hook plugin, it calls this entry point. The plugin receives the API version and file handles, and must fill in the `AE_Hooks` structure with its callback function pointers.

**Example entry point:**

```c
PF_Err MyHookEntry(
    A_long          major_version,
    A_long          minor_version,
    AE_FileSpecH   file_specH,
    AE_FileSpecH   res_specH,
    AE_Hooks        *hooksP)
{
    if (major_version > AE_HOOK_MAJOR_VERSION) {
        return PF_Err_UNRECOGNIZED_PARAM_TYPE;
    }

    hooksP->hook_refconPV     = MyAllocateContext();
    hooksP->death_hook_func   = MyDeathHook;
    hooksP->version_hook_func = MyVersionHook;
    hooksP->blit_hook_func    = MyBlitHook;
    hooksP->cursor_hook_func  = MyCursorHook;

    return PF_Err_NONE;
}
```

---

## The AE_Hooks Structure

The `AE_Hooks` structure is the central registration mechanism. It must match the internal `NIM_Hooks` layout (as noted in the header comment).

```c
typedef struct {
    void                *hook_refconPV;         // Your private data pointer
    void                *reservedAPV[8];        // Reserved -- do not use
    AE_DeathHook        death_hook_func;        // Called when plugin is unloaded
    AE_VersionHook      version_hook_func;      // Called to query plugin version
    struct SPBasicSuite  *pica_basicP;          // SP Basic suite (filled by host)
    AE_BlitHook         blit_hook_func;         // Called to display a frame
    AE_CursorHook       cursor_hook_func;       // Called for cursor updates
} AE_Hooks;
```

| Field | Direction | Purpose |
|-------|-----------|---------|
| `hook_refconPV` | Plugin sets | Opaque pointer passed back to all hook callbacks |
| `reservedAPV[8]` | -- | Reserved array, must not be written to |
| `death_hook_func` | Plugin sets | Cleanup callback |
| `version_hook_func` | Plugin sets | Version query callback |
| `pica_basicP` | Host sets | `SPBasicSuite` pointer for suite acquisition |
| `blit_hook_func` | Plugin sets | The main display callback |
| `cursor_hook_func` | Plugin sets | Cursor event callback |

Note that `pica_basicP` is provided by the host after the entry point returns. Your hook callbacks can use it to acquire suites, just as any other plugin would.

---

## Hook Callback Types

### Death Hook

```c
typedef void (*AE_DeathHook)(void *hook_refconPV);
```

Called when the hook plugin is being unloaded. Use this to free any resources allocated during the entry point or during operation. The refcon pointer you stored in `AE_Hooks::hook_refconPV` is passed back.

```c
void MyDeathHook(void *hook_refconPV) {
    MyContext *ctx = (MyContext *)hook_refconPV;
    if (ctx) {
        // Release GPU resources, close handles, etc.
        free(ctx);
    }
}
```

### Version Hook

```c
typedef void (*AE_VersionHook)(
    void        *hook_refconPV,   // >> Your refcon
    A_u_long    *versionPV);      // << Return your version
```

Called by AE to query the plugin's version number. Write your version into the output parameter.

```c
void MyVersionHook(void *hook_refconPV, A_u_long *versionPV) {
    *versionPV = 0x00010000; // Version 1.0.0.0
}
```

### Blit Hook (Primary Display Callback)

```c
typedef PF_Err (*AE_BlitHook)(
    void                        *hook_refconPV,   // >> Your refcon
    const AE_PixBuffer          *pix_bufP0,       // >> Pixel data (NULL = blank)
    const AE_ViewCoordinates    *viewP,           // >> View geometry
    AE_BlitReceipt              receipt,           // >> Async receipt
    AE_BlitCompleteFunc         complete_func0,    // >> Async completion callback
    AE_BlitInFlags              in_flags,          // >> Input flags
    AE_BlitOutFlags             *out_flags);       // << Output flags
```

This is the core of the hook system. AE calls this function whenever it needs to display a frame. The plugin receives the pixel buffer and view coordinates, and is responsible for presenting the image (e.g., sending it to a hardware preview monitor or custom renderer).

**Parameters:**

- `pix_bufP0` -- The pixel buffer to display. If `NULL`, the plugin should display a blank/black frame.
- `viewP` -- Geometry information about the view (frame size, origin, visible region).
- `receipt` -- An opaque receipt for asynchronous blit operations.
- `complete_func0` -- Completion callback for asynchronous blits. May be `NULL`.
- `in_flags` -- Flags describing the context of the blit.
- `out_flags` -- Output flags the plugin sets to communicate back to AE.

### Blit Complete Function

```c
typedef void (*AE_BlitCompleteFunc)(
    AE_BlitReceipt  receipt,
    PF_Err          err);      // Non-zero if error during async operation
```

Used with asynchronous blit operations. When the plugin sets `AE_BlitOutFlag_ASYNCHRONOUS` in the output flags, it must call this function when the blit completes.

### Cursor Hook

```c
typedef void (*AE_CursorHook)(
    void                    *hook_refconPV,   // >> Your refcon
    const AE_CursorInfo     *cursorP);        // >> Cursor information
```

Called when cursor events occur in the view. The `AE_CursorInfo` type is opaque (noted in the header as "opaque until I can decide what this means"), suggesting this interface was not fully finalized.

---

## Pixel Buffer Structure

The `AE_PixBuffer` structure describes a rectangular pixel buffer passed to the blit hook:

```c
typedef struct {
    A_long          widthL;         // Width in pixels
    A_long          heightL;        // Height in pixels
    A_long          depthL;         // Total bits per pixel: 32, 64, or 128
    AE_PixFormat    pix_format;     // ARGB on Mac, BGRA on Windows

    A_long          row_bytesL;     // Bytes per row (includes padding)
    A_long          chan_bytesL;    // Bytes per channel: 4, 8, or 16
    A_long          plane_bytesL;  // Bytes per sample: 1, 2, or 4 (float)

    void            *pixelsPV;     // Pointer to pixel data
} AE_PixBuffer;
```

### Bit Depth Correspondence

| Depth (`depthL`) | Channel Bytes (`chan_bytesL`) | Plane Bytes (`plane_bytesL`) | Format |
|---|---|---|---|
| 32 | 4 | 1 | 8-bit per channel (4 channels x 8 bits = 32) |
| 64 | 8 | 2 | 16-bit per channel (4 channels x 16 bits = 64) |
| 128 | 16 | 4 | 32-bit float per channel (4 channels x 32 bits = 128) |

### Pixel Format

```c
enum {
    AE_PixFormat_NONE = -1,    // Sentinel, meaningless
    AE_PixFormat_ARGB = 0,     // Alpha, Red, Green, Blue
    AE_PixFormat_BGRA          // Blue, Green, Red, Alpha
};
typedef A_long AE_PixFormat;
```

The pixel format is platform-dependent:
- **macOS:** Always `AE_PixFormat_ARGB`
- **Windows:** Always `AE_PixFormat_BGRA`

> **Pitfall:** The header says "for now" regarding this platform-dependent behavior. However, this has remained stable across many AE versions. Still, robust code should check the `pix_format` field rather than assuming based on platform.

### Accessing Pixel Data

```c
void ProcessPixBuffer(const AE_PixBuffer *buf) {
    if (!buf || !buf->pixelsPV) {
        // Blank frame requested
        return;
    }

    for (A_long y = 0; y < buf->heightL; y++) {
        // Use row_bytesL for stride, NOT widthL * chan_bytesL
        unsigned char *row = (unsigned char *)buf->pixelsPV + (y * buf->row_bytesL);

        for (A_long x = 0; x < buf->widthL; x++) {
            unsigned char *pixel = row + (x * buf->chan_bytesL);

            if (buf->plane_bytesL == 1) {
                // 8-bit: each channel is 1 byte
                unsigned char a = pixel[0];
                unsigned char r = pixel[1]; // Assuming ARGB
                unsigned char g = pixel[2];
                unsigned char b = pixel[3];
            }
            else if (buf->plane_bytesL == 2) {
                // 16-bit: each channel is 2 bytes
                unsigned short *p16 = (unsigned short *)pixel;
                // p16[0] = alpha, p16[1] = red, etc.
            }
            else if (buf->plane_bytesL == 4) {
                // 32-bit float: each channel is 4 bytes
                float *pF = (float *)pixel;
                // pF[0] = alpha, pF[1] = red, etc.
            }
        }
    }
}
```

> **Pitfall:** Always use `row_bytesL` for row stride, never compute it from `widthL * chan_bytesL`. The host may add padding bytes at the end of each row for alignment.

---

## View Coordinates

The `AE_ViewCoordinates` structure describes how the pixel buffer maps to the composition and the visible viewport:

```c
typedef struct {
    A_long  frame_widthL;       // Full composition width
    A_long  frame_heightL;      // Full composition height

    A_long  origin_xL;          // X offset of pix buffer in frame coords
    A_long  origin_yL;          // Y offset of pix buffer in frame coords

    A_long  view_rect_leftL;    // Visible viewport left
    A_long  view_rect_topL;     // Visible viewport top
    A_long  view_rect_rightL;   // Visible viewport right
    A_long  view_rect_bottomL;  // Visible viewport bottom
} AE_ViewCoordinates;
```

### Coordinate Relationships

```
+-- frame (frame_widthL x frame_heightL) --+
|                                           |
|   +-- pix buffer (at origin_x, origin_y) |
|   |   widthL x heightL                   |
|   |                                       |
|   |   +-- view_rect (visible area) --+    |
|   |   |                              |    |
|   |   |   What the user sees in      |    |
|   |   |   the comp viewer window     |    |
|   |   |                              |    |
|   |   +------------------------------+    |
|   |                                       |
|   +---------------------------------------+
|                                           |
+-------------------------------------------+
```

The pixel buffer may be a sub-region of the full frame (region of interest). The view rect describes what portion of the frame is visible in the AE viewer panel. A hook plugin displaying to an external monitor would typically want to show the full pixel buffer, while one overlaying UI might need the view rect to position elements.

---

## Blit Flags

### Input Flags

```c
enum {
    AE_BlitInFlag_NONE      = 0,
    AE_BlitInFlag_RENDERING = 1L << 0   // Blit is part of a render operation
};
typedef A_long AE_BlitInFlags;
```

When `AE_BlitInFlag_RENDERING` is set, the blit is occurring during an active render (e.g., RAM preview or export). The plugin may want to behave differently during rendering vs. interactive preview -- for instance, an external monitor plugin might not need to update during background renders.

### Output Flags

```c
enum {
    AE_BlitOutFlag_NONE         = 0,
    AE_BlitOutFlag_ASYNCHRONOUS = 1L << 0   // Blit will complete asynchronously
};
typedef A_long AE_BlitOutFlags;
```

If the plugin sets `AE_BlitOutFlag_ASYNCHRONOUS`, it is telling AE that the blit has not yet completed. The plugin must call the `AE_BlitCompleteFunc` (provided in the blit hook parameters) when the operation finishes.

---

## Asynchronous Blit Pattern

For hardware display or network-based preview, synchronous blitting may introduce unacceptable latency. The async pattern lets the plugin return immediately and notify AE when the display is complete.

```c
PF_Err MyBlitHook(
    void                        *hook_refconPV,
    const AE_PixBuffer          *pix_bufP0,
    const AE_ViewCoordinates    *viewP,
    AE_BlitReceipt              receipt,
    AE_BlitCompleteFunc         complete_func0,
    AE_BlitInFlags              in_flags,
    AE_BlitOutFlags             *out_flags)
{
    MyContext *ctx = (MyContext *)hook_refconPV;

    if (!pix_bufP0) {
        // Display blank frame synchronously
        ClearDisplay(ctx);
        *out_flags = AE_BlitOutFlag_NONE;
        return PF_Err_NONE;
    }

    if (complete_func0 && ctx->supports_async) {
        // Copy pixel data and submit to async display pipeline
        AsyncJob *job = CreateAsyncJob(ctx, pix_bufP0, viewP);
        job->receipt = receipt;
        job->complete_func = complete_func0;
        SubmitToDisplayThread(job);

        *out_flags = AE_BlitOutFlag_ASYNCHRONOUS;
    } else {
        // Synchronous fallback
        DisplayFrameSync(ctx, pix_bufP0, viewP);
        *out_flags = AE_BlitOutFlag_NONE;
    }

    return PF_Err_NONE;
}

// On the display thread, when done:
void OnDisplayComplete(AsyncJob *job, PF_Err err) {
    job->complete_func(job->receipt, err);
    FreeAsyncJob(job);
}
```

> **Pitfall:** If you declare asynchronous completion but never call the completion function, AE may hang waiting for the blit to finish. Always ensure the completion callback is invoked, even in error paths.

---

## Complete Hook Plugin Skeleton

```c
#include <AE_Hook.h>
#include <AE_Effect.h>

typedef struct {
    SPBasicSuite    *pica_basicP;
    A_long          frame_count;
} MyHookContext;

static void MyDeathHook(void *hook_refconPV) {
    MyHookContext *ctx = (MyHookContext *)hook_refconPV;
    if (ctx) {
        // Cleanup
        free(ctx);
    }
}

static void MyVersionHook(void *hook_refconPV, A_u_long *versionPV) {
    *versionPV = 0x00010000;
}

static PF_Err MyBlitHook(
    void                        *hook_refconPV,
    const AE_PixBuffer          *pix_bufP0,
    const AE_ViewCoordinates    *viewP,
    AE_BlitReceipt              receipt,
    AE_BlitCompleteFunc         complete_func0,
    AE_BlitInFlags              in_flags,
    AE_BlitOutFlags             *out_flags)
{
    MyHookContext *ctx = (MyHookContext *)hook_refconPV;
    *out_flags = AE_BlitOutFlag_NONE;

    if (!pix_bufP0) {
        return PF_Err_NONE;  // Blank frame, nothing to do
    }

    ctx->frame_count++;

    // Process the frame: send to external monitor, custom renderer, etc.
    // Access pixels: pix_bufP0->pixelsPV
    // Row stride: pix_bufP0->row_bytesL
    // Dimensions: pix_bufP0->widthL x pix_bufP0->heightL

    return PF_Err_NONE;
}

static void MyCursorHook(void *hook_refconPV, const AE_CursorInfo *cursorP) {
    // Handle cursor updates if needed
}

PF_Err MyHookPluginEntry(
    A_long          major_version,
    A_long          minor_version,
    AE_FileSpecH   file_specH,
    AE_FileSpecH   res_specH,
    AE_Hooks        *hooksP)
{
    if (major_version > AE_HOOK_MAJOR_VERSION) {
        return PF_Err_UNRECOGNIZED_PARAM_TYPE;
    }

    MyHookContext *ctx = (MyHookContext *)calloc(1, sizeof(MyHookContext));
    if (!ctx) {
        return PF_Err_OUT_OF_MEMORY;
    }

    hooksP->hook_refconPV     = ctx;
    hooksP->death_hook_func   = MyDeathHook;
    hooksP->version_hook_func = MyVersionHook;
    hooksP->blit_hook_func    = MyBlitHook;
    hooksP->cursor_hook_func  = MyCursorHook;

    // Note: hooksP->pica_basicP is set by the host AFTER this function returns.
    // Do not use it here.

    return PF_Err_NONE;
}
```

---

## When to Use Hooks

### Appropriate Use Cases

- **External monitor preview:** Sending composited frames to a dedicated preview monitor (SDI output, HDMI capture card, etc.)
- **Custom recording/capture:** Intercepting displayed frames for real-time recording
- **Alternative rendering display:** Routing frames to a custom rendering backend (e.g., VR headset preview)
- **Quality control overlays:** Overlaying waveform, vectorscope, or histogram displays on the output

### When NOT to Use Hooks

- **Standard effect rendering:** Use the normal `PF_Cmd_RENDER` / SmartFX pipeline instead
- **Modifying rendered output:** Hooks receive the final composited output; use effects for per-layer processing
- **General AEGP functionality:** Use the AEGP plugin interface with its richer suite ecosystem

---

## Pitfalls and Warnings

1. **pica_basicP is NOT available in the entry point.** The host fills in `AE_Hooks::pica_basicP` after the entry function returns. Do not attempt to acquire suites inside `MyHookPluginEntry`.

2. **Platform-dependent pixel format.** Always check `pix_bufP0->pix_format` rather than assuming ARGB or BGRA based on your build target.

3. **NULL pixel buffer means blank frame.** Do not dereference `pix_bufP0` without checking for NULL first.

4. **row_bytesL may include padding.** Never assume `row_bytesL == widthL * chan_bytesL`. The host may pad rows for alignment.

5. **Async completion is mandatory once declared.** If you set `AE_BlitOutFlag_ASYNCHRONOUS`, you must call `complete_func0` eventually, or AE will stall.

6. **The reserved array must not be touched.** The `reservedAPV[8]` field in `AE_Hooks` is used internally. Writing to it will corrupt internal data structures.

7. **Thread safety.** Blit hooks can be called from different threads depending on AE's rendering context. Protect shared state with appropriate synchronization if you cache data in your refcon.
