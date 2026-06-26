# Premiere Pro Video Segment API

The Video Segment API is Premiere Pro's representation of the timeline rendering graph. Unlike After Effects, which renders effects in a simple layer-and-effect stack model, Premiere decomposes the entire timeline into a tree of typed nodes representing clips, effects, transitions, compositors, and media sources. Understanding this system is essential for accelerated renderers, GPU filters, and any plugin that needs to inspect or render timeline content programmatically.

**SDK Headers Covered:**
- `PrSDKVideoSegmentSuite.h` -- Timeline segment tree traversal and node inspection
- `PrSDKVideoSegmentProperties.h` -- Property key definitions for all node types
- `PrSDKVideoSegmentRenderSuite.h` -- Async rendering of individual nodes and operators
- `PrSDKClipRenderSuite.h` -- Clip-level frame access and format negotiation

---

## Conceptual Model

### What Are Video Segments?

A Premiere Pro sequence is divided into **segments** -- contiguous time ranges where the rendering graph does not change structurally. A new segment boundary occurs whenever clips, transitions, or effects start or end. Within each segment, the rendering graph is a tree of **nodes**.

```
Sequence Timeline:
|--- Segment 0 ---|--- Segment 1 ---|--- Segment 2 ---|
     (Clip A)      (Clip A + B       (Clip B)
                    with transition)
```

Each segment has:
- A start time and end time (in sequence time)
- A segment offset (time offset for the segment's content)
- A unique hash (GUID) that identifies the rendering graph for caching

### The Node Tree

Within each segment, the rendering graph is a tree. The root is typically a Compositor node. Nodes have **inputs** (child nodes providing source frames) and **operators** (effect nodes that process the result).

```
Compositor
  +-- Clip (Track 2)
  |     +-- [operator] Effect: Color Correction
  |     +-- [operator] Effect: Blur
  |     +-- [input] Media
  +-- Clip (Track 1)
  |     +-- [operator] Effect: Levels
  |     +-- [input] Media
  +-- [operator] Transition: Cross Dissolve (between Track 1 clips)
```

### Node Types

Defined as string constants in `PrSDKVideoSegmentSuite.h`:

| Node Type String | Description |
|------------------|-------------|
| `"RenderableNodeClipImpl"` | A clip on a track (handles speed, time remapping) |
| `"RenderableNodeCompositorImpl"` | Composites multiple tracks together |
| `"RenderableNodeDisabledImpl"` | A disabled clip or effect |
| `"RenderableNodeEffectImpl"` | An applied video effect |
| `"RenderableNodeTransitionImpl"` | A transition between clips |
| `"RenderableNodeMediaImpl"` | A media source (file on disk) |
| `"RenderableNodePreviewImpl"` | A preview render file |
| `"RenderableNodeSolidColorImpl"` | A solid color generator |
| `"RenderableNodeMulticamImpl"` | A multicam clip |
| `"RenderableNode_AdjustmentImpl"` | An adjustment layer |
| `"RenderableNode_AdjustmentEffectImpl"` | An effect on an adjustment layer |

---

## PrSDKVideoSegmentSuite

Suite name: `"MediaCore Video Segment Suite"` (version 9)

### Version History

| Version | Release | Addition |
|---------|---------|----------|
| 4 | CS4+ | Base version |
| 5 | | Added stream labels |
| 6 | | Added `AcquireFirstNodeInTimeRange` |
| 7 | | Added graphics transforms |
| 8 | | Added color-managed `GetVideoSegmentsPropertiesExt` |
| 9 | | Added `GetNodeTimeScale` |

### Acquiring Video Segments

To inspect a timeline's rendering graph, first acquire a video segments ID:

```c
// Basic acquisition
prSuiteError (*AcquireVideoSegmentsID)(
    PrTimelineID inTimelineData,
    csSDK_int32* outVideoSegmentsID);

// With preview files substituted in
prSuiteError (*AcquireVideoSegmentsWithPreviewsID)(
    PrTimelineID inTimelineData,
    csSDK_int32* outVideoSegmentsID);

// Only opaque previews (for nested sequences)
prSuiteError (*AcquireVideoSegmentsWithOpaquePreviewsID)(
    PrTimelineID inTimelineData,
    csSDK_int32* outVideoSegmentsID);
```

Stream-label-aware variants (version 5+):

```c
prSuiteError (*AcquireVideoSegmentsIDWithStreamLabel)(
    PrTimelineID inTimelineData,
    PrSDKStreamLabel inStreamLabel,
    csSDK_int32* outVideoSegmentsID);
```

**All segment IDs are ref-counted and must be released:**

```c
prSuiteError (*ReleaseVideoSegmentsID)(csSDK_int32 inVideoSegmentsID);
```

### Querying Sequence Properties

```c
prSuiteError (*GetVideoSegmentsProperties)(
    PrTimelineID inTimelineData,
    prRect* outBounds,
    csSDK_int32* outPixelAspectRatioNumerator,
    csSDK_int32* outPixelAspectRatioDenominator,
    PrTime* outFrameRate,
    prFieldType* outFieldType);

// Color-managed version (version 8+)
prSuiteError (*GetVideoSegmentsPropertiesExt)(
    PrTimelineID inTimelineData,
    prRect* outBounds,
    csSDK_int32* outPixelAspectRatioNumerator,
    csSDK_int32* outPixelAspectRatioDenominator,
    PrTime* outFrameRate,
    prFieldType* outFieldType,
    PrSDKColorSpaceID* outColorSpaceID);
```

### Enumerating Segments

```c
// Get total segment count
prSuiteError (*GetSegmentCount)(csSDK_int32 inVideoSegmentsID, csSDK_int32* outNumSegments);

// Get info about the Nth segment
prSuiteError (*GetSegmentInfo)(
    csSDK_int32 inVideoSegmentsID,
    csSDK_int32 inIndex,
    PrTime* outStartTime,
    PrTime* outEndTime,
    PrTime* outSegmentOffset,
    prPluginID* outHash);  // Unique hash for caching
```

You can also get a hash for the entire segments object for change detection:

```c
prSuiteError (*GetHash)(csSDK_int32 inVideoSegmentsID, prPluginID* outHash);
```

### Acquiring Nodes

Nodes are ref-counted objects obtained by hash or by time:

```c
// By segment hash
prSuiteError (*AcquireNodeID)(
    csSDK_int32 inVideoSegmentsID,
    prPluginID* inHash,
    csSDK_int32* outVideoNodeID);

// By time (finds the segment containing that time)
prSuiteError (*AcquireNodeForTime)(
    csSDK_int32 inVideoSegmentsID,
    PrTime inTime,
    csSDK_int32* outVideoNodeID,
    PrTime* outSegmentOffset);

// By time range (version 6+)
prSuiteError (*AcquireFirstNodeInTimeRange)(
    csSDK_int32 inVideoSegmentsID,
    PrTime inStartTime, PrTime inEndTime,
    csSDK_int32* outVideoNodeID,
    PrTime* outSegmentOffset);

// Extended version with segment boundaries (version 9+)
prSuiteError (*AcquireFirstNodeInTimeRangeExt)(
    csSDK_int32 inVideoSegmentsID,
    PrTime inStartTime, PrTime inEndTime,
    csSDK_int32* outVideoNodeID,
    PrTime* outSegmentStartTime,
    PrTime* outSegmentEndTime,
    PrTime* outSegmentOffset);

// Release when done
prSuiteError (*ReleaseVideoNodeID)(csSDK_int32 inVideoNodeID);
```

### Inspecting Node Info

```c
prSuiteError (*GetNodeInfo)(
    csSDK_int32 inVideoNodeID,
    char* outNodeType,         // Buffer of kMaxNodeTypeStringSize (256)
    prPluginID* outHash,       // May differ from acquisition hash
    csSDK_int32* outFlags);    // PrNodeInfoFlag bitmask
```

Node info flags (`PrNodeInfoFlag`):

| Flag | Meaning |
|------|---------|
| `kPrNodeInfoFlag_IsCompletelyOpaque` | Node produces fully opaque output |
| `kPrNodeInfoFlag_IsCompletelyTransparent` | Node produces fully transparent output |
| `kPrNodeInfoFlag_DoesNotDependOnSequenceTime` | Output is time-independent |
| `kPrNodeInfoFlag_NodeDoesNotDependOnSegmentTime` | Output does not vary within segment |
| `kPrNodeInfoFlag_NodeDoesNotDependOnClipInOutTime` | Not affected by clip trimming |
| `kPrNodeInfoFlag_IsNOP` | Node is a no-op pass-through |
| `kPrNodeInfoFlag_NodeDoesNotDependOnTrackInputs` | Does not read from lower tracks |
| `kPrNodeInfoFlag_IsAdjustment` | Node is an adjustment layer |

### Traversing the Node Tree

**Inputs** are child nodes that provide source frames:

```c
prSuiteError (*GetNodeInputCount)(csSDK_int32 inVideoNodeID, csSDK_int32* outNumInputs);

prSuiteError (*AcquireInputNodeID)(
    csSDK_int32 inVideoNodeID,
    csSDK_int32 inIndex,
    PrTime* outOffset,           // Time offset relative to parent
    csSDK_int32* outInputVideoNodeID);  // Must be released
```

**Operators** are effects applied to a node's output:

```c
prSuiteError (*GetNodeOperatorCount)(csSDK_int32 inVideoNodeID, csSDK_int32* outNumOperators);

prSuiteError (*AcquireOperatorNodeID)(
    csSDK_int32 inVideoNodeID,
    csSDK_int32 inIndex,
    csSDK_int32* outOperatorVideoNodeID);  // Must be released

// Get the owner of an operator (version 6+)
prSuiteError (*AcquireOperatorOwnerNodeID)(
    csSDK_int32 inVideoNodeID,
    csSDK_int32* outOwnerNodeID);
```

### Node Properties

Properties are key-value pairs stored on nodes. You can iterate all properties or get a specific one:

```c
// Iterate all
prSuiteError (*IterateNodeProperties)(
    csSDK_int32 inVideoNodeID,
    SegmentNodePropertyCallback inCallback,
    csSDK_int32 inPluginObject);

// Where the callback signature is:
typedef prSuiteError (*SegmentNodePropertyCallback)(
    csSDK_int32 inPluginObject,
    const char* inKey,
    const prUTF8Char* inValue);

// Get a specific property
prSuiteError (*GetNodeProperty)(
    csSDK_int32 inVideoNodeID,
    const char* inKey,
    PrMemoryPtr* outValue);   // Allocated with PrNewPtr, caller must dispose
```

### Parameters and Keyframes

```c
prSuiteError (*GetParamCount)(csSDK_int32 inVideoNodeID, csSDK_int32* outParamCount);

prSuiteError (*GetParam)(
    csSDK_int32 inVideoNodeID,
    csSDK_int32 inIndex,
    PrTime inTime,           // In media time
    PrParam* outParam);

prSuiteError (*GetNextKeyframeTime)(
    csSDK_int32 inVideoNodeID,
    csSDK_int32 inIndex,
    PrTime inTime,
    PrTime* outKeyframeTime,
    csSDK_int32* outKeyframeInterpolationMode);
```

Keyframe interpolation modes (`PrKeyframeInterpolationModeFlag`):

| Value | Mode |
|-------|------|
| 0 | Linear |
| 4 | Hold |
| 5 | Bezier |
| 6 | Time |
| 7 | Time Transition Start |
| 8 | Time Transition End |

### Time Transforms

Clip nodes can have speed changes, reverse playback, and time remapping. To convert a node-local time to the time seen by its inputs:

```c
prSuiteError (*TransformNodeTime)(
    csSDK_int32 inVideoNodeID,
    PrTime inTime,
    PrTime* outTime);

// Get the instantaneous rate of change (version 9+)
prSuiteError (*GetNodeTimeScale)(
    csSDK_int32 inVideoNodeID,
    PrTime inTime,        // In untransformed media time
    double* outRate);     // Rate relative to containing sequence
```

### Graphics Transforms (Version 7+)

For nodes with Essential Graphics-style transforms:

```c
prSuiteError (*GetGraphicsTransformedParams)(
    csSDK_int32 inVideoNodeID,
    PrTime inTime,
    prFPoint64* outPosition,
    prFPoint64* outAnchor,
    prFPoint64* outScale,
    float* outRotation);

prSuiteError (*HasGraphicsGroup)(csSDK_int32 inVideoNodeID, bool* outHasGraphicGroup);
prSuiteError (*GetGraphicsGroupID)(csSDK_int32 inVideoNodeID, csSDK_int32* outGroupID);
```

---

## PrSDKVideoSegmentProperties

This header defines all property key strings. No property is ever guaranteed to be present on a node. Properties are categorized by node type:

### Media Node Properties

| Key | Type | Description |
|-----|------|-------------|
| `"MediaNode::ClipID"` | int32 | Clip identifier |
| `"MediaNode::MediaInstanceString"` | string | Media instance identifier |
| `"MediaNode::StreamFrameRate"` | PrTime | Source frame rate |
| `"MediaNode::StreamFrameWidth"` | int32 | Source frame width |
| `"MediaNode::StreamFrameHeight"` | int32 | Source frame height |
| `"MediaNode::StreamAlphaType"` | int32 | Alpha type of source |
| `"MediaNode::StreamFieldType"` | int32 | Field type of source |
| `"MediaNode::StreamColorSpace"` | string | Source color space |
| `"MediaNode::ClipSpeed"` | float64 | Playback speed multiplier |
| `"MediaNode::ClipBackwards"` | bool | Reverse playback |
| `"MediaNode::StreamFrameBlend"` | bool | Frame blending enabled |
| `"MediaNode::StreamTimeInterpolationType"` | uint32 | Time interpolation method |
| `"MediaNode::ClipScaleToFrameSize"` | bool | Auto-scale to frame size |
| `"MediaNode::ClipScaleToFramePolicy"` | int | Scale policy enum |
| `"MediaNode::ContentStart"` | PrTime | Media content start time |
| `"MediaNode::ContentEnd"` | PrTime | Media content end time |
| `"MediaNode::InPointMediaTimeAsTicks"` | int64 | Media in point in ticks |
| `"MediaNode::OutPointMediaTimeAsTicks"` | int64 | Media out point in ticks |
| `"MediaNode::NestedSequenceTimelineID"` | int32 | Timeline ID if nested sequence |

Scale policy values (`PrNodeScalePolicy`):

| Value | Policy |
|-------|--------|
| 0 | None |
| 1 | Scale to Frame (letterbox/pillarbox) |
| 2 | Scale to Fill with Crop |
| 3 | Scale to Fill with Distortion |

### Clip Node Properties

| Key | Type | Description |
|-----|------|-------------|
| `"ClipNode::ClipSpeed"` | float64 | Speed |
| `"ClipNode::ClipBackwards"` | bool | Reversed |
| `"ClipNode::TimeRemapping"` | keyframe data | Time remap curve |
| `"ClipNode::FrameHoldAtTime"` | PrTime | Frame hold time |
| `"ClipNode::FrameHoldFilters"` | bool | Hold filters at hold time |
| `"ClipNode::TrackID"` | int32 | Track identifier |
| `"ClipNode::TrackItemStartAsTicks"` | int64 | Track item start in sequence time |
| `"ClipNode::TrackItemEndAsTicks"` | int64 | Track item end in sequence time |
| `"ClipNode::AllowLinearCompositing"` | bool | Allow linear compositing (only set if false) |
| `"ClipNode::ToneMapSettings"` | string | Tone mapping configuration |
| `"ClipNode::UntrimmedDuration"` | int64 | Full duration before trimming |

### Effect Node Properties

| Key | Type | Description |
|-----|------|-------------|
| `"EffectNode::FilterMatchName"` | string | Effect match name |
| `"EffectNode::FilterCategoryName"` | string | Effect category |
| `"EffectNode::FilterOpaqueData"` | binary | Effect-specific data |
| `"EffectNode::EffectDuration"` | PrTime | Duration of the effect |
| `"EffectNode::RuntimeInstanceID"` | uint32 | Runtime instance identifier |
| `"EffectNode::StreamLabel"` | string | Stream label |

### Transition Node Properties

| Key | Type | Description |
|-----|------|-------------|
| `"TransitionNode::TransitionMatchName"` | string | Transition match name |
| `"TransitionNode::TransitionStartPercent"` | float32 | Start completion percentage |
| `"TransitionNode::TransitionEndPercent"` | float32 | End completion percentage |
| `"TransitionNode::TransitionDuration"` | PrTime | Transition duration |
| `"TransitionNode::TransitionSwitchSources"` | bool | Sources swapped |
| `"TransitionNode::TransitionReverse"` | bool | Reversed direction |
| `"TransitionNode::TransitionBorderWidth"` | float32 | Border width |
| `"TransitionNode::TransitionBorderColor"` | int32 | Border color |

---

## PrSDKVideoSegmentRenderSuite

Suite name: `"MediaCore Video Segment Render Suite"` (version 7)

This suite enables asynchronous rendering of individual nodes in the segment tree. It is primarily used by accelerated renderers and play modules that need to render specific portions of the timeline.

### Async Frame Production

```c
typedef void (*PrSDKVideoSegmentAsyncRenderCompletionProc)(
    PPixHand inRenderedFrame,    // Must be disposed via ppixSuite->Dispose()
    csSDK_int64 inCompletionData,
    prSuiteError inResult);

prSuiteError (*ProduceFrameAsync)(
    PrTimelineID inTimelineID,
    csSDK_int32  inNodeID,           // Any node except Effect nodes
    PrTime       inSequenceTime,     // For filters requesting other media
    PrTime       inSegmentTime,      // Time relative to the node
    PrTime       inSequenceTicksPerFrame,
    csSDK_int32  inSequenceWidth,    // 0 = no override
    csSDK_int32  inSequenceHeight,   // 0 = no override
    csSDK_int32  inSequencePixelAspectRatioNumerator,
    csSDK_int32  inSequencePixelAspectRatioDenominator,
    const SequenceRender_ParamsRec* inRenderParams,
    PrSDKVideoSegmentAsyncRenderCompletionProc inCompletionProc,
    csSDK_int64  inAsyncCompletionData,
    csSDK_int32* outRequestID);
```

**Important:** Effect nodes are not suitable for `ProduceFrameAsync`. In general, inputs (clips, media) work; operators (effects) do not. For effects, use `ApplyOperatorsToFrameAsync`.

### Applying Effects to Frames

```c
prSuiteError (*ApplyOperatorsToFrameAsync)(
    PrTimelineID inTimelineID,
    csSDK_int32  inClipNodeID,           // The clip containing the operator
    csSDK_int32  inOperatorStartIndex,   // Zero-based operator index
    csSDK_int32  inOperatorCount,        // Number of operators to apply
    PrTime       inSequenceTime,
    PrTime       inSegmentTime,
    PrTime       inSequenceTicksPerFrame,
    csSDK_int32  inSequenceWidth,
    csSDK_int32  inSequenceHeight,
    csSDK_int32  inSequencePixelAspectRatioNumerator,
    csSDK_int32  inSequencePixelAspectRatioDenominator,
    PPixHand     inInputFrame,           // NULL if start index is 0
    const SequenceRender_ParamsRec* inRenderParams,
    PrSDKVideoSegmentAsyncRenderCompletionProc inCompletionProc,
    csSDK_int64  inAsyncCompletionData,
    csSDK_int32* outRequestID);
```

If `inInputFrame` is NULL and `inOperatorStartIndex` is 0, the host will provide the clip node's input frame automatically.

### Applying Transitions

```c
prSuiteError (*ApplyTransitionToFrameAsync)(
    PrTimelineID inTimelineID,
    csSDK_int32  inTransitionNodeID,      // Must be a transition node
    PrTime       inSequenceTime,
    PrTime       inSegmentTime,
    PrTime       inSequenceTicksPerFrame,
    PPixHand     inOutgoingInputFrame,    // NULL = transparent black
    PPixHand     inIncomingInputFrame,    // NULL = transparent black
    const SequenceRender_ParamsRec* inRenderParams,
    PrSDKVideoSegmentAsyncRenderCompletionProc inCompletionProc,
    csSDK_int64  inAsyncCompletionData,
    csSDK_int32* outRequestID);
```

### Cache Checking

Before initiating an async render, check if the result is already cached:

```c
prSuiteError (*GetIdentifierForProduceFrameAsync)(/* same params minus completion */ prPluginID* outIdentifier);
prSuiteError (*GetIdentifierForApplyOperatorsToFrameAsync)(/* ... */ prPluginID* outIdentifier);
prSuiteError (*GetIdentifierForApplyTransitionToFrameAsync)(/* ... */ prPluginID* outIdentifier);
```

### Clip Prefetching

For direct media access from a specific clip:

```c
prSuiteError (*SelectClipFrameDescriptor)(
    PrClipID inClipID,
    PrTime inClipTime,
    const ClipFrameDescriptor* inDesiredClipFrameDescriptor,
    ClipFrameDescriptor* outBestFrameDescriptor);

prSuiteError (*InitiateClipPrefetch)(
    PrClipID inClipID,
    const ClipFrameDescriptor* inRequestedFrameDescriptor,
    PrTime inMediaTime,
    PrSDKVideoSegmentAsyncRenderCompletionProc inCompletionProc,
    csSDK_int64 inAsyncCompletionData,
    csSDK_int32* outRequestID);

prSuiteError (*CancelAsyncRequest)(csSDK_int32 inRequestID);
```

---

## PrSDKClipRenderSuite

Suite name: `"Premiere Clip Render Suite"` (version 3)

This suite provides direct access to clip-level rendering, bypassing the full segment tree. It is useful for importers and plugins that need to negotiate pixel formats and frame sizes with the source media.

### Capability Check

```c
prSuiteError (*SupportsClipRenderSuite)(
    PrClipID inClipID,
    prBool* outSupported,
    prBool* outAsyncIOSupported);  // Can be NULL
```

### Pixel Format Negotiation

```c
prSuiteError (*GetNumPixelFormats)(PrClipID inClipID, csSDK_int32* outNumPixelFormats);
prSuiteError (*GetPixelFormat)(PrClipID inClipID, csSDK_int32 inIndex, PrPixelFormat* outPixelFormat);

// Custom formats (version 3+)
prSuiteError (*GetNumCustomPixelFormats)(PrClipID inClipID, csSDK_int32* outNumPixelFormats);
prSuiteError (*GetCustomPixelFormat)(PrClipID inClipID, csSDK_int32 inIndex, PrPixelFormat* outPixelFormat);
```

Pixel formats are returned in preference order (index 0 = most preferred).

### Frame Size Negotiation

```c
prSuiteError (*GetNumPreferredFrameSizes)(
    PrClipID inClipID,
    PrPixelFormat inPixelFormat,
    csSDK_int32* outNumPreferredFrameSizes);

prSuiteError (*GetPreferredFrameSize)(
    PrClipID inClipID,
    PrPixelFormat inPixelFormat,
    csSDK_int32 inIndex,
    csSDK_int32* outWidth,    // 0 = any width
    csSDK_int32* outHeight);  // 0 = any height
```

### Asynchronous I/O

```c
prSuiteError (*InitiateAsyncRead)(
    PrClipID inClipID,
    const PrTime* inFrameTime,
    ClipFrameFormat* inFormat);

prSuiteError (*CancelAsyncRead)(
    PrClipID inClipID,
    const PrTime* inFrameTime,
    ClipFrameFormat* inFormat);
```

### Frame Retrieval

```c
prSuiteError (*FindFrame)(
    PrClipID inClipID,
    const PrTime* inFrameTime,
    ClipFrameFormat* inFormats,    // Array of acceptable formats (or NULL)
    csSDK_int32 inNumFormats,      // 0 if inFormats is NULL
    bool inSynchronous,            // true = will read from disk if needed
    PPixHand* outFrame);           // NULL if not found
```

`FindFrame` first looks in cache for the requested formats. If not found and `inSynchronous` is false, it will only decode from cached raw data (no disk access). If `inSynchronous` is true, it will hit disk.

### Field Type

```c
prSuiteError (*GetClipFieldType)(PrClipID inClipID, prFieldType* outFieldType);
```

---

## Practical Walkthrough: Traversing the Segment Tree

Here is a complete example of walking the video segment tree for a sequence:

```cpp
void InspectTimeline(PrTimelineID timelineID, PrSDKVideoSegmentSuite* segSuite)
{
    csSDK_int32 videoSegmentsID = 0;
    segSuite->AcquireVideoSegmentsID(timelineID, &videoSegmentsID);

    csSDK_int32 numSegments = 0;
    segSuite->GetSegmentCount(videoSegmentsID, &numSegments);

    for (csSDK_int32 i = 0; i < numSegments; i++)
    {
        PrTime startTime, endTime, offset;
        prPluginID hash;
        segSuite->GetSegmentInfo(videoSegmentsID, i, &startTime, &endTime, &offset, &hash);

        // Acquire the root node for this segment
        csSDK_int32 rootNodeID = 0;
        segSuite->AcquireNodeID(videoSegmentsID, &hash, &rootNodeID);

        // Recursively inspect
        InspectNode(rootNodeID, segSuite, 0);

        segSuite->ReleaseVideoNodeID(rootNodeID);
    }

    segSuite->ReleaseVideoSegmentsID(videoSegmentsID);
}

void InspectNode(csSDK_int32 nodeID, PrSDKVideoSegmentSuite* segSuite, int depth)
{
    char nodeType[kMaxNodeTypeStringSize];
    prPluginID hash;
    csSDK_int32 flags;
    segSuite->GetNodeInfo(nodeID, nodeType, &hash, &flags);

    // Log: nodeType, flags, depth

    // Check operators (effects)
    csSDK_int32 numOperators = 0;
    segSuite->GetNodeOperatorCount(nodeID, &numOperators);
    for (csSDK_int32 i = 0; i < numOperators; i++)
    {
        csSDK_int32 operatorNodeID = 0;
        segSuite->AcquireOperatorNodeID(nodeID, i, &operatorNodeID);
        InspectNode(operatorNodeID, segSuite, depth + 1);
        segSuite->ReleaseVideoNodeID(operatorNodeID);
    }

    // Check inputs (sources)
    csSDK_int32 numInputs = 0;
    segSuite->GetNodeInputCount(nodeID, &numInputs);
    for (csSDK_int32 i = 0; i < numInputs; i++)
    {
        PrTime inputOffset;
        csSDK_int32 inputNodeID = 0;
        segSuite->AcquireInputNodeID(nodeID, i, &inputOffset, &inputNodeID);
        InspectNode(inputNodeID, segSuite, depth + 1);
        segSuite->ReleaseVideoNodeID(inputNodeID);
    }
}
```

---

## How Segments Affect Effect Rendering and Caching

### Segment Hashes and Caching

Each segment has a unique hash. When the timeline changes (clips moved, effects added, parameters changed), segments are recalculated and their hashes change. This is the primary cache invalidation mechanism -- an accelerated renderer can compare segment hashes to determine which portions of the timeline need re-rendering.

### Effect Rendering Order

Effects are applied as operators on clip nodes. The operator index determines the order:
- Operator 0 is applied first (closest to the source)
- Higher indices are applied later (closer to the output)

This matches the order effects appear in the Effect Controls panel (top to bottom = index 0, 1, 2...).

### Transitions and the Segment Boundary

Transitions create their own segment because the rendering graph changes structure. During a transition, the segment's root compositor has both the outgoing and incoming clips as inputs, with the transition node processing them. The transition's `outReadIncomingTransition` flag in `PrGPUFilterFrameDependency` controls which clip of the transition provides the input frame.

---

## Pitfalls and Warnings

**Ref-counting is mandatory.** Every `Acquire*` call must have a matching `Release*`. Leaking node or segment IDs will cause memory leaks and potentially stale data.

**Properties are not guaranteed.** Always check the return value of `GetNodeProperty`. The absence of a property is not an error -- it simply means the default applies.

**Time domains are confusing.** Segment time, sequence time, clip time, and media time are all different. Use `TransformNodeTime` to convert between them. The `GetParam` function expects media time, while `GetSegmentInfo` returns sequence time.

**String properties are allocated with `PrNewPtr`.** You must dispose them with `PrDisposePtr` from the memory manager suite. Failing to do so leaks memory.

**The segment tree can change between calls.** If the user edits the timeline, your previously acquired segment IDs and node IDs may become invalid. Always re-acquire when responding to new render requests.

**`PrClipSegmentInfo::mClipPath` has a short lifetime.** The `PrSDKString` in this struct is disposed after the callback returns from `BuildSmartRenderSegmentList`. If you need to keep the path, copy it to your own storage and re-allocate.
