# Premiere Pro Compatibility: Building Cross-Host Plugins

After Effects effect plugins (`.aex` files) can run inside Premiere Pro with no recompilation, but the two hosts have significant behavioral differences. This document covers every major divergence, from host detection through rendering pipeline differences, so you can build a single plugin binary that works correctly in both applications.

## Host Detection

The primary mechanism for detecting the host is `in_data->appl_id`, a 4-byte identifier available in every command selector:

| Host | `appl_id` Value | FourCC |
|------|-----------------|--------|
| After Effects | `'FXTC'` | 0x46585443 |
| Premiere Pro | `'PrMr'` | 0x50724D72 |

```cpp
static PF_Err GlobalSetup(PF_InData *in_data, PF_OutData *out_data) {
    PF_Err err = PF_Err_NONE;

    if (in_data->appl_id == 'PrMr') {
        // Premiere Pro specific setup
        out_data->out_flags = PREMIERE_OUT_FLAGS;
        out_data->out_flags2 = PREMIERE_OUT_FLAGS2;
    } else {
        // After Effects setup
        out_data->out_flags = AE_OUT_FLAGS;
        out_data->out_flags2 = AE_OUT_FLAGS2;
    }

    return err;
}
```

> **Pitfall**: Do not check `appl_id` from `PluginDataEntryFunction` -- that function does not receive `in_data`. Use `EffectMain` / `PF_Cmd_GLOBAL_SETUP` instead.

To prevent your plugin from appearing in Premiere entirely, return `PF_OutFlag_I_AM_OBSOLETE` during `GLOBAL_SETUP` when the host is Premiere:

```cpp
if (in_data->appl_id == 'PrMr') {
    out_data->out_flags = PF_OutFlag_I_AM_OBSOLETE;
    return PF_Err_NONE;
}
```

Alternatively, install the `.aex` only in the AE-specific plugins folder rather than the shared MediaCore folder.

## Plugin Registration: PiPL vs PF_REGISTER_EFFECT

After Effects still requires a PiPL resource to discover plugins. Premiere Pro no longer uses PiPL; instead it uses the `PluginDataEntryFunction` / `PF_REGISTER_EFFECT` mechanism to discover effects.

In practice, you include both:

```cpp
// Entry point for Premiere discovery
extern "C" DllExport PF_Err PluginDataEntryFunction2(
    PF_PluginDataPtr    inPtr,
    PF_PluginDataCB2    inPluginDataCallBackPtr,
    SPBasicSuite        *inSPBasicSuitePtr,
    const char          *inHostName,
    const char          *inHostVersion)
{
    PF_Err result = PF_Err_INVALID_CALLBACK;

    result = PF_REGISTER_EFFECT_EXT2(
        inPtr,
        inPluginDataCallBackPtr,
        "My Effect",            // display name
        "ADBE MyEffect",        // match name
        "My Category",          // category
        AE_RESERVED_INFO,
        "EffectMain",           // entry point function name
        "https://example.com"   // support URL
    );

    return result;
}
```

The PiPL `.r` resource continues to be needed for AE. Both can coexist in the same binary.

## GlobalSetup Timing Differences

| Behavior | After Effects | Premiere Pro |
|----------|--------------|--------------|
| When `GLOBAL_SETUP` is called | First time the effect is applied to a layer | During application startup for all plugins |
| Can show dialogs | Yes (app is fully loaded) | No (app is still loading) |
| Consequence | Lazy initialization is fine | Must be fast; no UI |

Because Premiere calls `GLOBAL_SETUP` during loading, a slow or dialog-showing GlobalSetup will block application startup for all users, even those who never use your effect.

## Features Not Available in Premiere

### AEGP Suites

AEGP suites are After Effects-specific and are **not available** in Premiere Pro. Any code that acquires an AEGP suite will fail in Premiere. This includes:

- `AEGP_SuiteHandler` and all AEGP suite families
- `AEGP_PFInterfaceSuite`
- `AEGP_StreamSuite`
- `AEGP_DynamicStreamSuite`
- `AEGP_CompSuite`
- `AEGP_LayerSuite`
- `AEGP_EffectSuite`
- All other `AEGP_*` suites

Always guard AEGP suite calls with a host check:

```cpp
if (in_data->appl_id != 'PrMr') {
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    // Safe to use AEGP suites here
    ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(...));
}
```

Attempting to acquire AEGP suites in Premiere will trigger "Not able to acquire AEFX Suite" errors at runtime.

### SmartFX (Smart Render)

Premiere **does not support SmartFX**. The `PF_Cmd_SMART_PRE_RENDER` and `PF_Cmd_SMART_RENDER` selectors are never sent by Premiere. Your plugin must implement the standard (non-smart) `PF_Cmd_RENDER` path for Premiere compatibility.

A common pattern:

```cpp
// In GlobalSetup:
if (in_data->appl_id != 'PrMr') {
    out_data->out_flags2 |= PF_OutFlag2_SUPPORTS_SMART_RENDER;
}

// In EffectMain:
case PF_Cmd_SMART_PRE_RENDER:
    // AE only -- Premiere never sends this
    err = SmartPreRender(in_data, out_data, extra);
    break;

case PF_Cmd_SMART_RENDER:
    // AE only
    err = SmartRender(in_data, out_data, extra);
    break;

case PF_Cmd_RENDER:
    // Both hosts, but Premiere uses this exclusively
    err = Render(in_data, out_data, params, output);
    break;
```

### Custom UI (Custom Comp UI)

Custom effect UIs drawn in the composition panel (`PF_Cmd_EVENT` / `PF_EventType_DRAW`, `PF_EventType_CLICK`, etc.) are not supported in Premiere. Premiere does not have an equivalent of the AE composition panel for effect interaction.

### Unsupported or Problematic Parameter Types

| Feature | Status in Premiere |
|---------|-------------------|
| `PF_Param_ARBITRARY_DATA` | Works, but with caveats |
| `PF_Param_LAYER` (extra layer inputs) | Works for transitions; limited for effects |
| Custom UI on parameters | Not supported |
| `PF_PUI_CONTROL` | Not supported |
| `PF_PUI_TOPIC` | Works |
| `PF_PUI_STD_CONTROL_ONLY` | Works |
| Parameter supervision groups | Not supported |

### Other Missing Features

- `PF_ABORT()` does not work in Premiere (never returns cancel status)
- `PF_PROGRESS()` behavior is unreliable
- `checkout_layer_pixels` (SmartRender) is unavailable
- `PF_CHECKOUT_PARAM` always returns the pre-effects layer in Premiere (no "after effects" access)
- Frame blending and motion blur from the host side work differently

## Sequence Data and PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA

Premiere handles sequence data differently from AE, and this is one of the most critical compatibility issues.

### The Problem

In After Effects, `PF_Cmd_SEQUENCE_SETUP` is called on the UI thread when the effect is first applied. The plugin allocates sequence data, and AE manages flattening/unflattening for serialization.

In Premiere, `PF_Cmd_SEQUENCE_SETUP` is **not reliably called on the render thread**. This means:

- If your plugin relies on `SEQUENCE_SETUP` to initialize render-critical state, that state may be NULL when `PF_Cmd_RENDER` is called in Premiere.
- `PF_Cmd_SEQUENCE_RESETUP` can be called on either thread and with potentially NULL `sequence_data`.

### The Solution

Set `PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA` in your `out_flags2`. This flag tells the host to use the `PF_Cmd_GET_FLATTENED_SEQUENCE_DATA` command for serialization instead of the older `SEQUENCE_FLATTEN` path.

With this flag:
- `SEQUENCE_SETUP` is only called on the UI thread.
- `SEQUENCE_RESETUP` may be called on either thread. **You must handle `in_data->sequence_data == NULL` in RESETUP.**
- `GET_FLATTENED_SEQUENCE_DATA` is called for serialization instead of `SEQUENCE_FLATTEN`.

```cpp
// In GlobalSetup - REQUIRED for Premiere compatibility with sequence data
out_data->out_flags2 |= PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA;
```

If your plugin also sets `PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING`, then `PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA` is **mandatory** when threaded rendering is enabled.

### Defensive SEQUENCE_RESETUP

```cpp
case PF_Cmd_SEQUENCE_RESETUP: {
    // sequence_data may be NULL in Premiere!
    if (in_data->sequence_data) {
        MySeqData *seqP = reinterpret_cast<MySeqData*>(
            DH(in_data->sequence_data));
        if (seqP->is_flat) {
            // Unflatten
            UnflattenSequenceData(seqP);
        }
    } else {
        // Allocate fresh sequence data
        // This happens in Premiere when RESETUP is called
        // before SETUP has run on this thread
        err = AllocateDefaultSequenceData(in_data, out_data);
    }
    break;
}
```

## The PF_OutFlag_NON_PARAM_VARY Bug

In Premiere (confirmed in v23 and v24), setting `PF_OutFlag_NON_PARAM_VARY` causes keyframed parameters on adjustment layers to always return the first keyframe's value, regardless of the current time. This affects sliders, angles, checkboxes, and 2D points.

**Workaround**: Do not set this flag when running in Premiere, since the flag has no useful effect there anyway.

```cpp
if (in_data->appl_id != 'PrMr') {
    out_data->out_flags |= PF_OutFlag_NON_PARAM_VARY;
}
```

In AE, this flag can be replaced by the `MIX_GUID` mechanism during pre-render for more fine-grained caching control.

## Premiere's Rendering Pipeline Differences

### Channel Order

After Effects uses **ARGB** channel order. Premiere uses **BGRA** (and also supports VUYA for YUV).

When your plugin runs in Premiere and receives pixel data, the channel layout depends on which pixel format Premiere selected. See the [Premiere Pixel Formats](premiere-pixel-formats.md) document for complete details.

If you do not register explicit Premiere pixel format support, Premiere will convert to ARGB 8-bit to match the default AE format. This works but adds conversion overhead.

### Negative Row Bytes

Premiere can provide pixel buffers with **negative row bytes**, meaning the first row in memory is the bottom of the image (bottom-up layout). AE always uses positive row bytes (top-down).

```cpp
// Safe row iteration for both hosts
A_long rowbytes = in_data->width * sizeof(PF_Pixel8); // approximate
char *baseP = (char *)input_worldP->data;
A_long actual_rowbytes = input_worldP->rowbytes;

for (A_long y = 0; y < input_worldP->height; y++) {
    // This works for both positive and negative rowbytes
    PF_Pixel8 *pixP = (PF_Pixel8 *)(baseP + y * actual_rowbytes);
    // process row...
}
```

### 16-Byte Buffer Alignment

Premiere pixel buffers are 16-byte aligned, with potential padding at the end of each row. Always use `rowbytes` from the world structure, never compute it from width.

### No Buffer Expansion

Premiere does not support output buffer expansion (`PF_OutFlag_I_EXPAND_BUFFER`). If your effect needs to produce output larger than the input, you must handle this differently in Premiere.

## Parameter Visibility

Hiding parameters works differently between hosts:

**In After Effects**, use AEGP suites:
```cpp
// AE: use AEGP_DynamicStreamSuite
AEGP_EffectRefH meH = NULL;
AEGP_StreamRefH streamH = NULL;
suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(my_id, effect_ref, &meH);
suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(my_id, meH, param_index, &streamH);
suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(
    streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !visible);
```

**In Premiere**, use `PF_UpdateParamUI` with `PF_PUI_INVISIBLE`:
```cpp
// Premiere: use PF_PUI_INVISIBLE flag
if (in_data->appl_id == 'PrMr') {
    params[param_index]->ui_flags |= PF_PUI_INVISIBLE;
    params[param_index]->ui_flags |= PF_PUI_DISABLED;
    suites.ParamUtilsSuite3()->PF_UpdateParamUI(
        in_data->effect_ref, param_index, params[param_index]);
}
```

## Iterate Suites

The standard AE `PF_Iterate8Suite` may not work correctly in Premiere, particularly for non-ARGB pixel formats. For 8-bit BGRA data, Premiere has its own dedicated iteration suite. For 32-bit float data, you typically need to write your own iteration function.

The `Iterate8Suite` (Suitev2, available since 2022) works for 8-bit ARGB in Premiere. For BGRA formats, use the Premiere-specific iteration suite or manual iteration with `std::for_each` / parallel algorithms.

## Media Encoder Considerations

Media Encoder (AME) can render After Effects compositions, but:

- AEGP plugins are **not loaded** by Media Encoder.
- Effect plugins (`.aex`) are loaded if they are in the shared MediaCore folder.
- Static global variables can cause issues across different Media Encoder versions. Avoid them.
- If your plugin depends on an AEGP for initialization or resources, it will not function in AME.

## Plugin Cache

Premiere caches plugin information aggressively. If your plugin changes PiPL flags, parameter definitions, or other registration data, the old cached data may persist.

**To clear the cache**: Hold Shift or Alt when opening Premiere Pro. This forces a full plugin rescan.

## Cross-Host Development Checklist

- [ ] Detect host via `in_data->appl_id` in `GLOBAL_SETUP`
- [ ] Set host-specific `out_flags` and `out_flags2`
- [ ] Implement `PF_Cmd_RENDER` (not just SmartRender)
- [ ] Guard all AEGP suite calls with host checks
- [ ] Set `PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA` if using sequence data
- [ ] Handle NULL `sequence_data` in `SEQUENCE_RESETUP`
- [ ] Do not set `PF_OutFlag_NON_PARAM_VARY` in Premiere
- [ ] Do not rely on `PF_ABORT()` in Premiere
- [ ] Handle BGRA pixel format if registering Premiere pixel format support
- [ ] Handle negative row bytes
- [ ] Do not use AEGP suites for parameter visibility in Premiere
- [ ] Test with Media Encoder in addition to Premiere
- [ ] Include both PiPL resource and `PluginDataEntryFunction` for dual-host discovery
- [ ] Clear Premiere's plugin cache (Shift+launch) after changing plugin registration
