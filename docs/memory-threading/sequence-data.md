# Sequence Data

> 116 Q&As · source: AE plugin dev community Discord

### What is the Compute Cache API and how does it differ from sequence data and arb data?

Compute Cache replaces sequence data for MFR-compatible caching. Key differences: (1) Compute Cache can be read/written from any thread (render or UI), (2) it is NOT saved with the project (unlike arb data), (3) it's designed for per-frame computed values. Register it in GlobalSetup with AEGP_ClassRegister. The key function should include effect ref, layer ID, effect position, current time, and time scale. Pass in_data->pica_basicP through the options refcon pointer to access suites inside callback functions. Must be registered during GlobalSetup, first access possible during ParamSetup.

```cpp
#include "AE_ComputeCacheSuite.h"

struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
};

A_Err MyGenerateKeyFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeKeyP out_keyP) {
    access_cache_data* accessCacheData = static_cast<access_cache_data*>(optionsP);
    SPBasicSuite* bsuite = accessCacheData->in_data->pica_basicP;
    // Generate key using AEGP_HashSuite1...
    return A_Err_NONE;
}

static PF_Err GlobalSetup(...) {
    AEFX_SuiteScoper<AEGP_ComputeCacheSuite1> compute_suite(in_data, kAEGPComputeCacheSuite, kAEGPComputeCacheSuiteVersion1);
    AEGP_ComputeCacheCallbacks callbacks = { MyGenerateKeyFunc, MyComputeFunc, MyApproxSizeValueFunc, MyDeleteComputeValueFunc };
    compute_suite->AEGP_ClassRegister("com.mycompany.effect.myComputeCacheClass", &callbacks);
}
```

*Tags: `arb-data`, `caching`, `compute-cache`, `mfr`, `sequence-data`, `thread-safety`*

---

### How do you read sequence data in SmartRender since AE CC2021/2022?

Since CC2021/2022, in_data->seq_data is not correctly updated in render threads. You must use PF_EffectSequenceDataSuite1 with PF_GetConstSequenceData instead. The const handle must be locked with PF_LOCK_HANDLE before use. Note: According to Adobe engineers, PF_LOCK_HANDLE hasn't done anything since CS6 (it's a dummy function), but the code pattern of casting via the handle is still necessary for correct access.

```cpp
PF_ConstHandle const_seq;
AEFX_SuiteScoper<PF_EffectSequenceDataSuite1> seqdata_suite =
    AEFX_SuiteScoper<PF_EffectSequenceDataSuite1>(
        in_data,
        kPFEffectSequenceDataSuite,
        kPFEffectSequenceDataSuiteVersion1,
        out_data);
seqdata_suite->PF_GetConstSequenceData(in_data->effect_ref, &const_seq);
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `mfr`, `pf-get-const-sequence-data`, `sequence-data`, `smart-render`, `threading`*

---

### How can I identify a specific effect instance using AEGP_EffectCallGeneric and PF_Cmd_COMPLETELY_GENERAL?

AEGP_EffectCallGeneric has a limitation: you cannot call it from an effect to another instance of the same effect type. Even bouncing through a separate AEGP via a custom suite doesn't work if the call chain originates from the same effect type. For instance identification, alternatives include: (1) Put an ID in sequence data and access via AEGP_EffectCallGeneric from a different effect type. (2) Change a hidden param's name to serve as identifier using AEGP_SetStreamName() (doesn't survive save/load). (3) Use a separate AEGP to handle identification during idle processing.

*Tags: `aegp`, `completely-general`, `effect-instance`, `identification`, `sequence-data`*

---

### How can I set a parameter value when an effect is first applied (during ParamSetup or SequenceSetup)?

ParamSetup is called once per session per effect type, not per instance, so you can't set instance-specific values there. SequenceSetup also lacks a specific instance association - the params array contains junk data and acquiring an AEGP_EffectRef will crash. The solution is to set a flag in new sequence data saying 'this is a brand new instance', then check that flag during idle_hook or UPDATE_PARAMS_UI and make changes accordingly. For unique instance IDs, use sequence data as each sequence setup call is triggered only once per instance when created.

*Tags: `initialization`, `instance-id`, `param-setup`, `sequence-data`, `sequence-setup`*

---

### Why is sequence data different between PF_Cmd_EVENT (UI thread) and SmartPreRender (render thread)?

Sequence data is separate on the render and UI threads by design. The render threads are free to render asynchronously without having their data change from another thread. Global data IS shared between render and UI threads, though it's not instance-specific. To share per-instance data: store data in global data keyed by instance identifiers (comp item ID + layer ID + effect index on layer), protected by a mutex. Comp item IDs don't change during a session. Layer IDs don't change on reorder but may change on copy/paste. Effect index changes when moved in the stack.

*Tags: `global-data`, `mutex`, `render-thread`, `sequence-data`, `threading`, `ui-thread`*

---

### Why is PF_Cmd_SEQUENCE_FLATTEN called when an effect is first applied, and why are Sequence Resetup/Setdown called multiple times on different threads?

This is correct behavior when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set. AE flattens sequence data so it can be copied (for render thread copies). Since AE 2015, rendering uses separate threads with separate project copies. Resetup may receive unflat data, so tag your data to indicate flat/unflat state. Use PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA to allow AE to get the flat copy without losing the original unflat one, reducing unnecessary resetup calls. During sequence setup/resetup, don't assume the plugin can find its own layer/comp - AE sometimes constructs the instance before associating it to a layer. For unique instance IDs, flag sequence data as 'requires checking' on resetup, then scan for duplicates when the instance can be located.

*Tags: `flatten`, `instance-id`, `mfr`, `resetup`, `sequence-data`, `threading`*

---

### Can I save sequence data during PF_Cmd_SEQUENCE_SETDOWN?

No. Sequence setdown is AE's signal to destruct and free the data, not to save it. The data saved with the project is the last handle used or flattened. To have data saved with the project, set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING during global setup to receive PF_Cmd_SEQUENCE_FLATTEN calls, where you serialize your data into a flat handle that AE will store.

*Tags: `flatten`, `persistence`, `project-save`, `sequence-data`, `sequence-setdown`*

---

### How does PF_Cmd_SEQUENCE_RESETUP work and when is sequence data flattened?

PF_Cmd_SEQUENCE_RESETUP is called either when opening a project with saved (flat) sequence data, or after AE has just asked to flatten sequence data (for a save) and now needs to unflatten it. The in_data->sequence_data pointer will always point to flattened data at this point, and your task is to unflatten it. There is a flag in the input data that tells you if data is flat or not. The documentation phrase 're-create (usually unflatten) sequence data' means that for simple plugins that only use regular parameters and don't need flattening/unflattening, the unflatten step is unnecessary. The input data should always be flat and the output data should always be unflat for this function.

```cpp
// We got here because we're either opening a project w/saved (flat) sequence data,
// or we've just been asked to flatten our sequence data (for a save) and now we're blowing it back up.
```

*Tags: `flatten`, `project-save`, `sequence-data`, `sequence-resetup`, `unflatten`*

---

### When should the first frame be stored and how do you physically store an EffectWorld in sequence data?

The first frame is stored in the render function. You can store an EffectWorld directly in your sequence_data struct. On the first render, allocate a new world using PF_NEW_WORLD and copy the result into it. On subsequent frames, you can reuse and update this cached world. Use a hash or flag to track whether the result has been loaded and whether parameters have changed.

```cpp
typedef struct {
    PF_EffectWorld  resultW;
    bool            result_loaded = false;
    A_long          result_last_time;
    char            result_hash[260];
} sequenceData;

if (!seqdataP->result_loaded) {
    AEFX_CLR_STRUCT(seqdataP->resultW);
    ERR(PF_NEW_WORLD(tensorW.width, tensorW.height, PF_NewWorldFlag_NONE, &seqdataP->resultW));
}
PF_COPY(&cachingW, &seqdataP->resultW, NULL, NULL);
seqdataP->result_loaded = true;
```

*Tags: `caching`, `memory`, `render-loop`, `sequence-data`*

---

### How do you access and write to sequence data within the Render function?

You can access sequence data in the Render function by dereferencing the sequence_data pointer provided in out_data. Cast it to your sequence data struct type. You can then read from and write to any members of that struct throughout the render call. However, MFR may have implications for this workflow, so test accordingly.

```cpp
sequenceData *seqdataP = NULL;
seqdataP = *(sequenceData **)out_data->sequence_data;
// Now you can read/write to seqdataP->resultW, seqdataP->result_loaded, etc.
```

*Tags: `caching`, `render-loop`, `sequence-data`*

---

### How do you detect when to invalidate a cached frame and prevent using stale results?

Create a hash string from all the important parameter values and settings that affect the output (time_step, time_scale, quality, downsample_x, downsample_y, and any non-keyframable settings like LOD). On each render, compare the current hash with the one from the last frame. If they differ or if the current time is not exactly one frame after the last cached time, reset the result_loaded flag to false.

```cpp
A_char result_hash[260];
sprintf(result_hash, "%d%d%d%d%d%i",
    in_data->time_step,
    in_data->time_scale,
    in_data->quality,
    in_data->downsample_x,
    in_data->downsample_y,
    lod
);

if (strcmp(result_hash, seqdataP->result_hash) != 0 || current_time != (seqdataP->result_last_time + in_data->time_step)) {
  seqdataP->result_loaded = false;
}
seqdataP->result_last_time = current_time;
strcpy(seqdataP->result_hash, result_hash);
```

*Tags: `caching`, `params`, `sequence-data`*

---

### How can sequence_data be used to set values in the output data?

The out_data parameter has sequence_data as a property that can be used to set the values of sequence data. This functionality works in After Effects 2021 and will also work in 2022 if you don't set the MFR flag, though it may show a warning in 2022.

*Tags: `mfr`, `params`, `sequence-data`*

---

### How do plugins like Trapcode Particular cache and calculate multiple frames over time rather than just the last frame?

Plugins that need to cache multiple frames typically use their own internal state management system separate from the standard Sequence Data approach. While Sequence Data can store the last frame, plugins handling fluid simulations or particle systems that need to jump to arbitrary times in a composition use custom caching mechanisms. These plugins maintain their own frame cache, calculate values for multiple frames, and store them in memory structures that can be accessed when jumping to different times in the composition. The exact implementation depends on the plugin's architecture, but it generally involves managing a larger state buffer that holds multiple frames of simulation data rather than relying solely on the sequential frame-to-frame Sequence Data mechanism.

*Tags: `caching`, `memory`, `render-loop`, `sequence-data`*

---

### Can I save frames to arbitrary file formats and read them back in when caching Effect_World in sequence data?

Yes, you can save frames into any file format you want and read them back in. This approach allows you to work around sequence data storage limitations when caching multiple frames, similar to using a .dat file example.

*Tags: `caching`, `memory`, `sequence-data`*

---

### Can you store many PF_EffectWorld structures in a sequence data struct for caching purposes?

Yes, you can store thousands of PF_EffectWorld structures in your data struct. This can be used to cache frames by creating an array like PF_EffectWorld resultW[10000] and PF_EffectWorld matte[10000]. You can then implement logic in your main render function to cache all missing previous frames when a later frame is requested, using a loop to iterate and populate the cached frames before rendering the current frame.

```cpp
typedef struct {
    PF_EffectWorld  resultW[10000];
    PF_EffectWorld  matte[10000];
    bool            result_loaded = false;
    bool            isCaching = false;
    A_long          result_last_time;
    char            result_hash[260];
    int                firstFrame;
    bool            veryFirstLaunch;
    PF_InData        seqIn_data;
    PF_SampPB    seqSamp_pb;
} sequenceData;
```

*Tags: `caching`, `memory`, `sequence-data`, `smart-render`*

---

### What is the issue when trying to checkout multiple frames of a layer into an array during smart render?

When attempting to checkout multiple frames of a layer (such as thousands) using PF_CHECKOUT_PARAM in a loop, you will typically only succeed in filling about 12 matte worlds before encountering errors or bad callback parameters. This is a limitation of the checkout system - it is not designed to batch checkout multiple frames of the same layer into an array. The layer checkout system expects individual checkouts rather than mass array-based storage of the same layer across many frames.

```cpp
if (&seqdataP->matte[0].data != NULL && &checkout.u.ld.data != NULL) {
        for (int m = 0; m < 10000; m++) {
            if (int(in_data->time_scale*m) < int(in_data->total_time)) {
                    ERR(PF_CHECKOUT_PARAM(in_data,
                        VECTORMATTELAYER_DISK_ID,
                        int(in_data->time_scale * m),
                        in_data->time_step,
                        in_data->time_scale,
                        &checkout));
                    seqdataP->matte[m] = checkout.u.ld;
            }
        }
    }
```

*Tags: `layer-checkout`, `memory`, `sequence-data`, `smart-render`*

---

### Does changing image size or color depth affect the frame number where a cache-related crash occurs?

The suggestion is to test whether changing image size or color depth of the project changes the crash frame number, as this could indicate whether RAM access is being handled correctly with time-varying data, or if there's a conflict between sequence data writing and MFR.

*Tags: `caching`, `debugging`, `memory`, `mfr`, `sequence-data`*

---

### Can you write to sequence data during PF_update_param_ui with MFR?

No, you cannot write to sequence data during PF_update_param_ui with MFR. You can only do it in the user changed param callback. If you need to modify layer defaults, you may need to use AEGP instead.

*Tags: `aegp`, `mfr`, `params`, `sequence-data`*

---

### How do you invalidate rendered frames in smart render when external data changes?

Set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag and use extra->cb->GuidMixInPtr in pre-render to scan data that should trigger invalidation. You can add multiple GuidMixInPtr calls for multiple data sources. Store data that affects rendering in sequence data or static variables, not in globalData, since After Effects expects globalData to remain unchanged outside of globalsetup/setdown.

```cpp
if (extra->cb->GuidMixInPtr && !err) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan_to_force_render), reinterpret_cast<void*>(*data_to_scan_to_force_render));
}
```

*Tags: `mfr`, `render-loop`, `sequence-data`, `smart-render`*

---

### What is the difference between in_data->sequence_data and out_data->sequence_data?

in_data->sequence_data is where you read sequence data. out_data->sequence_data is where you write to sequence data. In CC2022 and later, there are special functions for reading/writing. You can only write to sequence data in specific contexts: sequence setup, resetup, user changed param, do dialog, or external_dependencies callbacks. Use a static variable instead if you need to modify data more freely.

*Tags: `aegp`, `params`, `sequence-data`*

---

### Is sequence data flattened or not?

Technically yes, sequence data can be flattened. However, since CC2022, non-flattened sequence data may not be refreshed as expected, as noted in the supervisor example.

*Tags: `render-loop`, `sequence-data`*

---

### How can frame N access custom data computed from the previous frame, accounting for previous effects in the chain?

The user explored using checkout Param during compute cache, but noted this approach doesn't account for previous effects. The solution involves retrieving data computed from the previous frame (same size as a frame) through the effect chain.

*Tags: `compute-cache`, `layer-checkout`, `render-loop`, `sequence-data`*

---

### What are the advantages of compute cache compared to arbdata for storing pointers to heap objects that are flattened/unflattened?

Compute cache can be written and read from any thread (render or UI thread) using std::atomic for thread safety. Unlike arbdata, compute cache is not stored with the project. Compute cache is designed to get a value per frame, whereas arbdata requires a global context where you define arrays or vectors to store data for different frames. Compute cache replaced sequence data after MFR was introduced since sequence data can no longer be used.

*Tags: `arb-data`, `compute-cache`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### Can data from compute cache be transferred to arbdata to persist it in the project?

Yes, data from the compute cache can be transferred to arbdata if needed so it gets saved in the project. You can read the compute cache from arb or sequence thread if you want.

*Tags: `arb-data`, `compute-cache`, `sequence-data`*

---

### How can instance-specific data be stored in a project file and accessed in global or paramssetup?

Arb data can be used for this purpose. Arb data can be accessed using special group functions for access and save operations, allowing you to store and retrieve instance-specific data that persists in the project file.

*Tags: `arb-data`, `globalsetup`, `params`, `sequence-data`*

---

### How should a plugin handle processing that requires the previous frame's data (frame n-1 for frame n)?

You have two main approaches: (1) Use sequence data, which is designed for simulation plugins that need to access previous frames, or (2) Check out the input frame n-1 at render time if you only need the pre-effect input frame rather than your plugin's output from frame n-1. If you need your plugin's own output from frame n-1, you'll need to store intermediate results and manage sequential execution, as After Effects has no built-in way to force sequential frame processing.

*Tags: `caching`, `layer-checkout`, `render-loop`, `sequence-data`*

---

### How can a plugin maintain a unique ID per instance when duplicating, and why does sequence data become out of sync between the UI thread and render threads?

Sequence data is the appropriate mechanism for storing per-instance unique IDs. When duplicating, catch the duplication event and assign a new unique ID to the sequence data. However, sequence data may become out of sync on render threads even though it updates correctly in UpdateUI and UserChangeParams. Two potential solutions: (1) Check if the sequence is flattened in the project, as this can force render threads to use the correct sequence value, and (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of directly accessing in_data->seq_data, as the latter has not been updated correctly since CC 2021/2022.

*Tags: `params`, `render-loop`, `sequence-data`, `threading`*

---

### How does After Effects store arbitrary data of a plugin to disk when that plugin is not installed?

Arbitrary data is serialized via PF_Cmd_Arbitrary_Callback using the flatten function, similar to sequence data. However, the flatten/unflatten versions are not accessible via AEGP functions. For plugins like 'curves' with stack variable handles, the data can be reinterpreted cast as char* and serialized. However, for plugins like 'liquify' that have pointers in their struct, serialization is problematic because deserialized pointer addresses are invalid on reload and will cause crashes.

*Tags: `aegp`, `arb-data`, `params`, `sequence-data`*

---

### How should sequence_data be accessed in SmartRender when it appears as nullptr?

Since After Effects v22, you must use the PF_EffectSequenceDataSuite to read sequence data in SmartRender. Create an AEFX_SuiteScoper for PF_EffectSequenceDataSuite1, call PF_GetConstSequenceData with the effect_ref to get a const handle, then lock that handle to access the sequence data. This provides read-only access to the sequence data.

```cpp
PF_ConstHandle const_seq;
AEFX_SuiteScoper<PF_EffectSequenceDataSuite1> seqdata_suite =
    AEFX_SuiteScoper<PF_EffectSequenceDataSuite1>(
        in_data,
        kPFEffectSequenceDataSuite,
        kPFEffectSequenceDataSuiteVersion1,
        out_data);
seqdata_suite->PF_GetConstSequenceData(in_data->effect_ref, &const_seq);
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `aegp`, `memory`, `sequence-data`, `smart-render`*

---

### Does using sequence data in SmartRender have a negative impact on performance?

No significant performance impact was observed. The read-only access pattern through the PF_EffectSequenceDataSuite in SmartRender does not cause noticeable performance degradation.

*Tags: `performance`, `sequence-data`, `smart-render`*

---

### Do you need to lock the handle when calling GetConstSequenceData for sequence data?

Yes, you need to lock the handle even when reading only. Use PF_LOCK_HANDLE(const_seq) to lock the handle before accessing it, and unlock it after use. This is necessary in multi-frame render (MFR) scenarios with multiple plugin applications, and the lack of locking can cause nullptr returns and crashes. While Adobe documentation doesn't clearly document this requirement, it appears to be essential for thread safety.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `aegp`, `debugging`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### Should sequence data be flattened when using the sequence data suite?

Flattening sequence data is recommended and should work in most cases. However, it's not strictly required - some developers report success without flattening it. The flattening approach is more commonly used and tested.

*Tags: `params`, `sequence-data`*

---

### How can a C++ plugin update a text layer every frame after rendering completes?

This is not cleanly possible due to AE's project locking during render operations. Possible workarounds include: (1) writing data to a compute cache or sequence data and reading it from the UI thread via the aegp_idle hook, (2) using AEGP_SetText from the UI thread instead of the render thread, (3) rendering text yourself in the render thread using an external library, or (4) saving to disk and having a ScriptUI panel with a timer read the data. The aegp_idle hook approach may not work without a GUI or in MediaEncoder. Setting text from the render thread causes a locked project error.

*Tags: `aegp`, `compute-cache`, `render-loop`, `sequence-data`, `threading`, `ui`*

---

### What resources exist to understand sequence data in After Effects plugin development?

Tobias Fleischer (reduxFX) has created comprehensive documentation about sequence data that has helped developers understand the concept clearly. This resource is considered essential reference material for working with sequence data in After Effects plugins.

*Tags: `aegp`, `documentation`, `reference`, `sequence-data`*

---

### What are the main challenges when updating GPU plugins for MFR compatibility?

GPU plugins face significant challenges with MFR that CPU plugins don't encounter. Issues include crashes that occur only after long rendering periods (making them hard to debug), GPU caching complications, and sequence data handling. OpenGL-based plugins particularly struggle with Mac-specific issues. James Whiffin reported that even using Adobe's GLator sample, Mac OpenGL compatibility remains problematic. The recommendation is to consider maintaining completely separate codebases: one for Metal on Mac and one for Windows, as OpenGL performance on Mac can be severely degraded compared to Windows.

*Tags: `caching`, `gpu`, `macos`, `metal`, `mfr`, `opengl`, `sequence-data`, `windows`*

---

### Is MFR compatibility straightforward for simple plugins that don't use sequence data or caching?

Yes, for simple plugins without sequence data or complex caching requirements, MFR compatibility is straightforward. Gabgren reported that after the initial panic about the MFR announcement, updating simple plugins required little more than a rebuild. Complex plugins using sequence data or advanced features require more significant refactoring, but basic plugins can be updated quickly.

*Tags: `caching`, `mfr`, `sequence-data`*

---

### How should you store GPU and CPU data across multiple frames in After Effects plugins?

For GPU work, store data in global data or sequence data and access it later. For CPU work, cache the PF_EffectWorld in the global or sequence data. This approach may need validation against the new 3-way-checkout mechanism.

*Tags: `caching`, `gpu`, `memory`, `sequence-data`, `smartfx`*

---

### How do you store and access the previous frame's EffectWorld in sequence data within the Render function?

You can store an EffectWorld in your sequence_data struct and access it during render(). Initialize the struct with fields for the cached result world, a boolean flag indicating if it's loaded, the last render time, and a hash string. In SequenceSetup, initialize these fields. In Render, check if the hash or frame timing has changed to determine if you should invalidate the cache. If the cache is valid and needed, use the stored resultW from sequence data. Store the new render result back into sequence data for the next frame using PF_NEW_WORLD and PF_COPY. Note: MFR may affect this workflow, and if you need to access the previous source frame (rather than the previous result), you'll need to use a suite instead.

```cpp
typedef struct {
    PF_EffectWorld  resultW;
    bool            result_loaded = false;
    A_long          result_last_time;
    char            result_hash[260];
} sequenceData;

static PF_Err Render(...) {
    sequenceData *seqdataP = *(sequenceData **)out_data->sequence_data;
    
    A_char result_hash[260];
    sprintf(result_hash, "%d%d%d%d%d%i",
        in_data->time_step,
        in_data->time_scale,
        in_data->quality,
        in_data->downsample_x,
        in_data->downsample_y,
        lod);
    
    // Reset cache if hash differs or frame is not sequential
    if (strcmp(result_hash, seqdataP->result_hash) != 0 || 
        current_time != (seqdataP->result_last_time + in_data->time_step)) {
        seqdataP->result_loaded = false;
    }
    
    seqdataP->result_last_time = current_time;
    strcpy(seqdataP->result_hash, result_hash);
    
    bool use_last_result = seqdataP->result_loaded && params[PARAM_ADDITIVE]->u.bd.value;
    
    if (use_last_result) {
        // Use seqdataP->resultW from previous frame
    }
    
    PF_EffectWorld cachingW;
    AEFX_CLR_STRUCT(cachingW);
    ERR(PF_NEW_WORLD(tensorW.width, tensorW.height, PF_NewWorldFlag_NONE, &cachingW));
    PF_COPY(&tensorW, &cachingW, NULL, NULL);
    
    if (!seqdataP->result_loaded) {
        AEFX_CLR_STRUCT(seqdataP->resultW);
        ERR(PF_NEW_WORLD(tensorW.width, tensorW.height, PF_NewWorldFlag_NONE, &seqdataP->resultW));
    }
    
    PF_COPY(&cachingW, &seqdataP->resultW, NULL, NULL);
    seqdataP->result_loaded = true;
    ERR(PF_DISPOSE_WORLD(&cachingW));
}
```

*Tags: `caching`, `memory`, `render-loop`, `sequence-data`*

---

### What is the Transfusion plugin and how does it demonstrate sequence data caching?

Transfusion is an After Effects plugin by gabgren that demonstrates caching the EffectWorld in sequence data across frames. The plugin stores the result of each render call in sequence_data, then uses that cached result on the next frame for additive effects. This approach works for 8-bit rendering without MFR (Multi-Frame Rendering), though MFR compatibility may require adjustments to this workflow.

*Tags: `caching`, `open-source`, `reference`, `sequence-data`*

---

### How can you set sequence data values in After Effects plugins?

You can use the out_data property which has sequence_data as a property that can be used to set the values of sequence data. This approach works in After Effects 2021 and 2022 (in 2022, it will work without the MFR flag set, though a warning may be shown).

*Tags: `aegp`, `mfr`, `sequence-data`*

---

### What is the Compute Cache useful for in After Effects plugin development?

The Compute Cache is a useful learning resource for understanding how to work with sequence data and plugin state management in After Effects plugins.

*Tags: `compute-cache`, `reference`, `sequence-data`*

---

### How do plugins like Trapcode Particular cache multiple frames over time for features like fluid simulation?

Plugins that need to cache multiple frames typically use the Sequence Data API to store frame information persistently. While a basic setup might store only the last frame in Sequence Data, plugins like Trapcode Particular that perform simulations (such as fluid dynamics) across many frames likely store cumulative or checkpoint data in the Sequence Data structure. This allows them to jump to any point in the composition and recalculate or retrieve cached simulation values. The key is leveraging Sequence Data's ability to persist data across multiple frames rather than just the immediate previous frame.

*Tags: `arb-data`, `caching`, `memory`, `sequence-data`*

---

### Can I save cached frames to a custom file format and read them back in a plugin?

Yes, you can save frames to any file format you want and read them back in. This is a common approach when caching large amounts of frame data that won't fit in sequence data. You can use custom binary formats (like .dat files) or other formats suitable for your needs. The plugin can write frames to disk during processing and read them back as needed, avoiding the limitations of storing arbitrarily large numbers of frames directly in sequence data.

*Tags: `caching`, `compute-cache`, `mfr`, `sequence-data`*

---

### How can you persist sequence data without consuming memory?

You can dump sequence data to a file (encoded or not) and read it back as needed. This approach allows you to save a sequence of data without loading it all into memory at once.

*Tags: `caching`, `file-io`, `memory`, `sequence-data`*

---

### How can you store and cache multiple PF_EffectWorlds in a sequence data structure for frame-by-frame processing?

You can store arrays of PF_EffectWorld structures in your sequence data struct (e.g., PF_EffectWorld resultW[10000]) along with tracking variables like result_loaded, isCaching, result_last_time, and result_hash. In your main render function, implement a loop that caches missing previous frames when a later frame is requested by checking the last calculated frame and iterating through frames using IterateFloatSuite1()->iterate() to render each intermediate frame into the cached array before processing the currently requested frame.

```cpp
typedef struct {
    PF_EffectWorld  resultW[10000];
    PF_EffectWorld  matte[10000];
    bool            result_loaded = false;
    bool            isCaching = false;
    A_long          result_last_time;
    char            result_hash[260];
    int                firstFrame;
    bool            veryFirstLaunch;
    PF_InData        seqIn_data;
    PF_SampPB    seqSamp_pb;
} sequenceData;
```

*Tags: `caching`, `memory`, `render-loop`, `sequence-data`*

---

### What is the correct approach for checking out multiple frames of a layer parameter during Smart Render for frame-based caching?

When trying to checkout multiple frames of a layer parameter (like a matte layer) during Smart Render checkins/checkouts, the standard PF_CHECKOUT_PARAM loop has limitations and typically fails after ~12 successful checkouts due to callback parameter constraints. The recommended approach is to checkout layers during the Smart Render process frame-by-frame rather than attempting to batch checkout all frames in a single loop, or to implement alternative caching strategies that respect the plugin architecture's layer checkout limitations.

```cpp
if (&seqdataP->matte[0].data != NULL && &checkout.u.ld.data != NULL) {
        for (int m = 0; m < 10000; m++) {
            if (int(in_data->time_scale*m) < int(in_data->total_time)) {
                    ERR(PF_CHECKOUT_PARAM(in_data,
                        VECTORMATTELAYER_DISK_ID,
                        int(in_data->time_scale * m),
                        in_data->time_step,
                        in_data->time_scale,
                        &checkout));
                    seqdataP->matte[m] = checkout.u.ld;
            }
        }
    }
```

*Tags: `caching`, `layer-checkout`, `memory`, `sequence-data`, `smart-render`*

---

### How can you diagnose whether a crash is related to RAM caching issues in After Effects plugins?

Try changing the size of the image or color depth of the project to see if the crash frame number changes. This can help determine if the issue is related to RAM access handling, particularly with time-varying data or conflicts between sequence data writing and MFR (Multiprocessing Framework).

*Tags: `caching`, `debugging`, `memory`, `mfr`, `sequence-data`*

---

### Can you write to sequence data during PF_update_param_ui callback?

No, you cannot write to sequence data during PF_update_param_ui. Writing to sequence data is only possible in the user changed param callback. With MFR, you also cannot write to sequence data during PF_update_param_ui. However, in certain cases you may be able to accomplish similar goals without AEGP by using PF_LayerDefault_NONE or similar approaches, though AEGP is often the recommended solution.

*Tags: `aegp`, `mfr`, `params`, `sequence-data`*

---

### How do you invalidate rendered frames when data changes in a plugin using smart render?

Set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag and use GuidMixInPtr in pre-render to scan data that should force re-rendering. In pre-render, call: if (extra->cb->GuidMixInPtr && !err) { extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan), reinterpret_cast<void*>(*data_to_scan)); }. You can add multiple GuidMixInPtr calls for different data sources. This works particularly well when data is stored in sequence data structures.

```cpp
if (extra->cb->GuidMixInPtr && !err) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan_to_force_render), reinterpret_cast<void*>(*data_to_scan_to_force_render));
}
```

*Tags: `params`, `render-loop`, `sequence-data`, `smart-render`*

---

### What are the constraints on writing to sequence_data in After Effects CC2022 and later?

In CC2022 and later, you cannot write to sequence_data whenever you want. Writing is only allowed during specific events: sequence setup, sequence resetup, user changed param, do dialog, and external dependencies. For unrestricted modifications, use a global static variable initialized once instead. Note that in_data->sequence_data is for reading, while out_data->sequence_data is for writing.

*Tags: `memory`, `params`, `sequence-data`*

---

### How can you trigger frame invalidation from external background threads in After Effects plugins?

GuidMixInPtr can be problematic for triggering invalidation from background threads when using global or sequence data variables. A reliable workaround is to change a composition property (like background color) via AEGP calls or scripting, which forces After Effects to check if frames need invalidation. Another approach is to use a hidden parameter that gets modified through AEGP or scripting calls, though this has limitations when modal dialogs are open. The method must ultimately signal to After Effects through a mechanism it actively monitors.

*Tags: `aegp`, `params`, `sequence-data`, `smart-render`, `threading`*

---

### How do you force After Effects to re-check whether frames need invalidating when using sequence data?

When sequence data isn't flattened, After Effects may not refresh frames as expected even when the data updates properly (verified via breakpoints). A workaround is to give AE a "kick" by changing composition background color or toggling a parameter, which forces it to check for invalidation. Alternatively, see the Adobe Community discussion on forcing rendering on events in separate threads: https://community.adobe.com/t5/after-effects-discussions/force-rendering-on-an-event-in-a-separate-thread-coming-back-to-the-main-ui-thread-on-an-event/m-p/13628197

*Tags: `debugging`, `render-loop`, `sequence-data`, `threading`*

---

### How can a plugin implement recursive operations where frame N depends on computed data from previous frames without checking out all intermediate frames during pre-render?

This is a known challenge when combining smart render with compute caches that require sequential frame dependencies. The user attempted several approaches: (1) using a compute cache with smart render, but this requires checking out all previous frames during pre-render when rendering frame 100, which is memory-inefficient; (2) using checkout params during compute cache, which doesn't account for previous effects; (3) using AEGP_CacheAndCheckoutFrame, which doesn't trigger pre_render/smart render callbacks on checked-out frames. A solution may involve manually managing frame dependencies outside the smart render system or using layer checkout strategically to load only necessary intermediate results rather than relying on automatic frame dependency detection.

*Tags: `aegp`, `compute-cache`, `layer-checkout`, `memory`, `sequence-data`, `smart-render`*

---

### What are the advantages of compute cache compared to arbdata for storing pointers to heap data?

Compute cache offers several advantages over arbdata: (1) it can be written/read from any thread (render or UI) with optional thread-safety using std::atomic, (2) it's designed per-frame rather than requiring a global context like sequence or arb data, (3) it provides functions to specify whether to wait if another thread is computing the same key. However, arbdata gets saved with the project while compute cache does not. Compute cache replaces sequence data now that MFR is introduced and sequence data can no longer be used.

*Tags: `arb-data`, `compute-cache`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### Can compute cache data be transferred to arbdata for project persistence?

Yes, data from the compute cache can be transferred to arbdata if needed so it gets saved in the project. Additionally, you can read the compute cache from the arb or sequence thread if required.

*Tags: `arb-data`, `compute-cache`, `params`, `sequence-data`*

---

### How do you handle dependencies on previous frames in smart render when you need data from multiple prior frames?

In smart render, if your compute needs the result of a previous frame (e.g., frame N-1) but also requires frame N-2, you cannot use the smart checkout layer. Instead, you must use the classical checkout/checkin parameter approach to access the frames you need.

*Tags: `layer-checkout`, `render-loop`, `sequence-data`, `smart-render`*

---

### How can you access the previous frame's data when processing frame n in an After Effects effect plugin?

You can check out frame n-1 at render time to access the input frame from the previous frame. For more complex cases where you need the output from your plugin's previous frame, you should use sequence data, which is specifically designed for simulation plugins that require frame-by-frame dependencies.

*Tags: `aegp`, `layer-checkout`, `render-loop`, `sequence-data`*

---

### What is the documentation for using sequence data in After Effects effect plugins?

Adobe's official documentation on Global Sequence Frame Data is available at https://ae-plugins.docsforadobe.dev/effect-details/global-sequence-frame-data.html#validating-sequence-data. This documentation provides guidance on sequence data validation and includes simulation plugins as a primary example use case.

*Tags: `aegp`, `documentation`, `reference`, `sequence-data`*

---

### How can you ensure sequence data stays synchronized across render threads when duplicating a plugin instance?

When duplicating a plugin and assigning a new unique ID to sequence data, the sequence data may become out of sync on render threads even though UpdateUI and UserChangeParams receive the correct values. Two potential solutions: (1) Check if the sequence is flattened in the project, as flattening can force render threads to get the correct sequence value. (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of relying on in_data->seq_data directly, as the latter is not updated correctly since CC2021/2022.

*Tags: `debugging`, `params`, `render-loop`, `sequence-data`, `threading`*

---

### How can you handle migration of sequence data when the plugin format changes over versions?

Add an extra hidden parameter that stores the version number when sequence data is saved. When loading, parse this version information and reset values accordingly. This approach becomes especially useful when you have complex migration logic to handle multiple format versions.

*Tags: `arb-data`, `params`, `sequence-data`, `versioning`*

---

### How can I access sequence_data in SmartRender when it appears as nullptr?

In After Effects v22 and later, you must use the PF_EffectSequenceDataSuite to read sequence data in SmartRender instead of directly accessing in_data->sequence_data. Use AEFX_SuiteScoper to obtain the suite, then call PF_GetConstSequenceData to retrieve a const handle. This handle can then be locked with PF_LOCK_HANDLE to access the sequence data structure. Note that this method provides read-only access.

```cpp
PF_ConstHandle const_seq;
AEFX_SuiteScoper<PF_EffectSequenceDataSuite1> seqdata_suite =
    AEFX_SuiteScoper<PF_EffectSequenceDataSuite1>(
        in_data,
        kPFEffectSequenceDataSuite,
        kPFEffectSequenceDataSuiteVersion1,
        out_data);
seqdata_suite->PF_GetConstSequenceData(in_data->effect_ref, &const_seq);
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `aegp`, `mfr`, `sequence-data`, `smartrender`*

---

### When should you lock a handle when reading sequence data in After Effects plugins?

When calling GetConstSequenceData(), you should lock the handle before casting and unlock after use, even for read-only access. The lock syntax is: seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq); followed by unlock. This is particularly important in multi-frame rendering (MFR) scenarios with multiple plugin applications, as failure to lock can cause nullptr returns and threading-related crashes. The Adobe documentation does not clearly document this requirement, but it is necessary to prevent data corruption across threads.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
// use seqP
PF_UNLOCK_HANDLE(const_seq);
```

*Tags: `aegp`, `debugging`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### What UI approach do some AE plugins use to improve interactivity with parameters?

Some plugins like Gifgun and Datamosh use an external application window (often built with C++ and ImGui) that allows users to adjust parameters interactively in a fullscreen UI, then pre-render and send results back to After Effects. This avoids the need for re-rendering on every parameter change within the comp, which would be required if using a custom UI directly in After Effects like Optical Flares does.

*Tags: `open-source`, `sequence-data`, `ui`, `workflow`*

---

### How can a C++ plugin update a text layer every frame during rendering without locking the project?

This is a fundamental limitation in After Effects: calling AEGP_SetText from the render thread causes the project to lock. Several workarounds exist: (1) Write data to sequence data or compute cache from the render thread, then read and apply it from the UI thread using the aegp_idle hook, which is called multiple times per second; (2) Use your own text rendering engine in the render thread with an external library; (3) Write to disk and have a ScriptUI panel with a timer read it back (though this is unreliable). However, none of these work reliably in MediaEncoder or headless rendering without a GUI. The fundamental issue is that modifying a text layer from render could create infinite loops, which is why AE locks the project.

*Tags: `aegp`, `compute-cache`, `render-loop`, `sequence-data`, `threading`, `ui`*

---

### What is a good reference for understanding sequence data in After Effects plugin development?

Tobias Fleischer (reduxFX) has created comprehensive documentation about sequence data that has been very helpful for developers learning this concept. His guide clarifies how sequence data works in After Effects plugins.

*Tags: `aegp`, `documentation`, `reference`, `sequence-data`*

---

### How can I checkout masks and paths from a different layer using the After Effects SDK?

You can fetch any mask using AEGP_MaskSuite6, then read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape parameter, you need to traverse the layer's dynamic streams. However, be aware that mask and shape vertex data is delivered unmodified—masks won't include the "expand" parameter modification, and shapes won't show rounding if applied by the user. Additionally, some shapes are parametric (like star shapes) and have no shape parameter to read.

*Tags: `aegp`, `layer-checkout`, `masks`, `sdk`, `sequence-data`*

---

### What AEGP functions can be used to reference and read arbitrary data from other effect instances?

Use AEGP_GetLayerEffectByIndex to get an effect reference from a layer using AEGP_LayerH and the effect index. Then use AEGP_GetNewStreamValue to read arb values from that effect. For consistent effect identification, store an ID in sequence data and retrieve it using AEGP_EffectCallGeneric.

*Tags: `aegp`, `arb-data`, `sequence-data`*

---

### How can an effect instance identify itself given an AEGP_EffectRefH handle?

Only an effect instance has access to its own sequence data handle when AE passes it during calls, and AE may move that memory between calls. For identifying specific instances, experts recommend using a mechanism where each effect instance stores a unique identifier (like a 64-bit ID generated at instantiation). Since direct inter-instance communication via AEGP_EffectCallGeneric() is problematic for same-type effects, alternative approaches include: (1) using a separate AEGP plugin to mediate calls, or (2) for non-persistent identification, changing a hidden parameter's name to encode the identifier and reading it via the stream suite.

*Tags: `aegp`, `arb-data`, `params`, `sequence-data`*

---

### How can I set a parameter value during the paramSetup function?

ParamSetup is called once per session per effect type, not once per instance, so all instances would get the same value. Instead, use sequence data setup (PF_Cmd_SEQUENCE_RESETUP), which is triggered once per instance when created. However, you cannot directly set parameters during sequence setup because the call doesn't have a specific instance associated with it and the params array contains junk data. The correct approach is to set a default value during the PF_ADD_FLOAT_SLIDER call in paramSetup, or set a flag in sequence data indicating a new instance, then check that flag during idle_hook or UPDATE_PARAMS_UI to make the change there.

*Tags: `aegp`, `params`, `sequence-data`*

---

### Can I modify parameters during PF_Cmd_SEQUENCE_RESETUP?

No, you cannot modify parameters during sequence setup calls. Sequence setup typically occurs without a specific instance context, meaning the passed params array contains invalid data and attempting to acquire an AEGP_EffectRef will crash. The workaround is to set a flag in the new sequence data marking it as a brand new instance, then check this flag during idle_hook or UPDATE_PARAMS_UI to apply parameter changes there, remembering to clear the flag afterward.

*Tags: `aegp`, `params`, `sequence-data`*

---

### How can sequence data be safely shared and synchronized between PF_Cmd_EVENT and SmartPreRender calls?

Sequence data is intentionally separate between the render and UI threads by design, allowing render threads to execute asynchronously without data changes from other threads. To share data between them, use global data (shared across threads but not instance-specific) or store render instance data with an identifier such as comp item ID + layer ID + effect index on layer, then have the UI thread look up the correct data using a mutex. Note that arb/hidden params can only be written from the UI thread; render threads cannot modify project data.

*Tags: `aegp`, `arb-data`, `render-loop`, `sequence-data`, `threading`*

---

### Which identifiers (comp item ID, layer ID, or effect index) remain stable when layers are reordered in After Effects?

Comp item ID doesn't change during a session but may change between sessions or when projects are imported. Layer ID doesn't change when layers are reordered, but can change when layers are copied, cut/pasted, or duplicated to another comp, and may be re-used after deletion. Layer IDs are only unique within a single comp. Effect index changes when the effect moves in the effect stack but not when the layer is reordered; multiple instances of the same effect on one layer can have the same index.

*Tags: `aegp`, `arb-data`, `sequence-data`*

---

### How can I access decoded video frames in a specific order for face detection in an After Effects plugin?

You can request any project item's image at any time using AEGP_RenderAndCheckoutFrame and AEGP_RenderSuite4. However, After Effects renders frames on-demand and in random order by design. For sequential processing requirements, consider two approaches: (1) fast sequential pre-processing upfront at the cost of user experience, or (2) slower random-access processing for simplicity. Examples like AE's camera tracker and Lockdown perform sequential background processing while keeping the UI responsive. You may also need to use AEIO_InqNextFrameTime() to determine frame order and build a custom data structure to track the sequence.

*Tags: `aegp`, `caching`, `render-loop`, `sequence-data`, `smart-render`, `ui`*

---

### Why is PF_Cmd_SEQUENCE_FLATTEN sent when an effect is first applied, and why are SEQUENCE_RESETUP and SEQUENCE_SETDOWN called multiple times on different threads?

PF_Cmd_SEQUENCE_FLATTEN is sent when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set during global setup. Flattening allows plugins to serialize complex data structures (containing pointers or other non-copyable elements) into a single copyable piece of memory. Multiple SEQUENCE_RESETUP calls occur because AE's rendering architecture separates UI and rendering threads, each maintaining their own copy of sequence data. The PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA flag allows returning a flattened copy without losing the original, reducing some resetup calls on the UI thread. The PF_OutFlag2_MUTABLE_RENDER_SEQUENCE_DATA_SLOWER flag accommodates plugins requiring separate sequence data per rendering thread. Multiple SEQUENCE_SETDOWN calls correspond to cleanup on different threads.

*Tags: `aegp`, `mfr`, `sequence-data`, `threading`*

---

### How can I store per-effect instance data and ensure unique instance IDs when effects are copied, duplicated, or loaded from disk?

There is no built-in mechanism for per-instance unique IDs. A practical workaround is to store an instance ID in sequence data and use global data to track instances. On SEQUENCE_RESETUP, flag the sequence data as 'requires checking the id', then scan the project (or layer) for conflicting IDs. If duplicates are found, deduce which is the original instance (by tracking where it was last seen) and assign a new ID to the copy. This approach requires careful state management across copy/paste, duplication, and file loading operations. Note that during SEQUENCE_RESETUP, you may receive either flattened or unflattened data, so tag your data structure to track its state.

*Tags: `aegp`, `arb-data`, `sequence-data`, `threading`*

---

### How can I detect when an effect is removed from a layer or when a layer is deleted in an After Effects plugin?

There is no direct API for detecting effect removal, but you can use two approaches: (1) Scan the project on an idle hook to catalog instances of your effect, and if an effect that was present is missing on the next scan, it was removed (note: this is not immediate and happens on the next idle hook call after deletion). (2) Monitor cut and delete operations using command hooks, though this is complex and unreliable because not all deletion methods trigger menu commands (e.g., redo operations just set project state without triggering delete commands). The PF_Cmd_SEQUENCE_SETDOWN command is called when the project closes or memory is purged, not when the effect is removed by the user. First, you need to define what 'removal' means for your use case (e.g., does it include undo states?) before choosing the best approach.

```cpp
case PF_Cmd_SEQUENCE_SETDOWN:
err = SequenceSetdown(in_data,out_data);
err = SequenceSetdown2(in_data, out_data);
break;
```

*Tags: `aegp`, `debugging`, `sdk`, `sequence-data`*

---

### Why does an After Effects effect plugin crash when saving a project after upgrading to the latest SDK with MFR support?

The crash on save is likely related to sequence data flattening. When upgrading to newer SDKs, if you set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag during global setup but don't implement the PF_Cmd_SEQUENCE_FLATTEN selector, After Effects will crash when trying to save the project. The SEQUENCE_FLATTEN command is sent when saving or duplicating sequences and requires you to flatten sequence data containing pointers or handles so it can be written to disk. Additionally, verify you're allocating new handles when modifying instance sequence data rather than changing data in the same handle. Basic troubleshooting includes cleaning build intermediates, rebuilding from scratch, testing sample SDK projects, and progressively commenting out plugin code to isolate the problem.

*Tags: `build`, `debugging`, `mfr`, `sdk`, `sequence-data`*

---

### What is the purpose of the PF_Cmd_SEQUENCE_FLATTEN selector in After Effects plugins?

PF_Cmd_SEQUENCE_FLATTEN is sent by After Effects when saving or duplicating sequences. It requires the plugin to flatten sequence data containing pointers or handles so the data can be correctly written to disk and saved with the project file. The flattened data must be properly byte-ordered for file storage. When this command is received, the plugin should free unflat data and set out_data->sequence_data to point to the new flattened data. As of SDK 6.0, if an effect's sequence data has been flattened, the effect may be deleted without receiving PF_Cmd_SEQUENCE_SETDOWN, and After Effects will dispose of the flat sequence data.

*Tags: `arb-data`, `deployment`, `sdk`, `sequence-data`*

---

### Can I save sequence data during SequenceSetdown?

No. SequenceSetdown is AE's mechanism for telling your plugin that data needs to be destructed and freed. The data saved with the project is the last handle used or flattened (depending on whether you have set the flattening flag). SequenceSetdown is not the appropriate place to save data, as the logic is that the data may not just be a simple handle to be freed, but rather a structure with pointers that need to be separately destructed and freed.

*Tags: `aegp`, `memory`, `sdk`, `sequence-data`*

---

### How do I properly flatten sequence data so it persists when reopening a project?

To flatten sequence data, set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag during global setup. This will cause your plugin to receive PF_Cmd_SEQUENCE_FLATTEN calls where you can perform the flattening. Do not attempt to flatten data in SequenceSetdown, as that callback is only for destructing and freeing data, not for saving it.

*Tags: `aegp`, `params`, `sdk`, `sequence-data`*

---

### How can I detect when an After Effects project is being saved or opened in a plugin?

There is no reliable way to detect in advance that a project is about to save. However, you can use sequence data to handle save events. Set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag on your effect instance, and After Effects will call your effect with a handle to store data during save operations, copy/paste events, and other occasions when sequence data is persisted. You can also check during idle time if the project dirty status has changed to deduce that a save occurred, but this is reactive rather than predictive.

*Tags: `aegp`, `sdk`, `sequence-data`*

---

### What is the issue with storing effect identification data in sequence data across render calls?

Sequence data stored during effect initialization may not persist consistently across different command calls like PF_Cmd_Render. Using AEGP_InstalledEffectKey instead provides a more reliable way to uniquely identify effect instances across the plugin lifecycle without relying on sequence data storage.

*Tags: `aegp`, `params`, `sequence-data`*

---

### How should you properly store and reuse a frame snapshot across multiple renders in an AE plugin?

You can create a temporary EffectWorld and store it in sequence_data, then use it as the output for your effect across multiple frames. Alternatively, you can access the pixel data directly via effect_worldP->data (the base address of the world's buffer), save it to a file using a library like tinypng, and then import it back as needed. You should avoid copying data directly onto a layer's source buffer because AE may refresh that buffer without the plugin's knowledge, and the buffer may have already been cached elsewhere in the pipeline.

```cpp
effect_worldP->data  // base address of the world's buffer for direct pixel access
```

*Tags: `aegp`, `caching`, `memory`, `render-loop`, `sequence-data`*

---

### Why isn't sequence_data updated when modified in UpdateParameterUI and passed to SequenceResetup?

The issue is that UpdateParameterUI is not the correct place to modify sequence_data with FORCE_RERENDER. According to Adobe documentation, FORCE_RERENDER works during PF_Cmd_USER_CHANGED_PARAM and also in CLICK and DRAG events (if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented). Modifications to sequence_data should be made in PF_Cmd_USER_CHANGED_PARAM instead of UpdateParameterUI to ensure the data is properly synchronized between UI and render threads.

*Tags: `aegp`, `params`, `sequence-data`, `threading`, `ui`*

---

### How do you trigger PF_Cmd_UPDATE_PARAMS_UI to dynamically show/hide parameters when user input changes?

To trigger PF_Cmd_UPDATE_PARAMS_UI, you need to set two flags: (1) PF_OutFlag_SEND_UPDATE_PARAMS_UI in global setup, and (2) PF_ParamFlag_SUPERVISE on the parameter you wish to supervise. Without PF_OutFlag_SEND_UPDATE_PARAMS_UI set during global setup, the UPDATE_PARAMS_UI command will not be triggered even if sequence_data changes. Additionally, use PF_UpdateParamUI and the Stream/DynamicStream suites to modify parameter visibility at runtime, and set PF_OutFlag_REFRESH_UI in out_data->out_flags to refresh the UI.

```cpp
// In global setup:
out_data->out_flags = out_data->out_flags | PF_OutFlag_SEND_UPDATE_PARAMS_UI;

// In param setup:
def.flags = PF_ParamFlag_SUPERVISE;

// In PF_Cmd_USER_CHANGED_PARAM:
out_data->out_flags = out_data->out_flags | PF_OutFlag_FORCE_RERENDER | PF_OutFlag_REFRESH_UI;
```

*Tags: `aegp`, `params`, `sequence-data`, `ui`*

---

### How do you properly handle undo/redo with sequence data and arbitrary data in After Effects plugins?

Sequence data does not participate in undo/redo, which makes it problematic as a base for calculations that depend on parameter changes. The recommended approach is to store a state identifier in both arbitrary data (arb data) and sequence data. When using sequence data, compare the state identifier in the arb to the one in the sequence data—if they mismatch, regenerate the sequence data and store the corresponding state identifier. Sequence data is best used only when other storage methods don't meet needs, such as for data that should survive a reset button, data that shouldn't trigger undo entries, or temporary data passed between modules.

*Tags: `arb-data`, `debugging`, `params`, `sequence-data`*

---

### What are the appropriate use cases for sequence data in After Effects plugin development?

Sequence data has limited but specific use cases: (1) flagging new instances during unique sequence setup calls, (2) storing data that survives the reset button (unlike arb data), (3) storing data that should not undo/redo or trigger re-renders, and (4) maintaining session data without bloating the project file. It should not be used as a base for parameter-dependent calculations or processed image data, as there is no reliable way to tell when param values have changed and different frames may be processed simultaneously.

*Tags: `arb-data`, `params`, `sequence-data`*

---

### Is there a reference document explaining After Effects sequence data structure and behavior?

The Redux FX documentation provides comprehensive information about sequence data in After Effects plugins, including structure and calling conventions: https://reduxfx.com/ae_seqdata

*Tags: `documentation`, `reference`, `sequence-data`*

---

### How can I detect when my effect plugin is first applied to a layer in After Effects?

Detect a new effect application by setting a flag in sequence data during the SEQUENCE_SETUP call, which is the only call unique to a new application. At that time, you cannot get a layer handle because AE hasn't associated the instance yet. Wait for the UPDATE_PARAMS_UI call (after the flag is set) to get the layer handle. Then iterate through effects on the layer and use AEGP_GetInstalledKeyFromLayerEffect() to compare InstallKeys with your effect's InstallKey to identify duplicates.

*Tags: `aegp`, `params`, `plugin`, `sequence-data`*

---

### How can I create a plugin that adds image layers and regenerates them when properties change?

This is possible by combining an AEGP panel plugin with an effect plugin. The AEGP panel (implemented similar to the SDK's 'Panelator' sample) creates and manages layers with an inspector window for user controls. The effect plugin handles rendering and stores configuration data using either hidden parameters (which can be read/written by the AEGP window) or sequence data (a memory chunk that stores arbitrary data without UI representation, undo stack entries, or reset behavior). The effect plugin can regenerate images whenever properties change, with the decisions made in the panel interface driving the effect's rendering behavior.

*Tags: `aegp`, `params`, `reference`, `sequence-data`, `ui`*

---

### How does sequence data get synchronized between the UI thread and render thread in Smart Rendering?

Sequence data synchronization between UI and render threads happens at specific times only: when a new instance is created and during specific UI events. Synchronization occurs when PF_Cmd_USER_CHANGED_PARAM is triggered, and in CLICK and DRAG events if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented. The mechanism involves PF_Cmd_GET_FLATTENED_SEQUENCE_DATA and PF_Cmd_SEQUENCE_RESETUP for flattening and unflattening respectively. However, PreRendering and SmartRendering do not automatically trigger this flattening. For reliable data persistence, Adobe recommends using invisible arb parameters which sync on every change, or implementing a message-passing mechanism through global data structures accessible to both threads.

*Tags: `arb-data`, `render-loop`, `sequence-data`, `smart-render`, `threading`*

---

### What is the best way to persist custom objects from the UI to the rendering thread in After Effects plugins?

The recommended approach is to use invisible arb parameters, which create undo entries and provide immediate invalidation of cached frames on changes. Alternatively, if arb parameters don't fit your design, you can devise a message-passing mechanism by placing instructions in a global data structure common to both threads, with the render thread checking for messages. Direct use of sequence data flattening outside of USER_CHANGED_PARAM or UI click/drag events is not reliable and should be avoided, as it can lead to sync issues with SmartRendering and PreRendering.

*Tags: `arb-data`, `params`, `sequence-data`, `smart-render`, `threading`*

---

### Why does changing a button parameter name cause a heap corruption crash when the project closes?

The issue occurs because modifying button parameter names using strcpy on param_union.button_d.u.namesptr outside of PF_Cmd_UPDATE_PARAMS_UI causes heap corruption that manifests during sequence data setdown. The solution is to only modify button text within the PF_Cmd_UPDATE_PARAMS_UI command selector. If modification is needed elsewhere, use strncpy with the string length (without the null terminator +1) to avoid corrupting the buffer, though this may cause garbled text display. Alternatively, consider using AEGP_ExecuteScript() with JavaScript to change button text, which may avoid the issue entirely.

```cpp
// CORRECT METHOD - only call within PF_Cmd_UPDATE_PARAMS_UI
PF_ParamDef param = *params[ID];
strcpy((char*)param.u.button_d.u.namesptr, newLabel);
AEGP_SuiteHandler handler(in_data->pica_basicP);
ERR(handler.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref, ID, &param));

// WORKAROUND - outside UPDATE_PARAMS_UI, use strncpy without null terminator
strncpy((char*)param_union.button_d.u.namesptr, label.c_str(), label.length());
```

*Tags: `aegp`, `debugging`, `memory`, `params`, `sequence-data`, `ui`*

---

### Why does AEGP_ExecuteScript fail with a modal dialog error when called in PF_Cmd_SEQUENCE_RESETUP?

You cannot run AEGP_ExecuteScript while any After Effects dialog is open, including the project loading progress window that appears during sequence setup. The error 'Unable to execute script at line 0. After Effects error: Can not run a script while a modal dialog is waiting for response' indicates a modal dialog is blocking script execution. You need to engineer your process so the script runs at a different time, such as in PF_Cmd_UPDATE_PARAMS_UI instead, where no blocking dialogs are present.

*Tags: `aegp`, `debugging`, `scripting`, `sequence-data`*

---

### Can AEGP_RegisterIdleHook be used in an effect plugin to sync sequence data and UI?

AEGP_RegisterIdleHook is a general call that is not instance-specific, even when placed in an effect plugin. It executes once globally rather than once per effect instance. To access sequence data from individual effect instances, you must use EffectCallGeneric() to find and call each instance separately, allowing each instance to perform its own sequence data comparison. This is the only way to access sequence data from an idle hook in an effect plugin.

*Tags: `aegp`, `effect-plugin`, `params`, `sequence-data`*

---

### How can I access data from another plugin in After Effects?

Getting parameter data is relatively straightforward using both the C and JavaScript APIs, and the 'project dumper' sample project demonstrates this. However, there are two significant limitations: (1) accessing arbitrary parameter data (custom type params) is difficult—while you can retrieve the data chunk, deciphering its structure is challenging without knowing the format; (2) accessing sequence data (data stored internally by the plugin rather than in params) is only possible if the plugin provides specific custom infrastructure to expose it, otherwise only the plugin itself can access it.

*Tags: `arb-data`, `params`, `reference`, `scripting`, `sequence-data`*

---

### How can I pass data from a PopupDialog function to the Render function in an After Effects plugin?

There are two recommended approaches: (1) Store the message in global_data along with an effect identifier (comp itemID, layerID, and effect index), so the render thread can check if its matching identifier has a message waiting. (2) Write the data to an arbitrary parameter using SetStreamValue(), which gets synchronized immediately and reliably between threads. The arb param approach also allows undo/redo functionality for the user.

*Tags: `aegp`, `arb-data`, `sequence-data`, `threading`*

---

### Is there an open-source example of using sequence_data and popup dialogs in an After Effects plugin?

The AE_tl_math plugin by crazylafo on GitHub demonstrates sequence data handling in an After Effects plugin. Available at: https://github.com/crazylafo/AE_tl_math (specifically the tl_math.cpp file). This example shows how to flatten sequences and work with data communication in plugins.

*Tags: `open-source`, `reference`, `sequence-data`, `ui`*

---

### How can you track which version of a plugin was used to save an After Effects project?

There is no dedicated API in the After Effects SDK to retrieve the plugin version that saved a project. The recommended approach is to store the version number in the sequence data of your effect. This allows you to detect version changes when reopening projects and handle updates accordingly.

*Tags: `aegp`, `params`, `sequence-data`, `versioning`*

---

### How do you prevent sequence data from being lost during the flatten call when saving projects?

In After Effects versions before CC2017, you need to purge unwanted sequence data during the flatten call and reconstruct it during resetup. Starting with CC2017 and later, a new API call allows you to keep the original sequence data while delivering a flattened copy, preventing data loss. However, this new call is currently only implemented for UI event calls and not for copy, duplicate, or save operations. Eventually it will replace the older flatten call entirely.

*Tags: `aegp`, `macos`, `params`, `sequence-data`, `windows`*

---

### Why is sequence_data different between PF_Cmd_RENDER and PF_Cmd_USER_CHANGED_PARAM in After Effects CC 2015 and later?

In After Effects CC 2015 and later, the render thread and UI thread have separate sequence_data handles that only synchronize at specific occasions. This is a design change from earlier versions like CS5.5. To force synchronization, set PF_OutFlag_FORCE_RERENDER during UserChangedParam, which forces a flatten call on the UI thread and an unflatten on the render thread, syncing the render thread's sequence data with the UI data.

*Tags: `params`, `render-loop`, `sequence-data`, `threading`, `ui`*

---

### How can I force a rerender after PF_Cmd_SEQUENCE_RESETUP or PF_Cmd_SEQUENCE_SETUP when effect state changes?

Use GuidMixInPtr to incorporate custom state flags (such as a watermarked or license status flag) into the state that After Effects uses to determine whether a cached frame is valid. When you push the relevant flag into the mix, After Effects will automatically trigger a re-render when that state changes. This feature is documented in the CC2015 SDK and later versions.

*Tags: `caching`, `compute-cache`, `render-loop`, `sdk`, `sequence-data`*

---

### How can I access output width and height values in a UI event handler?

The out_data.width and out_data.height values are only available during rendering and will be 0 in UI event handlers. To access these values in a DragHandle or DoClick event, you need to cache the desired values in sequence_data during the render call, then retrieve them during the event call. Note that in After Effects 13.5+, sequence_data cannot be changed from the render thread—only changes from the UI thread persist to the project and render thread.

*Tags: `params`, `render-loop`, `sequence-data`, `threading`, `ui`*

---

### How can I share data between the render thread and UI event handlers?

Since After Effects 13.5, sequence_data modified in the render thread cannot be accessed in UI threads. Two workarounds are: (1) add a pointer to sequence_data that is shared between threads (requiring logic to distinguish live pointers from saved invalid ones), or (2) store data in the global_data handle which is shared between threads and instances (requiring logic to identify which data belongs to which instance). The recommended approach is caching values in sequence_data during render and reading them back in event handlers from the UI thread.

*Tags: `arb-data`, `render-loop`, `sequence-data`, `threading`, `ui`*

---

### How should you handle sequence data structure changes when adding new parameters?

When modifying sequence data structure, if you extend or modify the structure, old project files loaded with the new plugin will attempt to match the old sequence data to the new structure, potentially causing conflicts. Best practice is to only add new parameters with new IDs at the end of the current parameter set without changing the order of existing parameters. Do not replace existing parameter definitions, only append new ones. This ensures backward compatibility with projects saved using older plugin versions.

*Tags: `arb-data`, `deployment`, `params`, `sequence-data`*

---

### How do you get frame data like PF_world from sequence_data in After Effects plugins?

sequence_data is a PF_Handle that you should cast to whatever structure you're using. Since you know the structure you're working with, you can cast the handle directly to access the corresponding memory region. The key distinction is between smartFX and non-smartFX effects: non-smartFX effects get input images through params[0]->u.ld, while both types can use PF_CHECKOUT_PARAM() to specify source times. smartFX effects use checkout_layer_pixels() instead. For any source time checkout, you must set the PF_OutFlag_WIDE_TIME_INPUT flag. For accessing pixels outside layer params, use AEGP_GetReceiptWorld().

*Tags: `layer-checkout`, `params`, `sequence-data`, `smartfx`*

---

### How can I ensure a function runs after parameter values are updated in an After Effects plugin?

Instead of directly modifying parameter values and setting change flags, use AEGP_SetStreamValue() for instantaneous changes. There is no function guaranteed to be called after values change, but UPDATE_PARAMS_UI might get called and a render call is likely. Alternatively, you can set a flag in the sequence data to notify yourself of post-change operations that need to take place during UPDATE_PARAMS_UI or render calls.

```cpp
params[POINT]->u.td.x_value = FLOAT2FIX(pointX);
params[POINT]->u.td.y_value = FLOAT2FIX(pointY);
params[POINT]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `aegp`, `params`, `render-loop`, `sequence-data`*

---

### How do you get input layer data with effects applied during an arbitrary draw event in After Effects?

Use extra->cb->checkout_layer_pixels() instead of PF_CHECKOUT_PARAM(), as the latter only fetches the layer's source without previous effects. However, checkout_layer_pixels() requires special suites available only during smart render calls and cannot be called during UI events. A better approach is to cache histogram data in sequence data during render calls and use PF_GetCurrentState and PF_HasParamChanged during draw events to detect parameter changes, then force re-render with PF_OutFlag_FORCE_RERENDER.

```cpp
typedef struct {
  bool didRender;
  /* histogram data */
} Histogram;
typedef struct {
  Histogram* histograms;
} my_sequence_data, *my_sequence_dataP, **my_sequence_dataH;

/* During render, cache at current time */
sequence->histograms[in_data->current_time / in_data->time_step]

/* During draw event, detect changes */
if (PF_HasParamChanged(in_data, extra)) {
  event_extraP->evt_out_flags |= PF_OutFlag_FORCE_RERENDER;
}
```

*Tags: `aegp`, `caching`, `sequence-data`, `smart-render`, `ui`*

---

### What is the recommended way to store large numbers of parameters that need to be saved in a project file?

For storing many parameters (e.g., 1000+ values) that need to be persisted in the project file, use sequence data or arbitrary data rather than individual built-in parameter types. These are automatically saved and recalled with the project. Ensure you correctly handle copying and flattening/deflattening of values in the appropriate functions, especially if using pointers. The SDK examples demonstrate the correct implementation.

*Tags: `arb-data`, `deployment`, `params`, `sequence-data`*

---

### How do I extract path data from a stream in After Effects using the SDK?

To get path data, first obtain the stream reference for the path. Then use StreamSuite2()->AEGP_GetNewStreamValue() to retrieve the stream value at a specific time, casting it to PF_PathOutlinePtr. Once you have the path outline pointer, use PathDataSuite1() functions like PF_PathPrepareSegLength() and PF_PathGetSegLength() to query segment data. The NULL parameter for PF_ProgPtr can be passed as NULL in this context.

```cpp
suites.StreamSuite2()->AEGP_GetNewStreamValue(NULL, pathStreamH, AEGP_LTimeMode_CompTime, &time, TRUE, &value);
suites.PathDataSuite1()->PF_PathPrepareSegLength(NULL, (PF_PathOutlinePtr)value.val.mask, index, freq, &pathSegment);
suites.PathDataSuite1()->PF_PathGetSegLength(NULL, (PF_PathOutlinePtr)value.val.mask, index, &pathSegment, &segLength);
```

*Tags: `aegp`, `reference`, `sdk`, `sequence-data`*

---

### Why can't I access parameter input layer pixel data in the DoClick function?

The incoming image data is only available during the render call, not in DoClick. However, there are several workarounds: (1) Store input/output pixels in sequence data during render for later access, though cached frames won't update; (2) Use AEGP_GetReceiptWorld() to fetch pixels of any video item at any time, though rendering composites can be slow; (3) In CS4 and earlier, use the UI drawing mechanism during the draw event; (4) Best practice: store click location and flags in sequence data during DoClick, then trigger a re-render to process pixels during the render call.

*Tags: `aegp`, `doclick`, `params`, `pixel`, `render-loop`, `sequence-data`*

---

### How should I store fixed pixel coordinates in an After Effects plugin preset?

You can store x,y coordinates using several methods: (1) Point params - simple approach, can be hidden from UI; (2) Arbitrary type params - allows custom data structures, supports undo, triggers re-render; (3) Sequence data - single instance per plugin, handles data changes manually, doesn't trigger re-render. Data is stored with the project file. Refer to the ColorGrid sample for arbitrary params and the Supervisor sample for sequence_data.

*Tags: `arb-data`, `params`, `sequence-data`, `ui`*

---

### When duplicating a layer with custom effects, which effect receives the PF_Cmd_SEQUENCE_FLATTEN command—the old or new effect?

The FLATTEN command is sent only to the old effect. It tells the effect to collect and prepare all its data for storing. Both the old and new effects receive the UNFLATTEN command afterward. To distinguish between old and new effects during UNFLATTEN, store identification data during FLATTEN (such as effect index, layer ID, and composition item ID), then check these identifiers during UNFLATTEN or the first updateParamsUI call to determine which effect is which.

*Tags: `aegp`, `params`, `sequence-data`*

---

### How do I distinguish between the original and duplicated effect when both have identical sequence data after duplication?

Store identifying information about the effect during the FLATTEN command, such as its effect index, layer ID, and composition item ID. During the UNFLATTEN command (or preferably at the first updateParamsUI call), compare the current effect's properties against the stored identification data. If the properties match, it is the old effect; if any differ, it is the new duplicated effect.

*Tags: `aegp`, `params`, `sequence-data`*

---

### How can I prevent a slider parameter from resetting to its default value when the Effect Reset button is clicked?

You cannot tell a parameter to ignore the reset operation directly. However, you can store the parameter value in sequence data and restore it when the SEQUENCE_RESETUP callback is triggered. Alternatively, if the parameter is used as a unique identifier, you can skip storing it as a parameter altogether and maintain it only in sequence data, retrieving it via AEGP_EffectCallGeneric() when needed. You could also try setting a different default value during the param_setup call, though param_setup is typically called only once when the effect is first applied.

*Tags: `aegp`, `params`, `sdk`, `sequence-data`*

---

### Can I use sequence_data to store a circular buffer for frame caching in an After Effects temporal filter effect?

Yes, you can use sequence_data to store a circular buffer. Allocate memory using PF_NEW_HANDLE during PF_Cmd_SEQUENCE_SETUP (though you can allocate at any time). Check if in_data->sequence_data == NULL to determine if memory has been allocated. Be aware of important considerations: (1) You cannot reliably know if cached frames are still valid after user edits, (2) Frames may be acquired out of order if the user scrubs randomly—track timestamps to handle this, (3) You won't know when scrubbing ends and sequential rendering begins, (4) If you set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING, the entire buffer is saved in the project file. Modern After Effects caches frames internally, so re-requesting past frames often retrieves cached data rather than forcing re-renders. The sequence_data lifecycle includes: PF_Cmd_SEQUENCE_SETUP (new instance), PF_Cmd_SEQUENCE_RESETUP (project load/paste/save), PF_Cmd_SEQUENCE_FLATTEN (save/copy preparation), and PF_Cmd_SEQUENCE_SETDOWN (cleanup).

```cpp
if (in_data->sequence_data == NULL) {
  // Allocate memory
  in_data->sequence_data = PF_NEW_HANDLE(buffer_size);
} else {
  // Memory already allocated, reuse it
}
```

*Tags: `aegp`, `caching`, `memory`, `render-loop`, `sequence-data`*

---
