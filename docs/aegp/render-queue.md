# Render Queue Interaction: The Queuebert Example

## Overview

The After Effects render queue is the primary mechanism for exporting compositions. The AEGP API provides three suites for programmatic render queue control:

- **`AEGP_RenderQueueSuite1`** -- global render queue operations (add comps, start/stop rendering)
- **`AEGP_RQItemSuite3/4`** -- individual render queue item management
- **`AEGP_OutputModuleSuite4`** -- output module configuration (format, path, channels, crop, audio)

The **Queuebert** SDK example (`Examples/AEGP/Queuebert/`) demonstrates nearly every render queue API call: adding compositions to the queue, iterating items, configuring output modules, setting file paths, adjusting audio/video settings, and managing output options.

---

## AEGP_RenderQueueSuite1

The top-level render queue suite (frozen since AE 5.0) has only three functions:

| Function | Purpose |
|----------|---------|
| `AEGP_AddCompToRenderQueue` | Adds a composition to the render queue with an output path |
| `AEGP_SetRenderQueueState` | Starts, pauses, or stops the render queue |
| `AEGP_GetRenderQueueState` | Queries current render queue state |

### Render Queue States

| State | Meaning |
|-------|---------|
| `AEGP_RenderQueueState_STOPPED` | Queue is idle |
| `AEGP_RenderQueueState_PAUSED` | Rendering is paused |
| `AEGP_RenderQueueState_RENDERING` | Actively rendering |

> **Warning:** Transitioning directly from `STOPPED` to `PAUSED` is illegal. You must go through `RENDERING` first.

### Adding a Comp to the Render Queue

```cpp
AEGP_CompH compH;
// ... obtain compH from a layer, project item, etc.

ERR(suites.RenderQueueSuite1()->AEGP_AddCompToRenderQueue(
    compH,
    "C:\\output\\render.mov"));  // Platform-specific path
```

This creates a new render queue item with a default output module pointing to the given path.

---

## AEGP_RQItemSuite (Versions 3 and 4)

This suite manages individual render queue items. The Queuebert example uses version 3; version 4 (frozen in AE 14.1) adds `AEGP_DeleteRQItem`.

### Key Functions

| Function | Purpose |
|----------|---------|
| `AEGP_GetNumRQItems` | Returns total number of items in the render queue |
| `AEGP_GetRQItemByIndex` | Gets an item handle by zero-based index |
| `AEGP_GetNextRQItem` | Iterates to the next item (pass `RQ_ITEM_INDEX_NONE` for first) |
| `AEGP_GetNumOutputModulesForRQItem` | Returns number of output modules on an item |
| `AEGP_GetRenderState` / `AEGP_SetRenderState` | Get/set item render status |
| `AEGP_GetStartedTime` | When rendering started (returns `{0,1}` if not started) |
| `AEGP_GetElapsedTime` | Elapsed render time (returns `{0,1}` if not rendered) |
| `AEGP_GetLogType` / `AEGP_SetLogType` | Control render logging verbosity |
| `AEGP_GetComment` / `AEGP_SetComment` | Get/set the item's comment string |
| `AEGP_GetCompFromRQItem` | Retrieve the composition associated with a queue item |
| `AEGP_RemoveOutputModule` | Remove an output module from an item |
| `AEGP_DeleteRQItem` | (Suite 4) Delete an entire render queue item (undoable) |

### Render Item Status

| Status | Value | Meaning |
|--------|-------|---------|
| `AEGP_RenderItemStatus_NONE` | -2 | Invalid/uninitialized |
| `AEGP_RenderItemStatus_WILL_CONTINUE` | -1 | Will continue rendering |
| `AEGP_RenderItemStatus_NEEDS_OUTPUT` | 0 | Missing output path |
| `AEGP_RenderItemStatus_UNQUEUED` | 1 | Ready but not queued |
| `AEGP_RenderItemStatus_QUEUED` | 2 | Ready and queued for render |
| `AEGP_RenderItemStatus_RENDERING` | 3 | Currently rendering |
| `AEGP_RenderItemStatus_USER_STOPPED` | 4 | User cancelled render |
| `AEGP_RenderItemStatus_ERR_STOPPED` | 5 | Error during render |
| `AEGP_RenderItemStatus_DONE` | 6 | Render completed |

> **Warning:** `AEGP_SetRenderState` returns `Err_PARAMETER` if called while the render queue is not in `STOPPED` state. It returns `Err_RANGE` for invalid status values, and `Err_PARAMETER` for status transitions that do not make sense (e.g., queuing an item with no output path set).

### Log Types

| Type | Meaning |
|------|---------|
| `AEGP_LogType_ERRORS_ONLY` | Log only errors |
| `AEGP_LogType_PLUS_SETTINGS` | Log errors and render settings |
| `AEGP_LogType_PER_FRAME_INFO` | Log every frame (verbose) |

---

## AEGP_OutputModuleSuite4

This suite controls everything about how an output module renders: format, channels, file path, crop, stretch, audio settings, and post-render actions.

### Output Module Functions

| Function | Purpose |
|----------|---------|
| `AEGP_GetOutputModuleByIndex` | Get output module handle by index |
| `AEGP_GetEnabledOutputs` / `AEGP_SetEnabledOutputs` | Control video/audio output flags |
| `AEGP_GetOutputChannels` / `AEGP_SetOutputChannels` | RGB, RGBA, or Alpha-only |
| `AEGP_GetStretchInfo` / `AEGP_SetStretchInfo` | Output stretch/resize settings |
| `AEGP_GetCropInfo` / `AEGP_SetCropInfo` | Output cropping rectangle |
| `AEGP_GetSoundFormatInfo` / `AEGP_SetSoundFormatInfo` | Audio format (sample rate, channels, encoding) |
| `AEGP_GetEmbedOptions` / `AEGP_SetEmbedOptions` | Project link embedding |
| `AEGP_GetPostRenderAction` / `AEGP_SetPostRenderAction` | What happens after rendering |
| `AEGP_GetOutputFilePath` / `AEGP_SetOutputFilePath` | Get/set the output file path |
| `AEGP_AddDefaultOutputModule` | Add a new output module with default settings |
| `AEGP_GetExtraOutputModuleInfo` | Query format name, info string, sequence/multi-frame status |

### Output Type Flags

```cpp
AEGP_OutputType_VIDEO  = (1 << 0)   // Include video
AEGP_OutputType_AUDIO  = (1 << 1)   // Include audio
```

These are bitmask flags that can be combined:

```cpp
output = AEGP_OutputType_AUDIO | AEGP_OutputType_VIDEO;
ERR(suites.OutputModuleSuite4()->AEGP_SetEnabledOutputs(rqItemH, omH, output));
```

### Video Channel Options

| Constant | Output |
|----------|--------|
| `AEGP_VideoChannels_RGB` | RGB only (no alpha) |
| `AEGP_VideoChannels_RGBA` | RGB + Alpha |
| `AEGP_VideoChannels_ALPHA` | Alpha channel only |

### Embedding Options

| Constant | Behavior |
|----------|----------|
| `AEGP_Embedding_NOTHING` | No project link embedded |
| `AEGP_Embedding_LINK` | Embed a link to the project |
| `AEGP_Embedding_LINK_AND_COPY` | Embed link and copy of project |

### Post-Render Actions

| Constant | Behavior |
|----------|----------|
| `AEGP_PostRenderOptions_IMPORT` | Import rendered file back into project |
| `AEGP_PostRenderOptions_IMPORT_AND_REPLACE_USAGE` | Import and replace the comp's usage |
| `AEGP_PostRenderOptions_SET_PROXY` | Set rendered file as proxy |

---

## How the Queuebert Example Works

Queuebert registers a menu command under the File menu. It enables itself only when the render queue has at least one item. When invoked, it performs a comprehensive tour of the render queue API.

### Entry Point

```cpp
A_Err EntryPointFunc(struct SPBasicSuite *pica_basicP, ...)
{
    // Standard AEGP registration
    ERR(suites.CommandSuite1()->AEGP_GetUniqueCommand(&S_queuebert_cmd));
    ERR(suites.CommandSuite1()->AEGP_InsertMenuCommand(
        S_queuebert_cmd, "QueueBert",
        AEGP_Menu_FILE, AEGP_MENU_INSERT_SORTED));
    ERR(suites.RegisterSuite5()->AEGP_RegisterCommandHook(
        S_my_id, AEGP_HP_BeforeAE, AEGP_Command_ALL, CommandHook, NULL));
    ERR(suites.RegisterSuite5()->AEGP_RegisterUpdateMenuHook(
        S_my_id, UpdateMenuHook, NULL));
}
```

### Menu Enabling Logic

Queuebert only enables when the render queue has items:

```cpp
static A_Err UpdateMenuHook(...)
{
    A_long num_rq_itemsL = 0;
    ERR(suites.RQItemSuite3()->AEGP_GetNumRQItems(&num_rq_itemsL));

    if (num_rq_itemsL) {
        ERR(suites.CommandSuite1()->AEGP_EnableCommand(S_queuebert_cmd));
    } else {
        ERR(suites.CommandSuite1()->AEGP_DisableCommand(S_queuebert_cmd));
    }
    return err;
}
```

### Command Handler: The Full Render Queue Tour

When invoked, the Queuebert CommandHook performs these operations in sequence:

#### 1. Add Compositions to the Render Queue

```cpp
ERR(suites.RQItemSuite3()->AEGP_GetNumRQItems(&rq_item_countL));
if (rq_item_countL) {
    ERR(suites.RQItemSuite3()->AEGP_GetCompFromRQItem(0, &compH));
    for (A_long iL = 0; !err && iL < 6; ++iL) {
        ERR(suites.RenderQueueSuite1()->AEGP_AddCompToRenderQueue(
            compH, "C:\\output.mov"));
        ERR(suites.RQItemSuite3()->AEGP_GetNextRQItem(
            (AEGP_RQItemRefH)iL, &rq_itemH));
        ERR(suites.RQItemSuite3()->AEGP_SetRenderState(
            rq_itemH, AEGP_RenderItemStatus_UNQUEUED));
    }
}
```

This takes the comp from the first render queue item and adds it six more times, setting each to `UNQUEUED` status.

#### 2. Query and Modify Render Queue Item Settings

```cpp
ERR(suites.RQItemSuite3()->AEGP_GetLogType(0, &log_type));
ERR(suites.RQItemSuite3()->AEGP_SetLogType(0, AEGP_LogType_PLUS_SETTINGS));
ERR(suites.RQItemSuite3()->AEGP_GetRenderState(0, &status));
ERR(suites.RQItemSuite3()->AEGP_SetRenderState(0, TRUE));
ERR(suites.RQItemSuite3()->AEGP_GetStartedTime(0, &startT));
ERR(suites.RQItemSuite3()->AEGP_GetElapsedTime(0, &elapsedT));
```

#### 3. Configure Output Module Video Settings

```cpp
// Enable both video and audio output
output = AEGP_OutputType_AUDIO | AEGP_OutputType_VIDEO;
ERR(suites.OutputModuleSuite4()->AEGP_SetEnabledOutputs(0, 0, output));

// Set RGBA output channels
ERR(suites.OutputModuleSuite4()->AEGP_SetOutputChannels(
    0, 0, AEGP_VideoChannels_RGBA));
```

#### 4. Configure Stretch and Crop

```cpp
// Enable high-quality stretching
ERR(suites.OutputModuleSuite4()->AEGP_SetStretchInfo(
    0, 0, TRUE, AEGP_StretchQual_HIGH));

// Set crop to 200x100 pixels
cropR.right  = 200;
cropR.bottom = 100;
ERR(suites.OutputModuleSuite4()->AEGP_SetCropInfo(0, 0, TRUE, cropR));
```

#### 5. Configure Audio Settings

```cpp
if (!audio_enabledB || !audio_settings.sample_rateF) {
    audio_settings.num_channelsL    = 1;        // Mono
    audio_settings.sample_rateF     = 44100;    // 44.1 kHz
    audio_settings.bytes_per_sampleL = 4;       // 32-bit
    audio_settings.encoding         = AEGP_SoundEncoding_FLOAT;

    ERR(suites.OutputModuleSuite4()->AEGP_SetSoundFormatInfo(
        0, 0, audio_settings, audio_enabledB));
}
```

#### 6. Configure Embedding and Post-Render Actions

```cpp
ERR(suites.OutputModuleSuite4()->AEGP_SetEmbedOptions(
    0, 0, AEGP_Embedding_LINK_AND_COPY));

ERR(suites.OutputModuleSuite4()->AEGP_SetPostRenderAction(
    0, 0, AEGP_PostRenderOptions_IMPORT_AND_REPLACE_USAGE));
```

#### 7. Manage Output Modules

```cpp
// Add a new default output module
ERR(suites.OutputModuleSuite4()->AEGP_AddDefaultOutputModule(0, &omrefH));

// Remove the first output module
ERR(suites.RQItemSuite3()->AEGP_RemoveOutputModule(0, 0));

// Set the output file path on the remaining module
ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(
    0,
    (AEGP_OutputModuleRefH)(outmod_countL - 1),
    "C:\\whee.mov"));
```

#### 8. Set Item Comment

```cpp
ERR(suites.RQItemSuite3()->AEGP_GetComment(0, commentZ));
ERR(suites.RQItemSuite3()->AEGP_SetComment(0, "That's pronounced cue-BARE!"));
```

---

## AEGP_SoundDataFormat Structure

The audio settings structure used with `Get/SetSoundFormatInfo`:

| Field | Type | Description |
|-------|------|-------------|
| `num_channelsL` | `A_long` | Number of audio channels (1=mono, 2=stereo) |
| `sample_rateF` | `A_FpLong` | Sample rate in Hz (e.g., 44100, 48000) |
| `bytes_per_sampleL` | `A_long` | Bytes per sample (1, 2, or 4) |
| `encoding` | `AEGP_SoundEncoding` | `AEGP_SoundEncoding_UNSIGNED_PCM`, `AEGP_SoundEncoding_SIGNED_PCM`, or `AEGP_SoundEncoding_FLOAT` |

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Caching `AEGP_RQItemRefH` handles | **Never cache these.** They are invalidated by any reordering, addition, or removal of render queue items. Always re-query. |
| Caching `AEGP_OutputModuleRefH` handles | Same rule -- invalidated by any output module change. Always re-query. |
| Calling `AEGP_SetRenderState` while rendering | Returns `Err_PARAMETER`. The render queue must be `STOPPED` first. |
| Setting status to `QUEUED` without an output path | Returns `Err_PARAMETER`. Set the output path first. |
| Assuming output module indices are stable | After `AEGP_RemoveOutputModule` or `AEGP_AddDefaultOutputModule`, all indices shift. Re-query `AEGP_GetNumOutputModulesForRQItem`. |
| Using `AEGP_SetOutputFilePath` with wrong separators | Use platform-native separators: backslashes on Windows, colons (classic Mac) or forward slashes on macOS |
| Not checking `rq_item_countL` before accessing items | Accessing index 0 when the queue is empty will cause errors |

---

## Iterating the Render Queue from a Command Hook (Gotchas)

When a render-queue command hook (`AEGP_HP_BeforeAE`, `AEGP_Command_ALL`) needs to find *which* comps are about to render -- e.g. to pre-process them before AE dispatches frames -- two non-obvious traps cost real debugging time:

### `AEGP_GetRQItemByIndex` is effectively 1-indexed

Despite the header describing a "zero-based index," index `0` returns a **null** `AEGP_RQItemRefH` in practice; the `N` real items reported by `AEGP_GetNumRQItems` live at indices **`1..N`**. A naive `for (i = 0; i < n; ++i)` loop therefore skips index 0 (null) *and* misses index `N` (the last item) -- so a queue with a **single** item finds **nothing**. Iterate inclusively over `1..N`:

```cpp
A_long n = 0;
suites.RQItemSuite4()->AEGP_GetNumRQItems(&n);
for (A_long i = 1; i <= n; ++i) {            // 1..N, not 0..N-1
    AEGP_RQItemRefH rq = nullptr;
    if (suites.RQItemSuite4()->AEGP_GetRQItemByIndex(i, &rq) != A_Err_NONE || !rq)
        continue;                            // tolerate a stray null
    AEGP_RenderItemStatusType st = AEGP_RenderItemStatus_NONE;
    suites.RQItemSuite4()->AEGP_GetRenderState(rq, &st);
    // QUEUED == 2; see the status table above
}
```

### `AEGP_GetNextRQItem` needs the right "first" sentinel

`AEGP_GetNextRQItem(current, &next)` returns the first item when `current` is the *ref* sentinel -- **not** `RQ_ITEM_INDEX_NONE` (an *index* sentinel). Casting `(AEGP_RQItemRefH)RQ_ITEM_INDEX_NONE` yields a bogus handle and the chain never starts (zero items iterated). Prefer the count + index loop above, which is unambiguous.

### Match the status enum, and accept the active set

In a `BeforeAE` render hook the item you care about may be `QUEUED` (2), `RENDERING` (3), or `WILL_CONTINUE` (-1) depending on exactly when the hook fires relative to AE's `STOPPED -> RENDERING` transition. Gate on the whole active set, not `QUEUED` alone, or you will intermittently miss your own render. (The 2302/2303 render/stop command pair acts as one toggle that the render-queue *state* disambiguates: it is "Render" when `STOPPED`, "Stop" when `RENDERING`.)

---

## Stateful (Simulation) Effects and Multi-Frame Rendering

An effect whose frame *N* depends on the result of frame *N-1* (a fluid/particle sim, an echo/feedback, paint accumulation) **cannot** rely on receiving frames in order. With Multi-Frame Rendering (on by default since AE 2021), AE dispatches preview *and* render frames out of order across threads, and **there is no SDK flag to force sequential order** -- not even by withholding the MFR flag (AE 2021+ still reorders). This is by design: MFR cannot support stateful computation.

The established solution -- used by AE's own **Camera Tracker** and confirmed by multiple shipping third-party plugins -- is to **compute the order-dependent range in one pass, yourself, off AE's request path, then serve the cached result**:

1. Hook the Render-Queue "Render" command (`AEGP_HP_BeforeAE`). When the queue moves toward `RENDERING` while still readable, scan the active RQ items (see gotchas above) for comps containing *your* effect (match by your effect's unique ID; with multiple SKUs installed, each hook keeps only its own so the absent SKU bails).
2. For each such comp, drive your own **in-order** computation synchronously (e.g. via `AEGP_EffectCallGeneric` with `PF_Cmd_COMPLETELY_GENERAL`) to bake the frame range to a disk/RAM cache, keyed by a validity hash of the sim-affecting params.
3. Let AE render normally; your `SmartRender` reads the pre-baked frames (a resolution-independent serve via box-downscale lets one full-res bake cover reduced-res previews). Out-of-order requests now hit faithful pre-computed data instead of diverging re-sims.

In GUI/interactive mode the same pre-compute can run on an idle hook with a progress UI; in headless export it simply runs before dispatch. Keep the **sim** params and the **look/render** params in *separate* validity hashes, so a look-only edit (palette, exposure) reuses the sim cache and only re-renders -- otherwise every cosmetic tweak forces a full (and, under MFR, divergent) re-sim.

---

## Practical Use Cases

### Batch Rendering Automation
Add multiple compositions to the render queue with pre-configured output settings, then start rendering -- all from a single menu command.

### Render Farm Integration
Query render queue items, extract output paths and settings, and dispatch them to a render farm manager.

### Output Template Application
Programmatically apply standardized output settings (codec, channels, audio format) to all items in the queue.

### Render Monitoring
Poll `AEGP_GetRenderState`, `AEGP_GetStartedTime`, and `AEGP_GetElapsedTime` to build custom render progress dashboards.

### Post-Render Workflows
Use `AEGP_SetPostRenderAction` with `IMPORT_AND_REPLACE_USAGE` to automatically replace proxy footage with final renders.

---

## Starting a Render Programmatically

To actually kick off rendering after populating the queue:

```cpp
// Ensure items are queued
ERR(suites.RQItemSuite3()->AEGP_SetRenderState(
    rq_itemH, AEGP_RenderItemStatus_QUEUED));

// Start the render queue
ERR(suites.RenderQueueSuite1()->AEGP_SetRenderQueueState(
    AEGP_RenderQueueState_RENDERING));
```

> **Note:** Once `AEGP_RenderQueueState_RENDERING` is set, the UI will show the render progress dialog and your AEGP will not regain control until idle hooks fire. Plan accordingly.

---

## Related Suites

| Suite | Relationship |
|-------|-------------|
| `AEGP_CompSuite` | Obtain `AEGP_CompH` handles to add to the render queue |
| `AEGP_ProjSuite` | Iterate project items to find compositions |
| `AEGP_CommandSuite1` | Register menu commands (as Queuebert does) |
| `AEGP_RegisterSuite5` | Register hook callbacks |
| `AEGP_MemorySuite1` | Dispose memory handles returned by `AEGP_GetComment` and `AEGP_GetOutputFilePath` |
