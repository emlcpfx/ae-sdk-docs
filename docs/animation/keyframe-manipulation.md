# Keyframe Manipulation via AEGP

This document covers programmatic keyframe manipulation using `AEGP_KeyframeSuite5`, including inserting, removing, and modifying keyframes, controlling interpolation types, tangents, easing, and batch keyframe operations.

> **Suite**: `AEGP_KeyframeSuite5` (version 5, frozen in AE 22.5)
> **Header**: `AE_GeneralPlug.h`
> **Prerequisite**: You must have an `AEGP_StreamRefH` for the property you want to keyframe. Obtain one via `AEGP_StreamSuite6::AEGP_GetNewLayerStream` or `AEGP_GetNewEffectStreamByIndex`.

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Getting a Stream Reference](#getting-a-stream-reference)
- [Querying Keyframes](#querying-keyframes)
- [Inserting Keyframes](#inserting-keyframes)
- [Reading Keyframe Values](#reading-keyframe-values)
- [Setting Keyframe Values](#setting-keyframe-values)
- [Interpolation Types](#interpolation-types)
- [Temporal Ease (Speed Curves)](#temporal-ease-speed-curves)
- [Spatial Tangents](#spatial-tangents)
- [Keyframe Flags](#keyframe-flags)
- [Keyframe Labels](#keyframe-labels)
- [Batch Keyframe Addition](#batch-keyframe-addition)
- [Deleting Keyframes](#deleting-keyframes)
- [Reading Values at Arbitrary Times](#reading-values-at-arbitrary-times)
- [Complete Example: Animating Position](#complete-example-animating-position)
- [Pitfalls and Warnings](#pitfalls-and-warnings)

---

## Core Concepts

Every animatable property in After Effects is represented as a **stream** (`AEGP_StreamRefH`). Keyframes are accessed by **index** (`AEGP_KeyframeIndex`, which is `A_long`). Indices are zero-based and re-number when keyframes are inserted or deleted.

Times are always specified as `A_Time` (numerator/denominator rational time) with an `AEGP_LTimeMode` indicating whether the time is in **layer time** or **comp time**:

```cpp
enum {
    AEGP_LTimeMode_LayerTime,
    AEGP_LTimeMode_CompTime
};
```

All keyframe modification functions marked `/* UNDOABLE */` in the header will participate in the undo system. Wrap related operations in an undo group via `AEGP_UtilitySuite` for a clean user experience.

---

## Getting a Stream Reference

Before manipulating keyframes you need an `AEGP_StreamRefH`. The most common path is through `AEGP_StreamSuite6`:

```cpp
AEGP_StreamRefH position_streamH = NULL;

// Get the position stream from a layer
ERR(stream_suite->AEGP_GetNewLayerStream(
    plugin_id,
    layerH,
    AEGP_LayerStream_POSITION,
    &position_streamH));
```

For effect parameters, use the effect stream approach:

```cpp
AEGP_StreamRefH param_streamH = NULL;

ERR(stream_suite->AEGP_GetNewEffectStreamByIndex(
    plugin_id,
    effect_refH,
    param_index,       // 0 = input layer, 1..n = your params
    &param_streamH));
```

> **Important**: You must dispose of stream references with `AEGP_DisposeStream` when done. Failure to do so leaks memory.

---

## Querying Keyframes

### Get Number of Keyframes

```cpp
A_long num_kfs = 0;
ERR(kf_suite->AEGP_GetStreamNumKFs(streamH, &num_kfs));
```

Special return values:
- `0` -- no keyframes (property may still have an expression, so not necessarily constant)
- `AEGP_NumKF_NO_DATA` -- stream type is `AEGP_StreamType_NO_DATA`; you cannot retrieve values

### Get Keyframe Time

```cpp
A_Time kf_time;
ERR(kf_suite->AEGP_GetKeyframeTime(
    streamH,
    key_index,              // 0-based
    AEGP_LTimeMode_CompTime,
    &kf_time));
```

---

## Inserting Keyframes

```cpp
AEGP_KeyframeIndex new_index;
A_Time insert_time = {0, 1};  // time 0

ERR(kf_suite->AEGP_InsertKeyframe(
    streamH,                    // <> stream is modified
    AEGP_LTimeMode_CompTime,
    &insert_time,
    &new_index));               // << index of the new (or existing) keyframe
```

If a keyframe already exists at the specified time, the stream is left unchanged and `new_index` receives the index of the existing keyframe. This makes it safe to call without checking first.

> **Warning**: After inserting a keyframe, all indices at or after the insertion point may shift. Do not cache keyframe indices across insertion calls.

---

## Reading Keyframe Values

```cpp
AEGP_StreamValue2 value;

ERR(kf_suite->AEGP_GetNewKeyframeValue(
    plugin_id,
    streamH,
    key_index,
    &value));

// Use the value based on stream type
// For a 1D stream:
A_FpLong one_d_val = value.val.one_d;

// For a 3D spatial stream (e.g., Position):
A_FpLong x = value.val.three_d.x;
A_FpLong y = value.val.three_d.y;
A_FpLong z = value.val.three_d.z;

// MUST dispose when done
ERR(stream_suite->AEGP_DisposeStreamValue(&value));
```

### Stream Value Union

The value is stored in an `AEGP_StreamVal2` union. Use the correct field based on the stream type:

| Stream Type | Union Field | Data Type |
|---|---|---|
| `AEGP_StreamType_OneD` | `val.one_d` | `A_FpLong` |
| `AEGP_StreamType_TwoD` | `val.two_d` | `AEGP_TwoDVal` (x, y) |
| `AEGP_StreamType_TwoD_SPATIAL` | `val.two_d` | `AEGP_TwoDVal` (x, y) |
| `AEGP_StreamType_ThreeD` | `val.three_d` | `AEGP_ThreeDVal` (x, y, z) |
| `AEGP_StreamType_ThreeD_SPATIAL` | `val.three_d` | `AEGP_ThreeDVal` (x, y, z) |
| `AEGP_StreamType_COLOR` | `val.color` | `AEGP_ColorVal` (a, r, g, b as `A_FpLong` 0-1) |
| `AEGP_StreamType_ARB` | `val.arbH` | `A_Handle` |
| `AEGP_StreamType_LAYER_ID` | `val.layer_id` | `AEGP_LayerIDVal` (A_long) |
| `AEGP_StreamType_MARKER` | `val.markerP` | `AEGP_MarkerValP` |
| `AEGP_StreamType_TEXT_DOCUMENT` | `val.text_documentH` | `AEGP_TextDocumentH` |

You can query the stream type at runtime:

```cpp
AEGP_StreamType stream_type;
ERR(stream_suite->AEGP_GetStreamType(streamH, &stream_type));
```

---

## Setting Keyframe Values

```cpp
AEGP_StreamValue2 value;
memset(&value, 0, sizeof(value));
value.streamH = streamH;
value.val.three_d.x = 100.0;
value.val.three_d.y = 200.0;
value.val.three_d.z = 0.0;

ERR(kf_suite->AEGP_SetKeyframeValue(
    streamH,
    key_index,
    &value));           // value is NOT adopted; you still own it
```

> **Note**: `AEGP_SetKeyframeValue` does not adopt the value. You do not need to dispose it after setting, but you should not reuse stream value objects obtained from `AEGP_GetNewKeyframeValue` without disposing them first.

---

## Interpolation Types

Each keyframe has separate **incoming** and **outgoing** interpolation types:

```cpp
enum {
    AEGP_KeyInterp_NONE = 0,
    AEGP_KeyInterp_LINEAR,
    AEGP_KeyInterp_BEZIER,
    AEGP_KeyInterp_HOLD,
};
```

### Get Interpolation

```cpp
AEGP_KeyframeInterpolationType in_interp, out_interp;

ERR(kf_suite->AEGP_GetKeyframeInterpolation(
    streamH,
    key_index,
    &in_interp,         // incoming interpolation
    &out_interp));       // outgoing interpolation
```

### Set Interpolation

```cpp
// Set to bezier in, hold out (value holds until next keyframe)
ERR(kf_suite->AEGP_SetKeyframeInterpolation(
    streamH,
    key_index,
    AEGP_KeyInterp_BEZIER,   // in
    AEGP_KeyInterp_HOLD));   // out
```

### Interpolation Behavior Summary

| Type | Behavior |
|---|---|
| `AEGP_KeyInterp_LINEAR` | Straight-line interpolation between keyframes |
| `AEGP_KeyInterp_BEZIER` | Smooth curve controlled by temporal ease handles |
| `AEGP_KeyInterp_HOLD` | Value stays constant until the next keyframe (step function) |

### Checking Valid Interpolation Types

Not all streams support all interpolation types. Check before setting:

```cpp
AEGP_KeyInterpolationMask valid_interps;
ERR(stream_suite->AEGP_GetValidInterpolations(streamH, &valid_interps));

if (valid_interps & AEGP_KeyInterpMask_BEZIER) {
    // Bezier interpolation is supported
}
```

---

## Temporal Ease (Speed Curves)

Temporal ease controls the speed at which a value changes approaching (incoming) and leaving (outgoing) a keyframe. Each dimension of the stream has independent ease values.

```cpp
typedef struct {
    A_FpLong    speedF;      // units per second at the keyframe
    A_FpLong    influenceF;  // 0-100, percentage of temporal influence
} AEGP_KeyframeEase;
```

### Get Temporal Ease

```cpp
// Get dimensionality first
A_short temporal_dims;
ERR(kf_suite->AEGP_GetStreamTemporalDimensionality(streamH, &temporal_dims));

// For each dimension (usually 1 for most properties)
for (A_long d = 0; d < temporal_dims; d++) {
    AEGP_KeyframeEase in_ease, out_ease;
    ERR(kf_suite->AEGP_GetKeyframeTemporalEase(
        streamH,
        key_index,
        d,              // dimension 0..temporal_dims-1
        &in_ease,
        &out_ease));
}
```

### Set Temporal Ease

```cpp
AEGP_KeyframeEase in_ease, out_ease;

// "Easy ease" style: 33% influence, auto speed
in_ease.speedF = 0.0;
in_ease.influenceF = 33.33;
out_ease.speedF = 0.0;
out_ease.influenceF = 33.33;

ERR(kf_suite->AEGP_SetKeyframeTemporalEase(
    streamH,
    key_index,
    0,                  // dimension
    &in_ease,
    &out_ease));
```

### Dimensionality

| Function | Returns |
|---|---|
| `AEGP_GetStreamValueDimensionality` | Number of value components (e.g., 3 for Position XYZ) |
| `AEGP_GetStreamTemporalDimensionality` | Number of independent temporal ease dimensions (often 1, even for multi-dimensional values, unless dimensions are separated) |

> **Warning**: Temporal dimensionality is not always the same as value dimensionality. For a 3D position with separated dimensions, temporal dimensionality is 3. For an unseparated 3D position, it is 1.

---

## Spatial Tangents

Spatial tangents control the shape of the motion path in the composition viewer. They only apply to spatial stream types (`AEGP_StreamType_TwoD_SPATIAL`, `AEGP_StreamType_ThreeD_SPATIAL`).

### Get Spatial Tangents

```cpp
AEGP_StreamValue2 in_tan, out_tan;

ERR(kf_suite->AEGP_GetNewKeyframeSpatialTangents(
    plugin_id,
    streamH,
    key_index,
    &in_tan,            // incoming tangent
    &out_tan));          // outgoing tangent

// For a 3D spatial stream, tangent values are offsets from the keyframe value:
A_FpLong in_tan_x = in_tan.val.three_d.x;
A_FpLong in_tan_y = in_tan.val.three_d.y;
A_FpLong in_tan_z = in_tan.val.three_d.z;

// Dispose both
ERR(stream_suite->AEGP_DisposeStreamValue(&in_tan));
ERR(stream_suite->AEGP_DisposeStreamValue(&out_tan));
```

### Set Spatial Tangents

```cpp
AEGP_StreamValue2 in_tan, out_tan;
memset(&in_tan, 0, sizeof(in_tan));
memset(&out_tan, 0, sizeof(out_tan));
in_tan.streamH = streamH;
out_tan.streamH = streamH;

// Set tangents as offsets from the keyframe position
in_tan.val.three_d.x = -50.0;
in_tan.val.three_d.y = 0.0;
out_tan.val.three_d.x = 50.0;
out_tan.val.three_d.y = 0.0;

ERR(kf_suite->AEGP_SetKeyframeSpatialTangents(
    streamH,
    key_index,
    &in_tan,
    &out_tan));
```

> **Note**: Spatial tangent values are NOT adopted by After Effects. You retain ownership and do not need to dispose values you construct yourself (only dispose those returned by `AEGP_GetNewKeyframeSpatialTangents`).

---

## Keyframe Flags

Keyframe flags control continuity and roving behavior:

```cpp
enum {
    AEGP_KeyframeFlag_NONE                  = 0x00,
    AEGP_KeyframeFlag_TEMPORAL_CONTINUOUS   = 0x01,
    AEGP_KeyframeFlag_TEMPORAL_AUTOBEZIER   = 0x02,
    AEGP_KeyframeFlag_SPATIAL_CONTINUOUS    = 0x04,
    AEGP_KeyframeFlag_SPATIAL_AUTOBEZIER   = 0x08,
    AEGP_KeyframeFlag_ROVING               = 0x10
};
```

| Flag | Meaning |
|---|---|
| `TEMPORAL_CONTINUOUS` | Incoming and outgoing temporal ease are linked (continuous speed curve) |
| `TEMPORAL_AUTOBEZIER` | AE automatically adjusts temporal tangents for smoothness |
| `SPATIAL_CONTINUOUS` | Incoming and outgoing spatial tangents are linked (smooth motion path) |
| `SPATIAL_AUTOBEZIER` | AE automatically adjusts spatial tangents for smoothness |
| `ROVING` | Keyframe timing auto-adjusts to maintain even spatial velocity |

### Get Flags

```cpp
AEGP_KeyframeFlags flags;
ERR(kf_suite->AEGP_GetKeyframeFlags(streamH, key_index, &flags));

if (flags & AEGP_KeyframeFlag_ROVING) {
    // This keyframe is roving
}
```

### Set Flags

Set one flag at a time:

```cpp
// Enable roving
ERR(kf_suite->AEGP_SetKeyframeFlag(
    streamH,
    key_index,
    AEGP_KeyframeFlag_ROVING,
    TRUE));

// Disable spatial auto-bezier
ERR(kf_suite->AEGP_SetKeyframeFlag(
    streamH,
    key_index,
    AEGP_KeyframeFlag_SPATIAL_AUTOBEZIER,
    FALSE));
```

> **Warning**: Setting flags one at a time means multiple calls are needed to configure a keyframe completely. Each call is undoable separately unless you wrap them in an undo group.

---

## Keyframe Labels

New in `AEGP_KeyframeSuite5` (AE 22.5+): keyframes can have color labels.

```cpp
// Get label
A_long label_index;
ERR(kf_suite->AEGP_GetKeyframeLabelColorIndex(streamH, key_index, &label_index));

// Set label
ERR(kf_suite->AEGP_SetKeyframeLabelColorIndex(streamH, key_index, 3));  // label index 3
```

---

## Batch Keyframe Addition

When adding many keyframes at once, the standard `AEGP_InsertKeyframe` + `AEGP_SetKeyframeValue` approach is slow because the stream is re-sorted after each insertion. Use the batch API instead:

```cpp
AEGP_AddKeyframesInfoH akH = NULL;

// 1. Begin batch mode
ERR(kf_suite->AEGP_StartAddKeyframes(streamH, &akH));

// 2. Add keyframes (they are not committed yet)
for (int i = 0; i < num_keyframes; i++) {
    A_long kf_index;
    A_Time t = {i * 30, 30};  // one per frame at 30fps

    ERR(kf_suite->AEGP_AddKeyframes(
        akH,
        AEGP_LTimeMode_CompTime,
        &t,
        &kf_index));

    // 3. Set the value for this keyframe
    AEGP_StreamValue2 val;
    memset(&val, 0, sizeof(val));
    val.streamH = streamH;
    val.val.one_d = (A_FpLong)i * 10.0;

    ERR(kf_suite->AEGP_SetAddKeyframe(akH, kf_index, &val));
}

// 4. Commit all keyframes at once (UNDOABLE)
ERR(kf_suite->AEGP_EndAddKeyframes(
    TRUE,       // TRUE to commit, FALSE to cancel
    akH));
```

> **Important**: If you pass `FALSE` to `AEGP_EndAddKeyframes`, no keyframes are added and the `akH` is disposed. Always call `AEGP_EndAddKeyframes` to avoid leaking the info handle.

---

## Deleting Keyframes

```cpp
// Delete keyframe at index 2
ERR(kf_suite->AEGP_DeleteKeyframe(streamH, 2));
```

> **Warning**: After deletion, all keyframes at higher indices shift down by one. If you are deleting multiple keyframes, iterate in reverse order (highest index first) to avoid index invalidation.

```cpp
// Safe multi-delete: reverse order
A_long num_kfs;
ERR(kf_suite->AEGP_GetStreamNumKFs(streamH, &num_kfs));

for (A_long i = num_kfs - 1; i >= 0; i--) {
    // Check some condition...
    ERR(kf_suite->AEGP_DeleteKeyframe(streamH, i));
}
```

---

## Reading Values at Arbitrary Times

To get the interpolated value of a stream at any time (not just at keyframes), use `AEGP_StreamSuite6::AEGP_GetNewStreamValue`:

```cpp
AEGP_StreamValue2 value;
A_Time query_time = {15, 30};  // 0.5 seconds at 30fps

ERR(stream_suite->AEGP_GetNewStreamValue(
    plugin_id,
    streamH,
    AEGP_LTimeMode_CompTime,
    &query_time,
    FALSE,          // FALSE = evaluate expressions; TRUE = pre-expression value
    &value));

// Use value...

ERR(stream_suite->AEGP_DisposeStreamValue(&value));
```

The `pre_expressionB` parameter is critical:

| Value | Behavior |
|---|---|
| `FALSE` | Returns the final value after expression evaluation |
| `TRUE` | Returns the raw keyframe-interpolated value, ignoring any expression on the stream |

For a simpler API that does not require stream disposal (but only works with primitive types, not ARB/Marker/Mask):

```cpp
AEGP_StreamVal2 simple_val;
AEGP_StreamType stream_type;

ERR(stream_suite->AEGP_GetLayerStreamValue(
    layerH,
    AEGP_LayerStream_OPACITY,
    AEGP_LTimeMode_CompTime,
    &query_time,
    FALSE,              // pre_expressionB
    &simple_val,
    &stream_type));     // can be NULL if you don't need it

A_FpLong opacity = simple_val.one_d;
```

---

## Complete Example: Animating Position

This example creates a simple two-keyframe position animation on a layer:

```cpp
A_Err AnimateLayerPosition(
    AEGP_PluginID       plugin_id,
    AEGP_LayerH         layerH,
    AEGP_StreamSuite6   *ss,
    AEGP_KeyframeSuite5 *kfs)
{
    A_Err err = A_Err_NONE;
    AEGP_StreamRefH streamH = NULL;

    // Get position stream
    ERR(ss->AEGP_GetNewLayerStream(
        plugin_id, layerH,
        AEGP_LayerStream_POSITION, &streamH));

    if (!err && streamH) {
        // Insert keyframe at frame 0
        A_Time t0 = {0, 30};
        AEGP_KeyframeIndex idx0;
        ERR(kfs->AEGP_InsertKeyframe(streamH,
            AEGP_LTimeMode_CompTime, &t0, &idx0));

        // Set position to (100, 100)
        if (!err) {
            AEGP_StreamValue2 val0;
            memset(&val0, 0, sizeof(val0));
            val0.streamH = streamH;
            val0.val.three_d.x = 100.0;
            val0.val.three_d.y = 100.0;
            val0.val.three_d.z = 0.0;
            ERR(kfs->AEGP_SetKeyframeValue(streamH, idx0, &val0));
        }

        // Insert keyframe at frame 60 (2 seconds at 30fps)
        A_Time t1 = {60, 30};
        AEGP_KeyframeIndex idx1;
        if (!err) {
            ERR(kfs->AEGP_InsertKeyframe(streamH,
                AEGP_LTimeMode_CompTime, &t1, &idx1));
        }

        // Set position to (500, 300)
        if (!err) {
            AEGP_StreamValue2 val1;
            memset(&val1, 0, sizeof(val1));
            val1.streamH = streamH;
            val1.val.three_d.x = 500.0;
            val1.val.three_d.y = 300.0;
            val1.val.three_d.z = 0.0;
            ERR(kfs->AEGP_SetKeyframeValue(streamH, idx1, &val1));
        }

        // Set both to bezier interpolation with easy ease
        if (!err) {
            ERR(kfs->AEGP_SetKeyframeInterpolation(streamH, idx0,
                AEGP_KeyInterp_BEZIER, AEGP_KeyInterp_BEZIER));
            ERR(kfs->AEGP_SetKeyframeInterpolation(streamH, idx1,
                AEGP_KeyInterp_BEZIER, AEGP_KeyInterp_BEZIER));

            AEGP_KeyframeEase ease = {0.0, 33.33};
            ERR(kfs->AEGP_SetKeyframeTemporalEase(streamH, idx0, 0, &ease, &ease));
            ERR(kfs->AEGP_SetKeyframeTemporalEase(streamH, idx1, 0, &ease, &ease));
        }

        ERR(ss->AEGP_DisposeStream(streamH));
    }

    return err;
}
```

---

## Pitfalls and Warnings

1. **Always dispose stream references and stream values.** Every `AEGP_GetNewLayerStream`, `AEGP_GetNewEffectStreamByIndex`, `AEGP_GetNewKeyframeValue`, and `AEGP_GetNewKeyframeSpatialTangents` allocates memory that you must free.

2. **Keyframe indices are volatile.** Any insert or delete operation can renumber existing keyframes. Never cache indices across modification calls.

3. **Check for `AEGP_StreamType_NO_DATA`.** Some streams (e.g., group headers) return `AEGP_NumKF_NO_DATA` from `AEGP_GetStreamNumKFs`. Do not attempt to read or write keyframes on these streams.

4. **Undo grouping is your responsibility.** Wrap related keyframe operations in `AEGP_StartUndoGroup` / `AEGP_EndUndoGroup` (from `AEGP_UtilitySuite`) so they appear as a single undo step.

5. **Thread safety.** The AEGP keyframe APIs are not thread-safe. Call them only from the main thread or during AEGP callbacks that are guaranteed to be on the main thread.

6. **Batch add is significantly faster.** When adding more than a handful of keyframes, always use `AEGP_StartAddKeyframes` / `AEGP_AddKeyframes` / `AEGP_SetAddKeyframe` / `AEGP_EndAddKeyframes`. The standard insert-then-set approach triggers internal re-sorts on every insertion.

7. **Spatial tangents are offsets, not absolute positions.** The tangent values represent displacement from the keyframe value, not world-space coordinates.

8. **Setting interpolation on the last keyframe's outgoing side has no visible effect.** Likewise for the first keyframe's incoming side. AE ignores these, but setting them does not cause errors.

9. **Expression interaction.** If a stream has an expression, keyframe values still exist but may be overridden at evaluation time. Use `AEGP_GetNewStreamValue` with `pre_expressionB = TRUE` to read the pre-expression (keyframe-interpolated) value, or `FALSE` to see what the user actually sees after expression evaluation.

10. **`AEGP_SetStreamValue` vs keyframes.** `AEGP_SetStreamValue` from `AEGP_StreamSuite6` is only legal when the stream has zero keyframes. If keyframes exist, you must use `AEGP_InsertKeyframe` + `AEGP_SetKeyframeValue`.

---

## AEGP_KeyframeSuite5 API Reference

| Function | Description |
|---|---|
| `AEGP_GetStreamNumKFs` | Get number of keyframes on a stream |
| `AEGP_GetKeyframeTime` | Get time of a keyframe by index |
| `AEGP_InsertKeyframe` | Insert a keyframe at a time (no-op if one exists) |
| `AEGP_DeleteKeyframe` | Delete a keyframe by index |
| `AEGP_GetNewKeyframeValue` | Read a keyframe value (must dispose) |
| `AEGP_SetKeyframeValue` | Set a keyframe value |
| `AEGP_GetStreamValueDimensionality` | Number of value components |
| `AEGP_GetStreamTemporalDimensionality` | Number of independent temporal ease dimensions |
| `AEGP_GetNewKeyframeSpatialTangents` | Read spatial tangents (must dispose) |
| `AEGP_SetKeyframeSpatialTangents` | Set spatial tangents |
| `AEGP_GetKeyframeTemporalEase` | Read temporal ease per dimension |
| `AEGP_SetKeyframeTemporalEase` | Set temporal ease per dimension |
| `AEGP_GetKeyframeFlags` | Read keyframe flags |
| `AEGP_SetKeyframeFlag` | Set one keyframe flag |
| `AEGP_GetKeyframeInterpolation` | Read in/out interpolation types |
| `AEGP_SetKeyframeInterpolation` | Set in/out interpolation types |
| `AEGP_StartAddKeyframes` | Begin batch keyframe addition |
| `AEGP_AddKeyframes` | Add a keyframe in batch mode |
| `AEGP_SetAddKeyframe` | Set value for a batch-added keyframe |
| `AEGP_EndAddKeyframes` | Commit or cancel batch keyframes |
| `AEGP_GetKeyframeLabelColorIndex` | Get keyframe label color (AE 22.5+) |
| `AEGP_SetKeyframeLabelColorIndex` | Set keyframe label color (AE 22.5+) |
