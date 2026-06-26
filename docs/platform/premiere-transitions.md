# Premiere Pro Transitions

This document covers Premiere Pro transition plugins, referencing the SDK 26.0 headers (`PrSDKEffect.h`, `PrSDKGPUFilter.h`). Transitions are a distinct plugin type from effects. If you know AE's effect system, the key architectural differences are: transitions receive two input clips, a completion ratio instead of a time value, and direction flags controlling the wipe/reveal geometry.

## How Transitions Differ from Effects

| Property | Premiere Effect (Video Filter) | Premiere Transition |
|---|---|---|
| Entry point type | `PRVideoFilterEntry` | `PRVideoTransitionEntry` |
| Data structure | `VideoRecord` / `VideoHandle` | `EffectRecord` / `EffectHandle` |
| Input clips | 1 (`source`) | 2 (`source1`, `source2`) |
| Progress | `part / total` = current time position | `part / total` = completion ratio (0% to 100%) |
| Direction controls | None | Arrow flags, start/end points, center, reverse flag |
| Selector enum | `PrFilterSelector` | `PrTransitionSelector` |

In the timeline, a transition sits at a cut point between two clips. The "outgoing" clip is ending (source1) and the "incoming" clip is beginning (source2).

---

## Entry Points

### Software Transition Entry

```cpp
// Old-style entry point (deprecated but still functional)
typedef PRFILTERENTRY (*EffectProcPtr)(
    short selector,
    EffectHandle theData);

// Modern entry point with plugin ID
typedef PRFILTERENTRY (*PRVideoTransitionEntry)(
    csSDK_int32 inID,
    short       inSelector,
    EffectHandle inData);
```

The entry point name in the PiPL is typically the function name registered as the transition entry. For GPU transitions, there is a separate entry point (see GPU section below).

### Software Effect Entry (for comparison)

```cpp
typedef PRFILTERENTRY (*PRVideoFilterEntry)(
    csSDK_int32 inID,
    short       selector,
    PrMemoryHandle theData);
```

---

## EffectRecord: The Transition Data Structure

When a transition selector fires, you receive an `EffectHandle` which dereferences to `EffectRecord`:

```cpp
typedef struct {
    PrMemoryHandle      specsHandle;       // Your persistent settings
    PPixHand            source1;           // Outgoing clip pixels
    PPixHand            source2;           // Incoming clip pixels
    PPixHand            destination;       // Output pixels (write here)

    csSDK_int32         part;              // Current position
    csSDK_int32         total;             // Total duration
                                           // part/total = completion ratio

    char                previewing;        // True during preview playback
    unsigned char       arrowFlags;        // Direction arrow bits
    char                reverse;           // Effect is being applied in reverse
    char                source;            // Sources are swapped

    prPoint             start;             // Starting point for directional effects
    prPoint             end;               // Ending point for directional effects
    prPoint             center;            // Reference center point

    void*               privateData;       // Editor-managed private data
    FXCallBackProcPtr   callBack;          // Callback (may be NULL)
    BottleRec*          bottleNecks;       // Bottleneck callbacks

    short               version;           // kEffectVersion
    short               sizeFlags;
    csSDK_int32         flags;             // Effect flags (see below)

    TDB_TimeRecord*     tdb;               // Time database
    piSuitesPtr         piSuites;          // Suite access
    PrTimelineID        timelineData;
    PrMemoryHandle      instanceData;      // Instance-specific data

    char                altName[MAX_FXALIAS];  // Alternate display name
    PrPixelFormat       pixelFormatSupported;  // Current pixel format query
    csSDK_int32         pixelFormatIndex;      // Pixel format query index
    csSDK_uint32        instanceID;            // Runtime instance ID
    TDB_TimeRecord      tdbTimelineLocation;   // Timeline position (setup only)
    csSDK_int32         sessionPluginID;       // Session-stable plugin ID
} EffectRecord, **EffectHandle;
```

### VideoRecord (Effect, for comparison)

```cpp
typedef struct {
    PrMemoryHandle      specsHandle;
    PPixHand            source;        // Single input
    PPixHand            destination;   // Output
    csSDK_int32         part;
    csSDK_int32         total;
    // ... similar fields but no source2, arrowFlags, etc.
} VideoRecord, **VideoHandle;
```

---

## Transition Selectors

From `enum PrTransitionSelector`:

| Selector | Value | Description |
|---|---|---|
| `esExecute` | 0 | Render one frame of the transition. |
| `esSetup` | 1 | Show settings dialog. |
| `esDisposeData` | 5 | Dispose of specsHandle data. |
| `esCanHandlePAR` | 6 | Report pixel aspect ratio handling capability. |
| `esGetPixelFormatsSupported` | 7 | Enumerate supported pixel formats. |
| `esCacheOnLoad` | 8 | Control DLL loading behavior. |

### Filter Selectors (for comparison)

| Selector | Value | Description |
|---|---|---|
| `fsExecute` | 0 | Render one frame of the effect. |
| `fsSetup` | 1 | Show settings dialog. |
| `fsDisposeData` | 3 | Dispose of specsHandle data. |
| `fsInitSpec` | 5 | Initialize default settings (no UI). |
| `fsGetPixelFormatsSupported` | 6 | Enumerate supported pixel formats. |
| `fsHasSetupDialog` | 8 | Report whether a setup dialog exists. |

---

## The Completion Ratio

The `part` and `total` fields define the transition progress:

```cpp
float completion = (float)effectRec->part / (float)effectRec->total;
// completion ranges from 0.0 (start) to 1.0 (end)
```

At `completion = 0.0`, source1 (outgoing clip) is fully visible.
At `completion = 1.0`, source2 (incoming clip) is fully visible.

If `effectRec->reverse` is true, the transition runs backward: source2 starts fully visible and source1 ends fully visible. Handle this by inverting your completion ratio:

```cpp
float t = (float)effectRec->part / (float)effectRec->total;
if (effectRec->reverse)
    t = 1.0f - t;
```

If `effectRec->source` is true, the source clips are swapped. In this case, `source1` is actually the incoming clip and `source2` is the outgoing clip:

```cpp
PPixHand outgoing = effectRec->source ? effectRec->source2 : effectRec->source1;
PPixHand incoming = effectRec->source ? effectRec->source1 : effectRec->source2;
```

---

## Effect Flags

The `flags` field in `EffectRecord` uses these bit flags:

```cpp
enum {
    kEffectFlags_DraftQuality              = 0x00000001,  // Render at draft quality
    kEffectFlags_TransitionHasIncomingClip  = 0x00000002,  // Incoming clip exists
    kEffectFlags_TransitionHasOutgoingClip  = 0x00000004,  // Outgoing clip exists
};
```

> **Critical**: A transition at the beginning of a sequence has no outgoing clip. A transition at the end has no incoming clip. Always check these flags before accessing `source1` or `source2`.

```cpp
bool hasOutgoing = (effectRec->flags & kEffectFlags_TransitionHasOutgoingClip) != 0;
bool hasIncoming = (effectRec->flags & kEffectFlags_TransitionHasIncomingClip) != 0;

if (!hasOutgoing)
{
    // source1 may be NULL or black -- treat as transparent/black
}
if (!hasIncoming)
{
    // source2 may be NULL or black
}
```

---

## Direction Arrows

The `arrowFlags` field indicates which directional arrows the user has enabled in the ECW (Effect Controls Window):

```cpp
enum {
    bitTop        = 0x01,
    bitRight      = 0x02,
    bitBottom     = 0x04,
    bitLeft       = 0x08,
    bitUpperRight = 0x10,
    bitLowerRight = 0x20,
    bitLowerLeft  = 0x40,
    bitUpperLeft  = 0x80,
};
```

Use these to determine the wipe direction:

```cpp
if (effectRec->arrowFlags & bitRight)
{
    // Wipe from left to right
}
else if (effectRec->arrowFlags & bitLeft)
{
    // Wipe from right to left
}
else if (effectRec->arrowFlags & bitTop)
{
    // Wipe from bottom to top
}
// ... etc.
```

The `start`, `end`, and `center` points provide additional geometry for directional transitions.

---

## Pixel Aspect Ratio Handling

When responding to `esCanHandlePAR`, return a combination of flags:

```cpp
case esCanHandlePAR:
{
    // We handle non-square pixels and need unity PAR for neither setup nor execute
    return prEffectCanHandlePAR;  // 0x4000
}
```

| Flag | Value | Meaning |
|---|---|---|
| `prEffectCanHandlePAR` | 0x4000 | Selector is implemented |
| `prEffectUnityPARSetup` | 0x0001 | Requires square pixels for setup dialog |
| `prEffectUnityPARExecute` | 0x0002 | Requires square pixels for rendering |

If you return `prEffectCanHandlePAR | prEffectUnityPARExecute`, Premiere will convert frames to square pixels before calling `esExecute`.

---

## Supported Pixel Formats

Handle `esGetPixelFormatsSupported` to declare what formats your transition can process. This is called repeatedly with `pixelFormatIndex` incrementing:

```cpp
case esGetPixelFormatsSupported:
{
    EffectRecord* rec = **theData;

    switch (rec->pixelFormatIndex)
    {
        case 0:
            rec->pixelFormatSupported = PrPixelFormat_BGRA_4444_8u;
            return esNoErr;
        case 1:
            rec->pixelFormatSupported = PrPixelFormat_BGRA_4444_32f;
            return esNoErr;
        default:
            return esBadFormatIndex;
    }
}
```

---

## Example: Cross-Dissolve Transition

A complete software cross-dissolve (based on the SDK_CrossDissolve example pattern):

```cpp
PRFILTERENTRY MyTransitionEntry(
    csSDK_int32   inID,
    short         inSelector,
    EffectHandle  inData)
{
    switch (inSelector)
    {
        case esExecute:
            return ExecuteTransition(inData);

        case esSetup:
            return esNoErr;  // No setup dialog

        case esGetPixelFormatsSupported:
        {
            EffectRecord* rec = **inData;
            switch (rec->pixelFormatIndex)
            {
                case 0:
                    rec->pixelFormatSupported = PrPixelFormat_BGRA_4444_8u;
                    return esNoErr;
                case 1:
                    rec->pixelFormatSupported = PrPixelFormat_BGRA_4444_32f;
                    return esNoErr;
                default:
                    return esBadFormatIndex;
            }
        }

        case esCanHandlePAR:
            return prEffectCanHandlePAR;

        default:
            return esUnsupported;
    }
}

short ExecuteTransition(EffectHandle inData)
{
    EffectRecord* effectRec = **inData;

    // Calculate blend ratio
    float t = (float)effectRec->part / (float)effectRec->total;
    if (effectRec->reverse)
        t = 1.0f - t;

    // Handle source swapping
    PPixHand outgoing = effectRec->source ? effectRec->source2 : effectRec->source1;
    PPixHand incoming = effectRec->source ? effectRec->source1 : effectRec->source2;
    PPixHand dest = effectRec->destination;

    // Check which clips are present
    bool hasOutgoing = (effectRec->flags & kEffectFlags_TransitionHasOutgoingClip) != 0;
    bool hasIncoming = (effectRec->flags & kEffectFlags_TransitionHasIncomingClip) != 0;

    // Get pixel suites
    PrSDKPPixSuite* ppixSuite = /* acquire from piSuites */;

    // Get destination properties
    prRect bounds;
    ppixSuite->GetBounds(dest, &bounds);
    int width = bounds.right - bounds.left;
    int height = bounds.bottom - bounds.top;

    char* destPixels = NULL;
    ppixSuite->GetPixels(dest, PrPPixBufferAccess_WriteOnly, &destPixels);
    csSDK_int32 destRowBytes;
    ppixSuite->GetRowBytes(dest, &destRowBytes);

    // Get source pixels (may be NULL if clip is missing)
    char* outgoingPixels = NULL;
    csSDK_int32 outgoingRowBytes = 0;
    if (hasOutgoing && outgoing)
    {
        ppixSuite->GetPixels(outgoing, PrPPixBufferAccess_ReadOnly, &outgoingPixels);
        ppixSuite->GetRowBytes(outgoing, &outgoingRowBytes);
    }

    char* incomingPixels = NULL;
    csSDK_int32 incomingRowBytes = 0;
    if (hasIncoming && incoming)
    {
        ppixSuite->GetPixels(incoming, PrPPixBufferAccess_ReadOnly, &incomingPixels);
        ppixSuite->GetRowBytes(incoming, &incomingRowBytes);
    }

    // Determine pixel format
    PrPixelFormat format;
    ppixSuite->GetPixelFormat(dest, &format);

    if (format == PrPixelFormat_BGRA_4444_8u)
    {
        for (int y = 0; y < height; y++)
        {
            const uint8_t* srcA = outgoingPixels
                ? reinterpret_cast<const uint8_t*>(outgoingPixels + y * outgoingRowBytes)
                : NULL;
            const uint8_t* srcB = incomingPixels
                ? reinterpret_cast<const uint8_t*>(incomingPixels + y * incomingRowBytes)
                : NULL;
            uint8_t* dst = reinterpret_cast<uint8_t*>(destPixels + y * destRowBytes);

            for (int x = 0; x < width; x++)
            {
                int idx = x * 4;
                for (int c = 0; c < 4; c++)
                {
                    uint8_t a = srcA ? srcA[idx + c] : 0;
                    uint8_t b = srcB ? srcB[idx + c] : 0;
                    dst[idx + c] = (uint8_t)(a * (1.0f - t) + b * t);
                }
            }
        }
    }
    else if (format == PrPixelFormat_BGRA_4444_32f)
    {
        for (int y = 0; y < height; y++)
        {
            const float* srcA = outgoingPixels
                ? reinterpret_cast<const float*>(outgoingPixels + y * outgoingRowBytes)
                : NULL;
            const float* srcB = incomingPixels
                ? reinterpret_cast<const float*>(incomingPixels + y * incomingRowBytes)
                : NULL;
            float* dst = reinterpret_cast<float*>(destPixels + y * destRowBytes);

            for (int x = 0; x < width * 4; x++)
            {
                float a = srcA ? srcA[x] : 0.0f;
                float b = srcB ? srcB[x] : 0.0f;
                dst[x] = a * (1.0f - t) + b * t;
            }
        }
    }

    return esNoErr;
}
```

---

## GPU-Accelerated Transitions

GPU transitions use the same `PrGPUFilter` interface as GPU effects, with transition-specific behavior:

```cpp
// PrSDKGPUFilter.h
#define PrGPUFilterEntryPointName "xGPUFilterEntry"

typedef prSuiteError (*PrGPUFilterEntry)(
    csSDK_uint32    inHostInterfaceVersion,
    csSDK_int32*    ioIndex,
    prBool          inStartup,     // 1=startup, 0=shutdown
    piSuitesPtr     piSuites,
    PrGPUFilter*    outFilter,
    PrGPUFilterInfo* outFilterInfo);
```

The `PrGPUFilter` struct contains callbacks:

```cpp
typedef struct {
    prSuiteError (*CreateInstance)(PrGPUFilterInstance* ioInstanceData);
    prSuiteError (*DisposeInstance)(PrGPUFilterInstance* ioInstanceData);
    prSuiteError (*GetFrameDependencies)(
        PrGPUFilterInstance* inInstanceData,
        const PrGPUFilterRenderParams* inRenderParams,
        csSDK_int32* ioQueryIndex,
        PrGPUFilterFrameDependency* outFrameDependencies);
    prSuiteError (*Precompute)(
        PrGPUFilterInstance* inInstanceData,
        const PrGPUFilterRenderParams* inRenderParams,
        csSDK_int32 inIndex,
        PPixHand inFrame);
    prSuiteError (*Render)(
        PrGPUFilterInstance* inInstanceData,
        const PrGPUFilterRenderParams* inRenderParams,
        const PPixHand* inFrames,
        csSDK_size_t inFrameCount,
        PPixHand* outFrame);
} PrGPUFilter;
```

### GPU Transitions vs GPU Effects

For **effects**: `inFrames[0]` is always the frame at the current time. Additional frames (temporal dependencies) follow in the order returned by `GetFrameDependencies`.

For **transitions**: `inFrames[0]` is the **incoming** clip frame and `inFrames[1]` is the **outgoing** clip frame. Transitions may not have additional frame dependencies beyond these two.

### Transition-Specific Frame Dependencies

Use `PrGPUDependency_TransitionInputFrame` to request frames from the incoming/outgoing clips:

```cpp
prSuiteError GetFrameDependencies(
    PrGPUFilterInstance* inInstanceData,
    const PrGPUFilterRenderParams* inRenderParams,
    csSDK_int32* ioQueryIndex,
    PrGPUFilterFrameDependency* outFrameDependencies)
{
    switch (*ioQueryIndex)
    {
        case 0:
            // Request outgoing clip frame
            outFrameDependencies->outDependencyType =
                PrGPUDependency_TransitionInputFrame;
            outFrameDependencies->outSequenceTime =
                inRenderParams->inSequenceTime;
            outFrameDependencies->outReadIncomingTransition = kPrFalse;
            (*ioQueryIndex)++;
            return suiteError_NoError;

        case 1:
            // Request incoming clip frame
            outFrameDependencies->outDependencyType =
                PrGPUDependency_TransitionInputFrame;
            outFrameDependencies->outSequenceTime =
                inRenderParams->inSequenceTime;
            outFrameDependencies->outReadIncomingTransition = kPrTrue;
            (*ioQueryIndex)++;
            return suiteError_NoError;

        default:
            return suiteError_NoMoreData;
    }
}
```

### Understanding Clip Sides

The terminology can be confusing. From the documentation in `PrSDKGPUFilter.h`:

> Whether a clip has an "incoming" or "outgoing" transition is from the perspective of the clip, not the transition. A transition on the left edge of the clip is an "incoming" transition, while a transition on the right edge of the clip is an "outgoing" transition. So from the perspective of a transition at a cut point, the clip on the left of the transition is "outgoing" and the clip on the right of the transition is "incoming".

In other words:
- `outReadIncomingTransition = kPrFalse` reads from the clip **left** of the cut (outgoing/ending)
- `outReadIncomingTransition = kPrTrue` reads from the clip **right** of the cut (incoming/starting)

### GPU Transition Render

```cpp
prSuiteError Render(
    PrGPUFilterInstance* inInstanceData,
    const PrGPUFilterRenderParams* inRenderParams,
    const PPixHand* inFrames,
    csSDK_size_t inFrameCount,
    PPixHand* outFrame)
{
    // inFrames[0] = incoming clip (right side of cut)
    // inFrames[1] = outgoing clip (left side of cut)

    // Calculate completion from clip time and transition duration
    // (you need to track these in your instance data from
    //  PrSDKVideoSegmentSuite during CreateInstance)
    float t = CalculateCompletion(inInstanceData, inRenderParams);

    // Perform GPU blend
    // Upload/process using CUDA, OpenCL, or Metal
    GPUCrossDissolve(inFrames[0], inFrames[1], *outFrame, t);

    return suiteError_NoError;
}
```

### PrGPUFilterRenderParams

The render parameters provide timing and quality information:

```cpp
typedef struct {
    PrTime           inClipTime;           // Time relative to clip
    PrTime           inSequenceTime;       // Time relative to sequence
    PrRenderQuality  inQuality;
    float            inDownsampleFactorX;
    float            inDownsampleFactorY;
    csSDK_uint32     inRenderWidth;
    csSDK_uint32     inRenderHeight;
    csSDK_uint32     inRenderPARNum;
    csSDK_uint32     inRenderPARDen;
    prFieldType      inRenderFieldType;
    PrTime           inRenderTicksPerFrame;
    pmFieldDisplay   inRenderField;
} PrGPUFilterRenderParams;
```

### PrGPUFilterInfo: Registration

The `outMatchName` in `PrGPUFilterInfo` must match a registered software transition. This links the GPU implementation to the software fallback:

```cpp
prSuiteError MyGPUEntry(
    csSDK_uint32     inHostInterfaceVersion,
    csSDK_int32*     ioIndex,
    prBool           inStartup,
    piSuitesPtr      piSuites,
    PrGPUFilter*     outFilter,
    PrGPUFilterInfo* outFilterInfo)
{
    if (inStartup)
    {
        outFilterInfo->outInterfaceVersion = PrGPUFilterInterfaceVersion;
        // Match to the software transition's PiPL match name
        // If NULL, defaults to this module's PiPL
        outFilterInfo->outMatchName = /* PrSDKString of match name */;

        outFilter->CreateInstance = MyCreateInstance;
        outFilter->DisposeInstance = MyDisposeInstance;
        outFilter->GetFrameDependencies = MyGetFrameDependencies;
        outFilter->Precompute = NULL;  // Not needed for transitions
        outFilter->Render = MyRender;

        return suiteError_NoError;
    }
    else
    {
        // Shutdown
        return suiteError_NoError;
    }
}
```

---

## Return Values

### Transition Returns

| Value | Meaning |
|---|---|
| `esNoErr` | Success |
| `esBadFormatIndex` | No more pixel formats to enumerate |
| `esDoNotCacheOnLoad` | Force DLL reload on each startup |
| `esUnsupported` | Selector not handled |

### Filter Returns (for comparison)

| Value | Meaning |
|---|---|
| `fsNoErr` | Success |
| `fsBadFormatIndex` | No more pixel formats |
| `fsDoNotCacheOnLoad` | Force DLL reload |
| `fsHasNoSetupDialog` | No setup dialog |
| `fsUnsupported` | Selector not handled |

---

## Version Constants

```cpp
#define TRANSITION_VERSION_13   12   // CS7 / CC (note: value 12, not 13)
#define kEffectVersion          TRANSITION_VERSION_13

#define VIDEO_FILTER_VERSION_12 12   // CS5.5
#define kVideoFilterVersion     VIDEO_FILTER_VERSION_12
```

Set `version` in `EffectRecord` or `VideoRecord` checks against these constants.

---

## Pitfalls and AE Developer Notes

1. **Transitions are not effects.** They have a different entry point signature, different data structure, and different selectors. You cannot simply reuse an AE effect as a Premiere transition without restructuring the code.

2. **Always check for missing clips.** A transition at the start or end of a sequence will have only one clip. Check `kEffectFlags_TransitionHasIncomingClip` and `kEffectFlags_TransitionHasOutgoingClip` before accessing source1/source2.

3. **Handle the reverse and source flags.** Users can reverse transitions and swap sources in the timeline. Your rendering must account for both flags, or the transition will appear incorrect when these options are toggled.

4. **GPU transitions receive frames in a different order than you might expect.** Frame 0 is incoming, frame 1 is outgoing. This is the opposite of the software path where source1 is typically outgoing. Test both software and GPU paths carefully.

5. **Pixel format negotiation is mandatory.** If you do not handle `esGetPixelFormatsSupported`, Premiere may not call your transition at all or may call it with an unexpected format.

6. **Draft quality matters.** Check `kEffectFlags_DraftQuality` and reduce rendering complexity during preview playback. This is how Premiere maintains real-time performance.

7. **specsHandle vs instanceData.** `specsHandle` contains persistent settings that are saved with the project. `instanceData` is runtime-only instance data that is recreated when the project is reopened. Use `specsHandle` for user-facing settings and `instanceData` for runtime caches.

8. **The callback pointer may be NULL.** The `callBack` field in `EffectRecord` is not always valid. Check for NULL before using it.

9. **No parameter UI system like effects.** Software transitions use `specsHandle` for settings and `esSetup` for a modal dialog. They do not have the rich parameter UI that exporters get through `PrSDKExportParamSuite`. GPU transitions get their parameters through `PrSDKVideoSegmentSuite` node properties.

10. **Thread safety.** Both software and GPU transition render calls can happen on multiple threads simultaneously. Ensure your rendering code is thread-safe and does not modify shared state.
