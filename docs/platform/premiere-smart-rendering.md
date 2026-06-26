# Premiere Pro Smart Rendering

Smart rendering in Premiere Pro allows an exporter to bypass unnecessary decoding and re-encoding by identifying timeline segments where the source media can be passed through directly. This is fundamentally different from After Effects' SmartFX system, which is about pre-render/render phase separation for effects. Premiere's smart rendering is an export optimization that analyzes the video segment tree to find pass-through opportunities.

This guide also covers the Accelerated Render system, which enables custom sequence renderers that can provide hardware-accelerated timeline rendering with their own compositing and effects pipelines.

**SDK Headers Covered:**
- `PrSDKSmartRenderingSuite.h` -- Smart render segment analysis
- `PrSDKAcceleratedRender.h` -- Custom sequence renderer architecture
- `PrSDKAcceleratedRenderInvocationSuite.h` -- Creating and controlling renderer instances
- `PrSDKQuality.h` -- Render and playback quality levels

---

## Smart Rendering (PrSDKSmartRenderingSuite)

Suite name: `"MediaCore Smart Rendering Suite"` (version 3)

### What Smart Rendering Means in Premiere

When exporting a sequence, Premiere can identify segments where the source media matches the output format closely enough that the compressed bitstream can be copied directly, without decoding to pixels and re-encoding. This dramatically speeds up exports and preserves quality.

The Smart Rendering Suite analyzes the video segment tree and produces a list of `PrClipSegmentInfo` entries that describe which source clips can be smart-rendered and which need full rendering.

### The PrClipSegmentInfo Structure

```c
typedef struct {
    csSDK_int32 mClipID;
    csSDK_int64 mSegmentStartTime;
    csSDK_int64 mSegmentEndTime;
    csSDK_int64 mSegmentOffset;
    csSDK_int64 mClipStartTime;
    csSDK_int64 mClipEndTime;
    PrSDKString mClipPath;       // Disposed after callback returns -- copy if needed
    csSDK_int64 mMediaStartTime;
    csSDK_int64 mMediaEndTime;
} PrClipSegmentInfo;
```

Each entry describes a contiguous range of the timeline that maps to a specific clip and time range within that clip's media file. The `mClipPath` gives you the file path, while the media start/end times tell you exactly which portion of the source bitstream to copy.

### Building the Segment List

The suite uses a callback pattern. You provide a function that receives each segment:

```c
typedef void (*SegmentInfoCallback)(void* inCallbackData, PrClipSegmentInfo* inClipSegmentInfo);
```

Then call one of the build functions:

```c
// Standard: includes preview files where available
prSuiteError (*BuildSmartRenderSegmentList)(
    SegmentInfoCallback inCallbackFunc,
    void*               inCallbackData,
    csSDK_int32         inSegmentsID,       // From PrSDKVideoSegmentSuite
    PrTime              inTimeBase,
    PrPixelFormat       inPixelFormat);

// Without preview files
prSuiteError (*BuildSmartRenderSegmentListNoPreviewFiles)(
    SegmentInfoCallback inCallbackFunc,
    void*               inCallbackData,
    csSDK_int32         inSegmentsID,
    PrTime              inTimeBase,
    PrPixelFormat       inPixelFormat);

// Ancillary data (captions, metadata)
prSuiteError (*BuildAncillaryDataSegmentMap)(
    SegmentInfoCallback inCallbackFunc,
    void*               inCallbackData,
    csSDK_int32         inVideoSegmentsID,
    PrTime              inTimeBase,
    PrPixelFormat       inPixelFormat);
```

### Color-Managed Variants (Version 3)

```c
prSuiteError (*BuildColorManagedSmartRenderSegmentList)(
    SegmentInfoCallback inCallbackFunc,
    void*               inCallbackData,
    csSDK_int32         inSegmentsID,
    PrTime              inTimeBase,
    PrPixelFormat       inPixelFormat,
    PrSDKColorSpaceID   inColorSpaceID);

prSuiteError (*BuildColorManagedSmartRenderSegmentListNoPreviewFiles)(
    SegmentInfoCallback inCallbackFunc,
    void*               inCallbackData,
    csSDK_int32         inSegmentsID,
    PrTime              inTimeBase,
    PrPixelFormat       inPixelFormat,
    PrSDKColorSpaceID   inColorSpaceID);
```

### Usage in an Exporter

```cpp
struct SmartRenderData {
    std::vector<PrClipSegmentInfo> segments;
};

void SmartRenderCallback(void* inCallbackData, PrClipSegmentInfo* inClipSegmentInfo)
{
    auto* data = static_cast<SmartRenderData*>(inCallbackData);

    PrClipSegmentInfo copy = *inClipSegmentInfo;
    // IMPORTANT: mClipPath is disposed after this callback returns.
    // If you need to keep it, you must copy the string data now.
    // copy.mClipPath = MakeCopyOfPrSDKString(inClipSegmentInfo->mClipPath);

    data->segments.push_back(copy);
}

void AnalyzeForSmartRendering(
    PrSDKSmartRenderingSuite* smartSuite,
    PrSDKVideoSegmentSuite* segSuite,
    PrTimelineID timelineID,
    PrTime timeBase,
    PrPixelFormat targetFormat)
{
    csSDK_int32 segmentsID = 0;
    segSuite->AcquireVideoSegmentsID(timelineID, &segmentsID);

    SmartRenderData data;
    smartSuite->BuildSmartRenderSegmentList(
        SmartRenderCallback, &data, segmentsID, timeBase, targetFormat);

    for (auto& seg : data.segments)
    {
        if (seg.mClipID != 0)
        {
            // This segment can be smart-rendered from the source clip
            // Copy bitstream from seg.mMediaStartTime to seg.mMediaEndTime
        }
        else
        {
            // This segment must be fully rendered
        }
    }

    segSuite->ReleaseVideoSegmentsID(segmentsID);
}
```

### When Smart Rendering Applies

Smart rendering is possible when ALL of the following are true for a segment:
1. The segment contains a single clip with no effects (or only passthrough effects)
2. The source codec matches the output codec
3. The source frame rate matches the output frame rate
4. No speed changes, time remapping, or frame blending are active
5. No color space conversions are needed (or the color-managed variant is used with matching spaces)
6. The clip is not trimmed at non-GOP boundaries (for long-GOP codecs)

---

## Accelerated Render System (PrSDKAcceleratedRender.h)

The Accelerated Render system allows third-party plugins to replace Premiere's built-in software renderer with a custom rendering pipeline. This is used by GPU acceleration plugins (like the Mercury Playback Engine) and hardware vendor integrations.

### Version History

| Version | Release | Notes |
|---------|---------|-------|
| 100 (1) | CS4 | Initial version |
| 200 (2) | CS5 | |
| 300 (3) | CS5.0.1 | |
| 400 (4) | CS5.5 | |
| 500 (5) | CS6 | |
| 600 (6) | CS7 | |
| 700 (7) | CC8 | |
| 800 (8) | Pr 14.0 | |
| 900 (9) | | Export LUT support |
| 1000 (10) | | Render session start/finish |
| 1100 (11) | | Hardware resident frame support |

### Renderer Registration

An accelerated renderer provides its identity during startup:

```c
typedef struct {
    prPluginID   outIdentifier;         // Unique GUID
    int          outPriority;           // 0 = host renderer priority
                                        // >0 = higher wins
                                        // <0 = not enabled by default
                                        // <= -100 = hidden from UI
    prUTF16Char  outDisplayName[256];
} arRendererInfo;
```

### Standard Parameters

```c
typedef struct {
    int      inInterfaceVer;          // RENDERMOD_VERSION
    int      inPluginID;
    int      inIndex;
    void*    ioPluginPrivateInstanceData;  // Set in Startup, used throughout
    piSuites* (*inGetPluginInterfaceSuiteCallback)();
} arStdParms;
```

### Sequence Data

Provided when setting up a renderer for a specific sequence:

```c
typedef struct {
    PrTimelineID     inTimelineID;
    int              inExportFlags;
    PrTime           inTimelineFrameRate;
    int              inTimelineWidth;
    int              inTimelineHeight;
    csSDK_uint32     inTimelinePARNum;
    csSDK_uint32     inTimelinePARDen;
    prFieldType      inTimelineFieldType;
    void*            ioPrivatePluginSequenceData;
    PrSDKStreamLabel inStreamLabel;
} arSequenceData;
```

### Render Requests

Render requests have evolved through multiple versions. The base version:

```c
typedef struct {
    PrTime       inTime;
    int          inWidth;
    int          inHeight;
    csSDK_uint32 inPARNum;
    csSDK_uint32 inPARDen;
    PrPixelFormat inPixelFormat;
    PrRenderQuality inQuality;
    bool         inPrefetchOnly;       // Just prefetch, do not render
    bool         inCompositeOnBlack;

    AcceleratedRendererCompletionCallback inCompletionCallback;
    void*        inCompletionCallbackData;
    csSDK_int32  inRequestID;          // Provided by host for cancellation

    bool         inRenderFields;
    imRenderContext mRenderContext;
} arRenderRequest;
```

Later versions added:
- **Version 2** (`arRenderRequest2`): Caption stream format
- **Version 3** (`arRenderRequest3`): Final render pixel format
- **Version 4** (`arRenderRequest4`): Color space ID for color management
- **Version 5** (`arRenderRequest5`): Export LUT ID
- **Version 6** (`arRenderRequest6`): Hardware resident frame output support

The completion callback:

```c
enum {
    arCompletion_Success   =  0,
    arCompletion_Error     = -1,
    arCompletion_Cancelled = -2
};

typedef void (*AcceleratedRendererCompletionCallback)(
    void* inCallbackData,
    csSDK_int32 inRequestID,
    PPixHand inPPix,
    int inCompletion);
```

### Segment Status

Renderers report what they can handle per-segment:

```c
typedef struct {
    PrTime inStartTime;    // Provided by host
    PrTime outEndTime;     // Filled by plugin
    int    outStatus;      // arSegmentStatus enum
} arVideoSegmentInfo;
```

### Selectors

Accelerated renderers respond to selectors dispatched through their entry point. Key selectors include:
- **Startup/Shutdown**: Initialize and clean up the renderer
- **SequenceSetup/SequenceSetdown**: Prepare for a specific sequence
- **RenderFrame**: Asynchronous render request
- **CancelFrame**: Cancel a pending render
- **QuerySegment**: Report rendering capabilities for a time range
- **PrefetchFrame**: Pre-load media for upcoming renders

---

## PrSDKAcceleratedRenderInvocationSuite

Suite name: `"MediaCore Accelerated Render Invocation Suite"` (version 4)

This suite lets other plugins create and control accelerated renderer instances programmatically (rather than using the UI-selected renderer).

### Getting the Current Renderer

```c
prSuiteError (*GetCurrentAcceleratedSequenceRendererID)(
    prPluginID* outRendererID);  // NULL when software renderer is selected
```

### Creating a Renderer Instance

```c
prSuiteError (*CreateAcceleratedSequenceRenderer)(
    prPluginID*   inRendererID,
    PrTimelineID  inSequence,       // Must be a top-level sequence
    prBool        inUsePreviews,
    csSDK_uint32* outRendererInstanceID);

// With stream label (version 3+)
prSuiteError (*CreateAcceleratedSequenceRendererWithStreamLabel)(
    prPluginID*      inRendererID,
    PrTimelineID     inSequence,
    prBool           inUsePreviews,
    PrSDKStreamLabel inStreamLabel,
    csSDK_uint32*    outRendererInstanceID);

prSuiteError (*DisposeAcceleratedSequenceRenderer)(
    csSDK_uint32 inRendererInstanceID);
```

### Initiating Renders

```c
prSuiteError (*InitiateRender)(
    csSDK_uint32     inRendererInstanceID,
    arRenderRequest* ioRenderData);  // inRequestID filled by host

// Version 4: with caption support
prSuiteError (*InitiateRender2)(
    csSDK_uint32      inRendererInstanceID,
    arRenderRequest2* ioRenderData);

prSuiteError (*CancelRender)(
    csSDK_uint32 inRendererInstanceID,
    csSDK_uint32 inRequestID);
```

### Querying Segment Properties

```c
prSuiteError (*QuerySegmentProperties)(
    csSDK_uint32    inRendererInstanceID,
    PrTime          inStartTime,
    PrTime*         outEndTime,
    arSegmentStatus* outStatus,
    PrPixelFormat*  outPixelFormats,       // Array to fill
    csSDK_int32*    ioPixelFormatCount);   // In: array size, Out: count filled
```

### Display Name and Color Management

```c
prSuiteError (*GetAcceleratedSequenceRendererDisplayName)(
    prPluginID* inRendererID,
    prUTF16Char outDisplayName[256]);

prSuiteError (*GetEnabledDisplayColorManagement)(
    prBool* outEnabledDisplayColorManagement);
```

---

## Quality Settings (PrSDKQuality.h)

### Render Quality

Used during export and effect rendering:

```c
typedef enum {
    kPrRenderQuality_Max     = 4,
    kPrRenderQuality_High    = 3,
    kPrRenderQuality_Medium  = 2,
    kPrRenderQuality_Low     = 1,
    kPrRenderQuality_Draft   = 0,
    kPrRenderQuality_Invalid = 0xFFFFFFFF,
} PrRenderQuality;
```

### Playback Quality

Used during real-time playback:

```c
typedef enum {
    kPrPlaybackQuality_Invalid = 4,
    kPrPlaybackQuality_High    = 3,
    kPrPlaybackQuality_Draft   = 2,
    kPrPlaybackQuality_Auto    = 1,
} PrPlaybackQuality;
```

### Playback Fractional Resolution

Controls downsampling during playback for performance:

```c
typedef enum {
    kPrPlaybackFractionalResolution_Invalid    = 6,
    kPrPlaybackFractionalResolution_Sixteenth  = 5,
    kPrPlaybackFractionalResolution_Eighth     = 4,
    kPrPlaybackFractionalResolution_Quarter    = 3,
    kPrPlaybackFractionalResolution_Half       = 2,
    kPrPlaybackFractionalResolution_Full       = 1,
} PrPlaybackFractionalResolution;
```

---

## AE SmartFX vs. Premiere Smart Rendering: A Comparison

| Aspect | AE SmartFX | Premiere Smart Rendering |
|--------|-----------|--------------------------|
| Purpose | Pre-render/render phase separation for effects | Export optimization via bitstream pass-through |
| Scope | Single effect instance | Entire timeline |
| Phase model | `PF_Cmd_SMART_PRE_RENDER` -> `PF_Cmd_SMART_RENDER` | Segment analysis -> per-segment decision |
| Developer role | Effect author implements both phases | Exporter uses the suite; effects are passive |
| When it runs | Every frame during composition | Only during export |
| What it optimizes | Pixel format, ROI, and checkout for effects | Whether to decode/encode at all |

---

## Pitfalls and Warnings

**Smart rendering is an export-only optimization.** It has no effect on playback or preview rendering. During playback, the Accelerated Render system and GPU filters provide performance.

**`mClipPath` lifetime is limited.** The `PrSDKString` in `PrClipSegmentInfo` is disposed immediately after the callback returns. If you need to use the path later, copy the string data during the callback.

**Accelerated renderer version compatibility.** The header provides conversion functions (`ConvertToArRenderRequest2`, etc.) to handle version mismatches. Always check `inInterfaceVer` in `arStdParms` and handle older request formats gracefully.

**Priority ordering matters.** An accelerated renderer with priority > 0 will be preferred over the host renderer. Priority < 0 means the renderer must be manually selected by the user. Priority <= -100 hides it from the UI entirely.

**Render requests are asynchronous.** The completion callback may fire before `InitiateRender` returns. Do not assume synchronous completion. The `inRequestID` in the render request is filled by the host during the call.

**Color management changes in recent versions.** Version 4 of the render request added `inPrSDKFinalRenderColorSpaceID`. Version 5 added export LUT support (`inPrSDKFinalRenderLUTID`). Always initialize these to their `_Invalid` constants when converting from older request versions.
