# Advanced Effect Suites, Hash Suite, and Cache-On-Load

## Overview

This document covers three specialized headers that provide functionality beyond standard effect rendering:

- **AE_AdvEffectSuites.h** -- Application-level utilities (project management, info panel drawing, time formatting, item control) available to effect plugins
- **AE_HashSuite.h** -- Hashing utilities for cache key generation and render identification
- **AE_CacheOnLoadSuite.h** -- Control over whether AE caches plugin information at startup

These suites are less commonly used than the core rendering suites, but they solve specific problems that arise in production plugin development.

---

## AE_AdvEffectSuites.h: Advanced Application Suites

This header defines three distinct suites, each providing effect plugins with access to application-level functionality that goes beyond pixel processing.

### PF_AdvAppSuite -- Application Control

```c
#define kPFAdvAppSuite           "PF AE Adv App Suite"
#define kPFAdvAppSuiteVersion1   1   // Frozen in AE 5.0
#define kPFAdvAppSuiteVersion2   2   // Frozen in AE 6.0
```

This suite gives effect plugins the ability to interact with the AE application itself -- marking the project dirty, saving, managing foreground/background state, and drawing to the info panel.

#### Version 1 Functions (Frozen AE 5.0)

| Function | Signature | Purpose |
|----------|-----------|---------|
| `PF_SetProjectDirty` | `(void)` | Marks the project as modified (unsaved changes) |
| `PF_SaveProject` | `(void)` | Triggers a project save |
| `PF_SaveBackgroundState` | `(void)` | Saves the current foreground/background state |
| `PF_ForceForeground` | `(void)` | Forces AE to the foreground |
| `PF_RestoreBackgroundState` | `(void)` | Restores previously saved background state |
| `PF_RefreshAllWindows` | `(void)` | Redraws all AE panels and windows |
| `PF_InfoDrawText` | `(line1, line2)` | Draws 2 lines of text in the info panel |
| `PF_InfoDrawColor` | `(PF_Pixel color)` | Displays a color swatch in the info panel |
| `PF_InfoDrawText3` | `(line1, line2, line3)` | Draws 3 lines in the info panel |
| `PF_InfoDrawText3Plus` | `(line1, line2_jr, line2_jl, line3_jr, line3_jl)` | 3 lines with right/left justification |

#### Version 2 Additions (Frozen AE 6.0)

Version 2 adds one function to the version 1 set:

| Function | Signature | Purpose |
|----------|-----------|---------|
| `PF_AppendInfoText` | `(appendZ0)` | Appends text to the top info line for a number of ticks |

#### Usage: Info Panel Drawing

The info panel functions are useful during `PF_Cmd_DO_DIALOG` (custom UI interaction) or `PF_Cmd_EVENT` (parameter UI events) to provide feedback to the user:

```c
PF_Err HandleMouseMove(PF_InData *in_data, PF_OutData *out_data, /* ... */) {
    PF_AdvAppSuite2 *appSuite = NULL;
    in_data->pica_basicP->AcquireSuite(
        kPFAdvAppSuite,
        kPFAdvAppSuiteVersion2,
        (const void **)&appSuite);

    if (appSuite) {
        // Show pixel value under cursor
        char line1[256], line2[256];
        snprintf(line1, sizeof(line1), "X: %d  Y: %d", mouseX, mouseY);
        snprintf(line2, sizeof(line2), "R: %d  G: %d  B: %d", r, g, b);
        appSuite->PF_InfoDrawText(line1, line2);

        // Also show the color swatch
        PF_Pixel color = {255, (A_u_char)r, (A_u_char)g, (A_u_char)b};
        appSuite->PF_InfoDrawColor(color);

        in_data->pica_basicP->ReleaseSuite(kPFAdvAppSuite, kPFAdvAppSuiteVersion2);
    }
    return PF_Err_NONE;
}
```

#### Usage: Marking Project Dirty

When your effect's custom UI modifies persistent data (sequence data, arbitrary data), call `PF_SetProjectDirty` so AE knows the project has unsaved changes:

```c
PF_Err HandleParamChange(PF_InData *in_data, /* ... */) {
    // Modify sequence data
    MySeqData *seq = (MySeqData *)PF_LOCK_HANDLE(in_data->sequence_data);
    seq->custom_value = new_value;
    PF_UNLOCK_HANDLE(in_data->sequence_data);

    // Tell AE the project changed
    PF_AdvAppSuite1 *appSuite = NULL;
    in_data->pica_basicP->AcquireSuite(
        kPFAdvAppSuite, kPFAdvAppSuiteVersion1, (const void **)&appSuite);
    if (appSuite) {
        appSuite->PF_SetProjectDirty();
        in_data->pica_basicP->ReleaseSuite(kPFAdvAppSuite, kPFAdvAppSuiteVersion1);
    }

    return PF_Err_NONE;
}
```

> **Pitfall:** Do not call `PF_SaveProject` during rendering. It should only be called from UI-thread contexts (event handlers, dialog callbacks). Calling it during render can cause deadlocks or data corruption.

#### Usage: Foreground/Background Management

The foreground/background functions are used when your plugin needs to perform UI operations that require AE to be the active application (e.g., opening system dialogs):

```c
PF_Err MyDialog(PF_InData *in_data, /* ... */) {
    PF_AdvAppSuite1 *appSuite = NULL;
    in_data->pica_basicP->AcquireSuite(
        kPFAdvAppSuite, kPFAdvAppSuiteVersion1, (const void **)&appSuite);

    if (appSuite) {
        appSuite->PF_SaveBackgroundState();
        appSuite->PF_ForceForeground();

        // Show a system dialog, open a file picker, etc.
        ShowMyCustomDialog();

        appSuite->PF_RestoreBackgroundState();
        in_data->pica_basicP->ReleaseSuite(kPFAdvAppSuite, kPFAdvAppSuiteVersion1);
    }
    return PF_Err_NONE;
}
```

---

### PF_AdvTimeSuite -- Time Formatting

```c
#define kPFAdvTimeSuite            "PF AE Adv Time Suite"
#define kPFAdvTimeSuiteVersion1    1   // Frozen in AE 5.0
#define kPFAdvTimeSuiteVersion2    2
#define kPFAdvTimeSuiteVersion3    3   // Frozen in AE 14.2
#define kPFAdvTimeSuiteVersion4    4   // Frozen in AE 15.0
```

This suite formats time values according to the user's display preferences (timecode, frames, feet+frames). It is essential for any plugin that displays time values in its UI.

#### Time Display Buffer

```c
#define PF_MAX_TIME_LEN  31
// Allocate buffers as PF_MAX_TIME_LEN + 1
```

#### Time Display Preferences

The suite provides access to the user's time display settings through versioned preference structures:

**Version 3 (Current):**
```c
typedef struct {
    A_char      display_mode;           // Display format
    A_long      framemax;               // Maximum frame count
    A_long      frames_per_foot;        // Frames per foot (film)
    A_char      frames_start;           // Starting frame number
    A_Boolean   nondrop30B;             // Non-drop frame 30fps
    A_Boolean   honor_source_timecodeB; // Use source timecode
    A_Boolean   use_feet_framesB;       // Feet+frames mode
} PF_TimeDisplayPrefVersion3;
```

**Display format constants:**
```c
enum {
    PF_TimeDisplayFormatTimecode,       // HH:MM:SS:FF
    PF_TimeDisplayFormatFrames,         // Frame number
    PF_TimeDisplayFormatFeetFrames      // OBSOLETE (v1 only)
};
```

#### Version 4 Functions

| Function | Purpose |
|----------|---------|
| `PF_FormatTimeActiveItem` | Format time for the active composition |
| `PF_FormatTime` | Format time for a specific effect world |
| `PF_FormatTimePlus` | Format with additional comp-time flag |
| `PF_GetTimeDisplayPref` | Retrieve current time display preferences |
| `PF_TimeCountFrames` | Count frames between two time points |

#### Usage Example

```c
PF_Err DisplayCurrentTime(PF_InData *in_data) {
    PF_AdvTimeSuite4 *timeSuite = NULL;
    in_data->pica_basicP->AcquireSuite(
        kPFAdvTimeSuite,
        kPFAdvTimeSuiteVersion4,
        (const void **)&timeSuite);

    if (timeSuite) {
        A_char time_buf[PF_MAX_TIME_LEN + 1] = {0};

        // Format the current time of the active item
        timeSuite->PF_FormatTimeActiveItem(
            in_data->current_time,   // time value
            in_data->time_scale,     // time scale
            FALSE,                   // not a duration
            time_buf);

        // time_buf now contains something like "00:00:05:12" or "162"
        // depending on user preferences

        in_data->pica_basicP->ReleaseSuite(kPFAdvTimeSuite, kPFAdvTimeSuiteVersion4);
    }
    return PF_Err_NONE;
}
```

#### Counting Frames

Version 4 added `PF_TimeCountFrames`, useful for calculating frame ranges:

```c
A_Time start = {0, 24};       // 0 seconds at 24fps
A_Time step  = {1, 24};       // 1/24 second step
A_long frame_count = 0;

timeSuite->PF_TimeCountFrames(&start, &step, TRUE, &frame_count);
```

> **Pitfall:** When formatting times, always allocate at least `PF_MAX_TIME_LEN + 1` bytes for the output buffer. The maximum formatted time string can be 31 characters.

---

### PF_AdvItemSuite -- Item Control

```c
#define kPFAdvItemSuite          "PF AE Adv Item Suite"
#define kPFAdvItemSuiteVersion1  1   // Frozen in AE 5.0
```

This suite provides control over the active item in the timeline -- stepping through time, forcing rerenders, and querying effect state.

#### Functions

| Function | Signature | Purpose |
|----------|-----------|---------|
| `PF_MoveTimeStep` | `(in_data, world, time_dir, num_steps)` | Step time forward or backward for a specific item |
| `PF_MoveTimeStepActiveItem` | `(time_dir, num_steps)` | Step time for the active item |
| `PF_TouchActiveItem` | `(void)` | Mark the active item as needing re-render |
| `PF_ForceRerender` | `(in_data, world)` | Force a specific item to re-render |
| `PF_EffectIsActiveOrEnabled` | `(contextH, enabledPB)` | Check if an effect is active/enabled |

#### Time Step Direction

```c
enum {
    PF_Step_FORWARD,
    PF_Step_BACKWARD
};
typedef A_LegacyEnumType PF_Step;
```

#### Usage: Forcing Re-render

When your effect's external state changes (e.g., a file it reads is modified, a network resource updates), you can force AE to re-render:

```c
PF_Err OnExternalStateChange(PF_InData *in_data, PF_EffectWorld *world) {
    PF_AdvItemSuite1 *itemSuite = NULL;
    in_data->pica_basicP->AcquireSuite(
        kPFAdvItemSuite,
        kPFAdvItemSuiteVersion1,
        (const void **)&itemSuite);

    if (itemSuite) {
        itemSuite->PF_ForceRerender(in_data, world);
        in_data->pica_basicP->ReleaseSuite(kPFAdvItemSuite, kPFAdvItemSuiteVersion1);
    }
    return PF_Err_NONE;
}
```

#### Usage: Checking Effect State

```c
PF_Err CheckIfEnabled(PF_InData *in_data, PF_ContextH contextH) {
    PF_AdvItemSuite1 *itemSuite = NULL;
    in_data->pica_basicP->AcquireSuite(
        kPFAdvItemSuite, kPFAdvItemSuiteVersion1, (const void **)&itemSuite);

    if (itemSuite) {
        PF_Boolean is_enabled = FALSE;
        itemSuite->PF_EffectIsActiveOrEnabled(contextH, &is_enabled);

        if (!is_enabled) {
            // Effect is disabled -- skip expensive computation
        }

        in_data->pica_basicP->ReleaseSuite(kPFAdvItemSuite, kPFAdvItemSuiteVersion1);
    }
    return PF_Err_NONE;
}
```

> **Pitfall:** `PF_TouchActiveItem` and `PF_MoveTimeStepActiveItem` operate on whatever item is currently active in the AE UI. They should only be called from UI-thread contexts, not during rendering. The "active item" can change at any time due to user interaction.

---

## AE_HashSuite.h: Hashing for Cache Management

```c
#define kAEGPHashSuite           "AEGP Hash Suite"
#define kAEGPHashSuiteVersion1   1   // Frozen in AE 17.5.1
```

The Hash Suite provides utilities for generating deterministic hash values from arbitrary data. These hashes are used primarily as cache keys in conjunction with the Compute Cache Suite (`AE_ComputeCacheSuite.h`).

### The AEGP_GUID Type

```c
// Defined in AE_ComputeCacheSuite.h
typedef struct AEGP_GUID {
    A_long bytes[4];    // 128-bit GUID (4 x 32-bit integers)
} AEGP_GUID;
```

The hash output is a 128-bit GUID stored as four 32-bit integers. This provides sufficient collision resistance for cache key purposes.

### Functions

#### AEGP_CreateHashFromPtr

```c
SPAPI A_Err (*AEGP_CreateHashFromPtr)(
    const A_u_longlong  buf_sizeLu,   // >> Size of the buffer in bytes
    const void          *bufPV,       // >> Pointer to data to hash
    AEGP_GUID           *hashP);      // << Resulting hash
```

Creates a new hash from a raw memory buffer. This is the starting point for building a cache key.

#### AEGP_HashMixInPtr

```c
SPAPI A_Err (*AEGP_HashMixInPtr)(
    const A_u_longlong  buf_sizeLu,   // >> Size of the buffer in bytes
    const void          *bufPV,       // >> Pointer to data to mix in
    AEGP_GUID           *hashP);      // <> Existing hash, modified in place
```

Mixes additional data into an existing hash. Use this to combine multiple pieces of data into a single cache key. The hash is modified in place.

### Usage Pattern: Building a Cache Key

The typical pattern is to create a hash from the first piece of identifying data, then mix in additional pieces:

```c
A_Err BuildCacheKey(
    PF_InData       *in_data,
    A_long          param1_value,
    A_long          param2_value,
    A_FpLong        param3_value,
    PF_State        *layer_state,
    AEGP_GUID       *out_keyP)
{
    A_Err err = A_Err_NONE;

    AEGP_HashSuite1 *hashSuite = NULL;
    in_data->pica_basicP->AcquireSuite(
        kAEGPHashSuite,
        kAEGPHashSuiteVersion1,
        (const void **)&hashSuite);

    if (!hashSuite) return A_Err_MISSING_SUITE;

    // Start with the first parameter
    err = hashSuite->AEGP_CreateHashFromPtr(
        sizeof(param1_value),
        &param1_value,
        out_keyP);

    if (!err) {
        // Mix in additional parameters
        err = hashSuite->AEGP_HashMixInPtr(
            sizeof(param2_value),
            &param2_value,
            out_keyP);
    }

    if (!err) {
        err = hashSuite->AEGP_HashMixInPtr(
            sizeof(param3_value),
            &param3_value,
            out_keyP);
    }

    if (!err) {
        // Mix in the input layer state (identifies the specific frame content)
        err = hashSuite->AEGP_HashMixInPtr(
            sizeof(PF_State),
            layer_state,
            out_keyP);
    }

    in_data->pica_basicP->ReleaseSuite(kAEGPHashSuite, kAEGPHashSuiteVersion1);
    return err;
}
```

### Integration with Compute Cache Suite

The Hash Suite is designed to work with the Compute Cache Suite (`AE_ComputeCacheSuite.h`). The typical workflow is:

1. **Register** a compute class with callbacks (key generation, computation, sizing, deletion)
2. **Generate a key** using the Hash Suite in your `generate_key` callback
3. **Checkout** cached values using the Compute Cache Suite
4. **Check in** receipts when done

```c
// The generate_key callback uses HashSuite to build the cache key
A_Err MyGenerateKey(
    AEGP_CCComputeOptionsRefconP optionsP,
    AEGP_CCComputeKeyP           out_keyP)
{
    MyComputeOptions *opts = (MyComputeOptions *)optionsP;

    AEGP_HashSuite1 *hashSuite = NULL;
    opts->pica_basicP->AcquireSuite(
        kAEGPHashSuite, kAEGPHashSuiteVersion1, (const void **)&hashSuite);

    // Hash all inputs that affect the computation
    hashSuite->AEGP_CreateHashFromPtr(
        sizeof(opts->effect_version), &opts->effect_version, out_keyP);
    hashSuite->AEGP_HashMixInPtr(
        sizeof(opts->threshold), &opts->threshold, out_keyP);
    hashSuite->AEGP_HashMixInPtr(
        sizeof(opts->input_state), &opts->input_state, out_keyP);

    opts->pica_basicP->ReleaseSuite(kAEGPHashSuite, kAEGPHashSuiteVersion1);
    return A_Err_NONE;
}
```

### What to Hash

When building cache keys, include everything that affects the computed result:

| Include | Why |
|---------|-----|
| Effect parameter values | Different parameter values produce different results |
| Input layer state (via `PF_GetCurrentState`) | Different input pixels produce different results |
| Effect version/revision | Algorithm changes invalidate old cache entries |
| Bit depth | 8/16/32-bit may produce different results |
| Downsample factor | Reduced resolution renders differ from full |

| Do NOT Include | Why |
|----------------|-----|
| Absolute time (unless time-dependent) | Same inputs at different times should hit the same cache |
| Render thread ID | Cache is shared across threads |
| Memory addresses | These change between sessions |

> **Pitfall:** Forgetting to include a parameter in the hash key leads to stale cache hits -- the cache returns old data computed with different parameters. This manifests as the effect appearing to "not respond" to parameter changes.

> **Pitfall:** Including too much in the key (e.g., changing data that does not affect the result) leads to unnecessary cache misses, defeating the purpose of caching.

---

## AE_CacheOnLoadSuite.h: Controlling Startup Caching

```c
#define kPFCacheOnLoadSuite          "PF Cache On Load Suite"
#define kPFCacheOnLoadSuiteVersion1  1
```

This is the simplest suite in this document. It has a single function that controls whether AE caches your plugin's information at startup.

### The Problem It Solves

By default, AE reads and caches information about each plugin during startup (scanning PiPL resources, building menu entries, etc.). Once cached, AE does not need to load the plugin's binary on subsequent launches -- it uses the cached data instead.

However, some plugins need to be loaded fresh every time. For example:
- Plugins that dynamically generate their effect name or parameter list based on external configuration files
- Plugins that perform license validation at startup
- Plugins whose PiPL properties change between launches

### The Function

```c
typedef struct PF_CacheOnLoadSuite1 {
    SPAPI PF_Err (*PF_SetNoCacheOnLoad)(
        PF_ProgPtr  effect_ref,        // >> Your effect reference
        long        effectAvailable);   // >> Whether the effect should be available
} PF_CacheOnLoadSuite1;
```

#### Parameters

- `effect_ref` -- Your plugin's `PF_ProgPtr` (obtained from `in_data->effect_ref`)
- `effectAvailable` -- If non-zero, the effect is available but will not be cached; if zero, the effect is marked as unavailable

### Usage

Call this during `PF_Cmd_GLOBAL_SETUP`:

```c
PF_Err GlobalSetup(PF_InData *in_data, PF_OutData *out_data) {
    PF_CacheOnLoadSuite1 *cacheSuite = NULL;
    in_data->pica_basicP->AcquireSuite(
        kPFCacheOnLoadSuite,
        kPFCacheOnLoadSuiteVersion1,
        (const void **)&cacheSuite);

    if (cacheSuite) {
        // Tell AE to reload this plugin every startup
        cacheSuite->PF_SetNoCacheOnLoad(in_data->effect_ref, TRUE);
        in_data->pica_basicP->ReleaseSuite(kPFCacheOnLoadSuite, kPFCacheOnLoadSuiteVersion1);
    }

    // ... rest of global setup
    return PF_Err_NONE;
}
```

### When to Use

| Scenario | Use CacheOnLoad? |
|----------|-----------------|
| Standard effect with fixed parameters | No -- let AE cache (default behavior) |
| Plugin reads config file to determine params | Yes -- disable caching |
| License check changes effect availability | Yes -- disable caching |
| Plugin name depends on external resource | Yes -- disable caching |
| Dynamic parameter count based on runtime state | Yes -- disable caching |

> **Pitfall:** Disabling cache-on-load increases AE startup time because the plugin binary must be loaded and its `PF_Cmd_GLOBAL_SETUP` executed every launch. Only disable caching when truly necessary.

> **Pitfall:** The `effectAvailable` parameter with a value of 0 effectively hides the effect from AE. This can be used for conditional availability (e.g., hiding the effect if a license check fails), but be careful -- an effect that is always unavailable will confuse users who see it installed but cannot find it in the Effects menu.

---

## Compute Cache Suite: The Full Picture

Since the Hash Suite is designed to work with it, a brief overview of `AE_ComputeCacheSuite.h` (frozen in AE 18.2) is warranted.

### Registration

```c
#define kAEGPComputeCacheSuite          "AEGP Compute Cache"
#define kAEGPComputeCacheSuiteVersion1  1   // Frozen in AE 18.2
```

### Compute Class Registration

Plugins register a "compute class" with four callbacks:

```c
typedef struct AEGP_ComputeCacheCallbacks {
    A_Err (*generate_key)(
        AEGP_CCComputeOptionsRefconP optionsP,
        AEGP_CCComputeKeyP           out_keyP);

    A_Err (*compute)(
        AEGP_CCComputeOptionsRefconP optionsP,
        AEGP_CCComputeValueRefconP   *out_valuePP);

    size_t (*approx_size_value)(
        AEGP_CCComputeValueRefconP valueP);

    void (*delete_compute_value)(
        AEGP_CCComputeValueRefconP valueP);
} AEGP_ComputeCacheCallbacks;
```

| Callback | Purpose |
|----------|---------|
| `generate_key` | Build the cache key from options (use Hash Suite here) |
| `compute` | Perform the expensive computation |
| `approx_size_value` | Report memory size for cache eviction heuristics |
| `delete_compute_value` | Free the computed value when cache purges it |

### Checkout Patterns

The suite supports two checkout strategies:

#### Single Value Checkout

```c
AEGP_CCCheckoutReceiptP receipt = NULL;
err = cacheSuite->AEGP_ComputeIfNeededAndCheckout(
    "my.compute.class",
    myOptions,
    true,       // wait for other thread if computing
    &receipt);

if (!err) {
    AEGP_CCComputeValueRefconP value = NULL;
    cacheSuite->AEGP_GetReceiptComputeValue(receipt, &value);

    // Use the cached/computed value
    MyResult *result = (MyResult *)value;

    cacheSuite->AEGP_CheckinComputeReceipt(receipt);
}
```

#### Multi-Checkout (Avoiding Serialization)

When a render needs multiple cache values, this pattern keeps threads busy:

```c
// Phase 1: Non-blocking checkout attempts
AEGP_CCCheckoutReceiptP receipt1 = NULL, receipt2 = NULL;
A_Err err1 = cacheSuite->AEGP_ComputeIfNeededAndCheckout(
    classId, options1, false, &receipt1);  // do NOT wait
A_Err err2 = cacheSuite->AEGP_ComputeIfNeededAndCheckout(
    classId, options2, false, &receipt2);  // do NOT wait

// Phase 2: Wait for any that were not immediately available
if (err1 == A_Err_NOT_IN_CACHE_OR_COMPUTE_PENDING) {
    err1 = cacheSuite->AEGP_ComputeIfNeededAndCheckout(
        classId, options1, true, &receipt1);  // wait
}
if (err2 == A_Err_NOT_IN_CACHE_OR_COMPUTE_PENDING) {
    err2 = cacheSuite->AEGP_ComputeIfNeededAndCheckout(
        classId, options2, true, &receipt2);  // wait
}

// Phase 3: Use both values
// ...

// Phase 4: Check in both receipts
cacheSuite->AEGP_CheckinComputeReceipt(receipt1);
cacheSuite->AEGP_CheckinComputeReceipt(receipt2);
```

The `AEGP_CheckoutCached` function provides a non-computing lookup -- it returns the cached value if present, or `A_Err_NOT_IN_CACHE_OR_COMPUTE_PENDING` if not. This is useful for polling patterns (e.g., a UI thread checking whether a render thread has populated a histogram cache).

> **Pitfall:** Receipts must be checked in before returning to the host. Failing to check in a receipt leaks a reference and prevents cache purging.

> **Pitfall:** When unregistering a compute class with `AEGP_ClassUnregister`, all cached values are purged immediately because the `delete_compute_value` callback will no longer be available. Ensure no checkout receipts are outstanding when unregistering.

---

## Suite Version Compatibility Matrix

| Suite | Version | Frozen In | Key Addition |
|-------|---------|-----------|--------------|
| PF_AdvAppSuite | 1 | AE 5.0 | Core app control, info panel |
| PF_AdvAppSuite | 2 | AE 6.0 | `PF_AppendInfoText` |
| PF_AdvTimeSuite | 1 | AE 5.0 | Basic time formatting |
| PF_AdvTimeSuite | 2 | -- | Updated display prefs struct |
| PF_AdvTimeSuite | 3 | AE 14.2 | Version3 prefs (A_long fields) |
| PF_AdvTimeSuite | 4 | AE 15.0 | `PF_TimeCountFrames` |
| PF_AdvItemSuite | 1 | AE 5.0 | Time stepping, rerender, state query |
| AEGP_HashSuite | 1 | AE 17.5.1 | Hash creation and mixing |
| PF_CacheOnLoadSuite | 1 | -- | Startup cache control |
| AEGP_ComputeCacheSuite | 1 | AE 18.2 | Full compute cache system |

When targeting older AE versions, always attempt to acquire the latest version first, then fall back:

```c
PF_AdvTimeSuite4 *timeSuite4 = NULL;
PF_AdvTimeSuite3 *timeSuite3 = NULL;

SPErr err = pica->AcquireSuite(kPFAdvTimeSuite, kPFAdvTimeSuiteVersion4, (const void **)&timeSuite4);
if (err != kSPNoError) {
    // Fall back to version 3
    err = pica->AcquireSuite(kPFAdvTimeSuite, kPFAdvTimeSuiteVersion3, (const void **)&timeSuite3);
}
```

---

## Summary

These three headers address distinct needs:

- **AE_AdvEffectSuites.h** provides application-level control that effect plugins occasionally need: marking projects dirty, formatting time strings, drawing to the info panel, and forcing re-renders. These are "escape hatches" from the pure pixel-processing world of effects into the application UI layer.

- **AE_HashSuite.h** provides the building blocks for deterministic cache keys. Combined with the Compute Cache Suite, it enables plugins to cache expensive computations (histograms, LUT generation, spatial analysis) across frames and render threads without re-computing identical results.

- **AE_CacheOnLoadSuite.h** is a single-function suite for the rare case where a plugin needs to be freshly loaded at every AE launch rather than having its metadata cached.

Most plugins will use the AdvApp suite for info panel drawing and project dirty flagging. The Hash and Compute Cache suites are for performance-critical plugins that perform expensive pre-computations. The CacheOnLoad suite is for plugins with dynamic configuration.
