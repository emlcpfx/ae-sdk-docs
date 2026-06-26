# Frame Caching

> 98 Q&As · source: AE plugin dev community Discord

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

### How do you check if an input frame has changed to avoid unnecessary re-processing?

Use PF_GetCurrentState to detect if an input has changed. In SmartFX, the PreRender/SmartRender pipeline handles this more naturally. You can also use GuidMixInPtr to mix in any data that should trigger re-rendering - if parameters or other state change, mix that into the GUID and AE will know to call SmartRender again. Note: if the layer is continuously rasterized, transforms are applied before your effect receives the input.

*Tags: `caching`, `guid-mixin`, `input-change`, `optimization`, `pf-get-current-state`*

---

### Why does memory usage spike to 6+ GB when I add PF_OutFlag_NON_PARAM_VARY to my plugin?

This is normal AE behavior - it's hoarding cached frames. PF_OutFlag_NON_PARAM_VARY tells AE the output varies independently of parameters, so AE caches each unique frame. Verify by purging AE's memory (Edit > Purge > All Memory) - if RAM consumption drops to expected levels, everything is working correctly. AE will release cached memory when it's needed for new frame renders. PF_OutFlag_WIDE_TIME_INPUT is only needed if you use a layer selector and sample it at different times. Implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup.

*Tags: `caching`, `frame-setdown`, `frame-setup`, `memory`, `non-param-vary`, `purge`*

---

### How do you cache frames in an After Effects effect plugin and use the cached result as input for processing the next frame?

This requires implementing frame caching within your plugin's render loop. You need to store the output of frame N, then retrieve it to use as input for frame N+1. This typically involves maintaining a persistent buffer or cache structure across multiple render calls. The exact implementation depends on whether you're using SmartFX or legacy plugins, but generally involves allocating memory to store intermediate results and managing that cache through the plugin's lifecycle to ensure the previous frame's output is available when processing subsequent frames.

*Tags: `caching`, `memory`, `output-rect`, `render-loop`, `smartfx`*

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

### How do plugins like Trapcode Particular cache and calculate multiple frames over time rather than just the last frame?

Plugins that need to cache multiple frames typically use their own internal state management system separate from the standard Sequence Data approach. While Sequence Data can store the last frame, plugins handling fluid simulations or particle systems that need to jump to arbitrary times in a composition use custom caching mechanisms. These plugins maintain their own frame cache, calculate values for multiple frames, and store them in memory structures that can be accessed when jumping to different times in the composition. The exact implementation depends on the plugin's architecture, but it generally involves managing a larger state buffer that holds multiple frames of simulation data rather than relying solely on the sequential frame-to-frame Sequence Data mechanism.

*Tags: `caching`, `memory`, `render-loop`, `sequence-data`*

---

### Can I save frames to arbitrary file formats and read them back in when caching Effect_World in sequence data?

Yes, you can save frames into any file format you want and read them back in. This approach allows you to work around sequence data storage limitations when caching multiple frames, similar to using a .dat file example.

*Tags: `caching`, `memory`, `sequence-data`*

---

### What flags or methods are used to create a caching effect where rendering waits for frame caching to complete before updating?

This is a technique used in higher-end plugins like Particular where moving the Current Time Indicator (CTI) causes the plugin to wait for frame caching to complete before updating the render. This involves using smart render techniques and potentially compute cache mechanisms to defer rendering until cache state is ready.

*Tags: `caching`, `compute-cache`, `render-loop`, `smart-render`*

---

### What causes an infinite progress bar at the end of a caching effect?

An infinite progress bar look can be triggered by an incoherent PF_Progress value. Give it impossible values like 200/100 and it should display this infinite progress behavior.

*Tags: `caching`, `debugging`, `ui`*

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

### Does changing image size or color depth affect the frame number where a cache-related crash occurs?

The suggestion is to test whether changing image size or color depth of the project changes the crash frame number, as this could indicate whether RAM access is being handled correctly with time-varying data, or if there's a conflict between sequence data writing and MFR.

*Tags: `caching`, `debugging`, `memory`, `mfr`, `sequence-data`*

---

### What causes PF_Interrupt_CANCEL to occur in the middle of a render in After Effects?

PF_Interrupt_CANCEL can occur during rendering when certain conditions are met, such as when CAPS LOCK is off and the composition has not been previously rendered to the RAM cache. The issue appears to be related to some interaction with keyboard state or cache status, as the interrupt does not occur when CAPS LOCK is on or when the composition has been fully previewed in the RAM cache beforehand.

*Tags: `caching`, `debugging`, `memory`, `render-loop`*

---

### How do you invalidate cached frames to force After Effects to re-render them instead of using cached versions?

The user is asking about the mechanism used by effects like Warp Stabilizer and 3D Camera Tracker to invalidate frames. No answer was provided in this conversation.

*Tags: `caching`, `render-loop`, `smart-render`*

---

### What is a recommended approach for tracking memory allocations and deallocations in C++ plugins?

Wrap memory operations in custom optionally-logging wrappers to get a definitive record of what happened. Additionally, have all C++ classes descend from a base class like ObjectBase that keeps track of live instances of each type, allowing you to report current counts for allocations versus frees.

*Tags: `caching`, `debugging`, `memory`*

---

### How should the Compute Cache API be implemented with callback functions?

Use the AEGP_ComputeCacheSuite1 to register a compute class with four callback functions: MyGenerateKeyFunc (to identify the effect ref, layer id, effect position, current time, and time scale id), MyComputeFunc (to perform the computation), MyApproxSizeValueFunc (to return the approximate size of the cached value), and MyDeleteComputeValueFunc (to free the cached value). Register these callbacks using AEGP_ClassRegister with a unique class identifier string.

```cpp
#include "AE_ComputeCacheSuite.h"
A_Err MyGenerateKeyFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeKeyP out_keyP) {
    LOG("Generate Key function");
    return A_Err_NONE;
}
A_Err MyComputeFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeValueRefconP *out_valuePP) {
    LOG("Compute Functions");
    return A_Err_NONE;
}
size_t MyApproxSizeValueFunc(AEGP_CCComputeValueRefconP valueP) {
    LOG("Obtain size function");
    return 0;
}
void MyDeleteComputeValueFunc(AEGP_CCComputeValueRefconP valueP) {
    LOG("Delete function");
}
static PF_Err GlobalSetup(PF_InData* in_data, PF_OutData* out_data, PF_ParamDef* params[], PF_LayerDef* output) {
    AEFX_SuiteScoper<AEGP_ComputeCacheSuite1> compute_suite = AEFX_SuiteScoper<AEGP_ComputeCacheSuite1>(in_data, kAEGPComputeCacheSuite, kAEGPComputeCacheSuiteVersion1);
    AEGP_ComputeCacheCallbacks callbacks = { MyGenerateKeyFunc, MyComputeFunc, MyApproxSizeValueFunc, MyDeleteComputeValueFunc };
    compute_suite->AEGP_ClassRegister("com.mycompany.effect.myComputeCacheClass", &callbacks);
}
```

*Tags: `aegp`, `caching`, `compute-cache`, `memory`*

---

### How can you force After Effects to de-cache frames when changing a plugin's globaldata variable?

Setting PF_OutFlag_FORCE_RERENDER alone may not work reliably when mixing in a globaldata variable via extra->cb->GuidMixInPtr. One workaround is to "kick" AE by triggering an internal frame cache check, such as changing the composition background color and then changing it back. However, this is not ideal; using GuidMixInPtr for forcing rerenders can work properly if implemented correctly.

*Tags: `caching`, `compute-cache`, `params`*

---

### How should a plugin handle processing that requires the previous frame's data (frame n-1 for frame n)?

You have two main approaches: (1) Use sequence data, which is designed for simulation plugins that need to access previous frames, or (2) Check out the input frame n-1 at render time if you only need the pre-effect input frame rather than your plugin's output from frame n-1. If you need your plugin's own output from frame n-1, you'll need to store intermediate results and manage sequential execution, as After Effects has no built-in way to force sequential frame processing.

*Tags: `caching`, `layer-checkout`, `render-loop`, `sequence-data`*

---

### How can you efficiently copy a OpenCV Mat to an After Effects layer?

Use the Iterate Suite and copy entire rows at a time with memcpy() instead of copying pixel-by-pixel. Respect the layer->rowBytes value. Advance both the OpenCV buffer by its own rowbyte and the layer buffer by its own rowbyte to maintain proper memory alignment. Ensure both buffers have the same pixel format (ARGB) and are uchar type for safe memory alignment.

```cpp
static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        char* layerRow = (char*)layer->data + (y * layer->rowbytes);
        char* matRow = mat.ptr(y);
        memcpy(layerRow, matRow, layer->width * sizeof(PF_Pixel));
    }
}
```

*Tags: `caching`, `memory`, `opencv`, `pixel-data`*

---

### Does smart render provide benefits for effects that require the full image each time they are processed?

Smart render optimizes by only requesting pixels required for specific processing. For effects that require the full image each time, smart render's primary optimization of selective pixel fetching provides limited benefit since the effect must process the entire image anyway. However, smart render can still provide benefits in other scenarios such as avoiding redundant computation when the effect is cached or when only part of the output is needed by downstream effects.

*Tags: `caching`, `output-rect`, `render-loop`, `smart-render`*

---

### What are the practical limits for working with a composition containing thousands of layers in After Effects?

After Effects struggles significantly even at 512 layers. As a user reports, working with over 500 layers becomes a nightmare in terms of performance and usability. The software has inherent limitations that make working with very large numbers of layers (such as 10,000) impractical, even if they are instances of only a small number of source files.

*Tags: `caching`, `memory`, `performance`, `render-loop`*

---

### What performance improvement was achieved after Adobe fixed the arb handling issue?

A x7 speed up was observed after Adobe fixed something in arb handling, which also fixed copy/paste effect times issues that were previously causing performance problems.

*Tags: `arb-data`, `caching`, `performance`*

---

### What is the purpose of rowbytes in After Effects layer data?

rowbytes exists as an optimization so that layers can be cropped without allocating extra memory. You can crop a layer by simply updating width and height and making the data pointer reference the new top-left pixel, leaving rowbytes untouched while keeping a reference to the old data for proper cleanup. After Effects does this often, such as when dragging a layer partially off the composition.

*Tags: `caching`, `layer-checkout`, `memory`, `output-rect`*

---

### Does rowbytes change when a layer is moved partially off-comp and then cache is purged?

When you load a layer and move it partially off-comp, rowbytes will be larger than the minimum required. However, after purging cache, rowbytes will likely return to 4*width (or 4*4*width for 32-bit data), indicating the cache optimization is recalculated.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `memory`*

---

### Is it possible to create a multi-layer 2D global illumination plugin where a renderer effect discovers and uses material properties from dummy effects on other layers?

This approach is theoretically possible with the After Effects SDK. You can use the AEGP suite to enumerate layers in a composition and checkout their properties. However, the key challenge is properly communicating dependencies to AE's internal dependency tracking system. You need to explicitly declare all layer and effect dependencies your renderer effect relies on, so that changes to materials or other layers trigger re-renders. This ensures the disk cache is used correctly—cached results are only used when all dependencies remain unchanged. The AEGP layer enumeration and property checkout APIs support this pattern, though it requires careful dependency management to avoid either missing updates or unnecessary re-renders.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `params`*

---

### How can a renderer effect communicate its dependencies on other layers and effects to AE's caching system?

Dependencies must be explicitly registered through the AEGP API when your renderer effect enumerates and checks out layers. When you call layer checkout functions to read pixel data, masks, shapes, and transforms from dependent layers, and query properties of effects on those layers, AE tracks these accesses as dependencies. To ensure proper cache invalidation, you should checkout all relevant layer properties at render time—if any of those checked-out properties change, AE will invalidate the cached result and trigger a re-render. This allows the disk cache to function correctly by only using cached data when all dependencies are identical.

*Tags: `aegp`, `caching`, `compute-cache`, `dependency`, `layer-checkout`*

---

### Is it possible to create a multi-layer effect that discovers material properties from per-layer dummy effects and performs global illumination rendering while properly communicating dependencies to After Effects' caching system?

Tobias Fleischer shared that he attempted a plugin-per-layer approach years ago but abandoned it due to render order troubles. Instead, he recommended using a single plugin with multiple layer parameters that pull in pixels from different layers, apply material/lighting options, and composite everything within that single plugin. This approach worked well despite AE's rigid parameter interface being somewhat clunky. The viability of the frankensteined multi-layer approach with proper dependency tracking depends on whether the SDK allows explicit communication of implicit dependencies to AE's caching system, though existing solutions tend to consolidate the logic into a single effect rather than relying on scattered per-layer effects.

*Tags: `caching`, `layer-checkout`, `output-rect`, `params`, `smart-render`*

---

### Is it possible to make a multi-layer 2D global illumination effect where a renderer effect discovers material metadata effects on other layers and communicates these dependencies to AE's caching system?

It is theoretically possible using AEGP_RenderAndCheckoutLayerFrame to enumerate layers and pass their GUIDs via GuidMixInPtr() to register dependencies. However, a more practical approach is to use hidden layer parameters (up to 999) with a script button that assigns layers from the composition to these parameters, allowing checkout_layer() to work efficiently with SmartPreRender. You can also mix in layer order information via GuidMixInPtr() to handle dynamic layer changes without manual updates.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `smart-render`*

---

### How should you handle caching when scraping data from dummy effects on other layers?

Call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects, and AE will cache everything fine.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`*

---

### How do 3D Camera Tracker and Warp Stabilizer output progress over frames with live updates without cache purging?

According to an After Effects developer, these built-in tools use internal APIs that are not available to regular plugin developers. The Warp Stabilizer effect 'cheats' and does not operate within the constraints of a normal AE Effect API plug-in. Adobe's internal team has access to secret suites that allow this functionality, which is not exposed to third-party developers. A feature request was made in 2023 to expose this capability to plugin developers, but it remains unavailable.

*Tags: `aegp`, `caching`, `debugging`, `mfr`, `smart-render`*

---

### What are the main challenges when updating GPU plugins for MFR compatibility?

GPU plugins face significant challenges with MFR that CPU plugins don't encounter. Issues include crashes that occur only after long rendering periods (making them hard to debug), GPU caching complications, and sequence data handling. OpenGL-based plugins particularly struggle with Mac-specific issues. James Whiffin reported that even using Adobe's GLator sample, Mac OpenGL compatibility remains problematic. The recommendation is to consider maintaining completely separate codebases: one for Metal on Mac and one for Windows, as OpenGL performance on Mac can be severely degraded compared to Windows.

*Tags: `caching`, `gpu`, `macos`, `metal`, `mfr`, `opengl`, `sequence-data`, `windows`*

---

### Is MFR compatibility straightforward for simple plugins that don't use sequence data or caching?

Yes, for simple plugins without sequence data or complex caching requirements, MFR compatibility is straightforward. Gabgren reported that after the initial panic about the MFR announcement, updating simple plugins required little more than a rebuild. Complex plugins using sequence data or advanced features require more significant refactoring, but basic plugins can be updated quickly.

*Tags: `caching`, `mfr`, `sequence-data`*

---

### How do you cache frames in an After Effects effect plugin and use the cached result as input for the next frame?

Nate asked about implementing frame caching in an effect plugin where frame 1 is processed and cached, then that cached result becomes the input for frame 2's processing, creating a cumulative effect. This is a common pattern for effects that depend on previous frame state.

*Tags: `caching`, `memory`, `mfr`, `render-loop`*

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

### Why does a displacement map effect sometimes use the original frame instead of the previously cached frame?

Nate is experiencing inconsistent behavior with a displacement mapping effect that should feed the distorted output back as input for the next frame. The effect sometimes works correctly (distorting and caching each frame to use as the next input) but randomly reverts to using the original frame instead of the cached frame. This suggests a potential issue with the frame caching or smart render logic that determines which input frame gets used for displacement. The problem appears to be intermittent rather than systematic.

*Tags: `caching`, `debugging`, `render-loop`, `smart-render`*

---

### How can you debug why a cached frame is not being used even though it should be?

To diagnose caching issues, first determine whether the frame is always cached but not always used, or if it's not being cached consistently. If it's always cached but not always used, the problem lies in the conditional logic that checks cache validity. Start by disabling conditions in the if statement that verify cache availability (such as hash checks) to isolate which condition is returning false and preventing cache usage.

*Tags: `caching`, `debugging`, `memory`, `smart-render`*

---

### How do plugins like Trapcode Particular cache multiple frames over time for features like fluid simulation?

Plugins that need to cache multiple frames typically use the Sequence Data API to store frame information persistently. While a basic setup might store only the last frame in Sequence Data, plugins like Trapcode Particular that perform simulations (such as fluid dynamics) across many frames likely store cumulative or checkpoint data in the Sequence Data structure. This allows them to jump to any point in the composition and recalculate or retrieve cached simulation values. The key is leveraging Sequence Data's ability to persist data across multiple frames rather than just the immediate previous frame.

*Tags: `arb-data`, `caching`, `memory`, `sequence-data`*

---

### How can you cache frame data that's too large to fit in RAM?

You can save binary data directly to disk for each frame (e.g., 0000.dat, 0001.dat, etc.), similar to how Houdini handles large computation results. After Effects may have a cache API for this, but if not, you'll need to implement your own disk-based caching system to store frame data when it exceeds available memory.

*Tags: `caching`, `memory`, `mfr`, `render-loop`*

---

### Can I save cached frames to a custom file format and read them back in a plugin?

Yes, you can save frames to any file format you want and read them back in. This is a common approach when caching large amounts of frame data that won't fit in sequence data. You can use custom binary formats (like .dat files) or other formats suitable for your needs. The plugin can write frames to disk during processing and read them back as needed, avoiding the limitations of storing arbitrarily large numbers of frames directly in sequence data.

*Tags: `caching`, `compute-cache`, `mfr`, `sequence-data`*

---

### How can you persist sequence data without consuming memory?

You can dump sequence data to a file (encoded or not) and read it back as needed. This approach allows you to save a sequence of data without loading it all into memory at once.

*Tags: `caching`, `file-io`, `memory`, `sequence-data`*

---

### What flags or methods are used in high-end plugins like Particular to cache frames and delay render updates until caching is complete?

The user is asking about caching mechanisms used in professional plugins like Particular that intelligently manage frame rendering by waiting for frame cache completion before updating the CTI (Current Time Indicator). This typically involves smart render and compute cache APIs provided by After Effects to optimize performance when scrubbing the timeline.

*Tags: `caching`, `compute-cache`, `render-loop`, `smart-render`*

---

### How does caching behavior work by default in the render function, and how can you update progress indication?

Caching calculations taking place in the render function will have default caching behavior where the output won't be updated until the plugin has populated it. For updating the loading bar specifically, use PF_Progress.

*Tags: `caching`, `mfr`, `render-loop`*

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

### How do you add time values when using the compute cache API in After Effects?

When adding time values to the compute cache, you need to use AEGP (After Effects General Plugin) APIs to properly interact with the time-based caching system.

*Tags: `aegp`, `caching`, `compute-cache`*

---

### How do you invalidate cached frames in After Effects so they will be re-rendered instead of using the cache?

This question was asked about invalidating rendered frames to force After Effects to re-render them, similar to how built-in effects like Warp Stabilizer and 3D Camera Tracker work. However, no answer was provided in the conversation.

*Tags: `aegp`, `caching`, `render-loop`, `smart-render`*

---

### How do you use the Compute Cache API in an After Effects plugin?

The Compute Cache API requires implementing four callback functions: a key generation function (MyGenerateKeyFunc) that identifies the effect ref, layer id, effect position, current time and time scale; a compute function (MyComputeFunc) that performs the actual computation; an approximate size function (MyApproxSizeValueFunc) that returns the memory size of cached values; and a delete function (MyDeleteComputeValueFunc) that frees memory. Register these callbacks in GlobalSetup using AEGP_ComputeCacheSuite1 with AEGP_ClassRegister. The key function should include effect reference, layer id, effect position, current time and time scale to enable per-frame effect caching.

```cpp
#include "AE_ComputeCacheSuite.h"
A_Err MyGenerateKeyFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeKeyP out_keyP) {
    LOG("Generate Key function");
    return A_Err_NONE;
}
A_Err MyComputeFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeValueRefconP *out_valuePP) {
    LOG("Compute Functions");
    return A_Err_NONE;
}
size_t MyApproxSizeValueFunc(AEGP_CCComputeValueRefconP valueP) {
    LOG("Obtain size function");
    return 0;
}
void MyDeleteComputeValueFunc(AEGP_CCComputeValueRefconP valueP) {
    LOG("Delete function");
}
static PF_Err GlobalSetup(PF_InData* in_data, PF_OutData* out_data, PF_ParamDef* params[], PF_LayerDef* output) {
    AEFX_SuiteScoper<AEGP_ComputeCacheSuite1> compute_suite = AEFX_SuiteScoper<AEGP_ComputeCacheSuite1>(in_data, kAEGPComputeCacheSuite, kAEGPComputeCacheSuiteVersion1);
    AEGP_ComputeCacheCallbacks callbacks = { MyGenerateKeyFunc, MyComputeFunc, MyApproxSizeValueFunc, MyDeleteComputeValueFunc };
    compute_suite->AEGP_ClassRegister("com.mycompany.effect.myComputeCacheClass", &callbacks);
}
```

*Tags: `aegp`, `caching`, `compute-cache`, `memory`*

---

### What memory management issues can occur when using the Compute Cache API on macOS during export?

When caching large data structures (e.g., 100MB memory handles) per frame during export on macOS, memory handles may not be destroyed quickly enough in the delete callback function, leading to RAM accumulation and crashes during long exports. Proper memory management in the MyDeleteComputeValueFunc callback is critical to avoid this issue.

*Tags: `caching`, `compute-cache`, `macos`, `memory`*

---

### How do you use GuidMixInPtr to force re-rendering in After Effects plugins?

Use the GuidMixInPtr callback to mix in a GUID dependency that forces re-renders. First, set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag. Then in PreRender, check that in_data->version.major >= PF_AE130_PLUG_IN_VERSION and verify extraP, extraP->cb, and extraP->cb->GuidMixInPtr are non-null and not the sentinel value 0xabababababababab. Call GuidMixInPtr with a value that changes (e.g., based on time or other conditions) to trigger re-renders. For timestamps, use gettimeofday on non-Windows platforms or GetTickCount64 on Windows, then multiply by 100 and add randomness to ensure uniqueness.

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data, PF_PreRenderExtra* extraP) {
    if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION && extraP && extraP->cb && extraP->cb->GuidMixInPtr && (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab) {
        int r = time(0) + in_data->current_time;
        extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(r), reinterpret_cast<void *>(&r));
    }
}

#ifndef WIN32
struct timeval tp;
gettimeofday(&tp, NULL);
long long dummy = (long long) tp.tv_sec * 1000L + tp.tv_usec / 1000;
#else
long long dummy = GetTickCount64();
#endif
dummy = dummy * 100 + rand() % 100;
extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(dummy), reinterpret_cast<void *>(&dummy));
```

*Tags: `caching`, `cross-platform`, `guid-dependencies`, `prerender`, `render-loop`*

---

### How can you efficiently copy pixel data from an OpenCV Mat to an After Effects layer?

Use the Iterate Suite and copy entire rows at a time with memcpy() instead of copying pixel-by-pixel. Respect the layer's rowBytes value when advancing pointers. Ensure both buffers have matching pixel format (ARGB for AE, BGRA for OpenCV by default) and are uchar type for proper memory alignment. Advance both the OpenCV buffer and layer buffer by their respective rowByte values during the copy operation.

```cpp
static PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
    return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        cv::Vec4b* matRow = mat.ptr<cv::Vec4b>(y);
        PF_Pixel* aeRow = (PF_Pixel*)((char*)layer->data + (y * layer->rowbytes));
        memcpy(aeRow, matRow, layer->width * sizeof(PF_Pixel));
    }
}
```

*Tags: `caching`, `memory`, `output-rect`, `render-loop`*

---

### What is the OpenCV Mat step parameter and how does it help with custom pixel data?

OpenCV's Mat class supports a step parameter (also called rowbytes) that allows you to create a Mat from user-allocated pixel data while specifying the stride/row pitch. This is documented in the OpenCV Mat constructor. When working with After Effects layer data, pass layerDef->rowbytes as the step parameter to correctly handle the memory layout of the pixel buffer, enabling efficient operations like mixChannels and memcpy.

```cpp
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Tags: `caching`, `layer-checkout`, `memory`, `opencv`*

---

### How can you create an OpenCV Mat that points to existing pixel data with custom row stride?

Use the cv::Mat constructor with the step parameter to specify custom row bytes. Example: cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes); This allows OpenCV to work directly with After Effects pixel buffers without copying data.

```cpp
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Tags: `caching`, `gpu`, `memory`, `opencv`*

---

### What is the purpose of rowbytes in After Effects layer data, and how does it enable memory optimization?

Rowbytes exists as an optimization so that layers can be cropped without allocating extra memory. You can crop a layer by updating width and height and making the data pointer reference the new top-left pixel, while leaving rowbytes unchanged and maintaining a reference to the original data for proper deallocation. After Effects uses this optimization frequently, such as when dragging a layer partially off the composition. Rowbytes is calculated as 4*width for 32-bit data, but may be larger if a layer is loaded and moved partially off-comp; however, purging cache typically resets it back to 4*width.

*Tags: `caching`, `memory`, `optimization`, `output-rect`*

---

### How can a plugin effect communicate implicit dependencies on other layers and effects to After Effects' caching system?

When building multi-layer effects that discover and depend on other layers' properties at runtime, you need to explicitly communicate these dependencies to AE's render dependency tracking system. This involves checking out the layers you depend on and notifying AE of those checkouts so the cache knows to invalidate when those layers change. The renderer effect should enumerate dependent layers, check out their pixels/masks/shapes/transforms, and use the appropriate AEGP calls to register these dependencies so AE's smart render and disk cache systems understand the effect needs to re-render when those layers are modified.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `memory`, `smart-render`*

---

### Is it feasible to build a 2D global illumination plugin using material property metadata effects on individual layers with a separate renderer effect?

Yes, this architecture is theoretically viable with the AE SDK. The pattern involves creating dummy effects on individual layers that only expose material parameters (emissive, diffuse, glass properties, etc.) without rendering output, while a separate renderer effect on an adjustment layer or solid discovers and interprets these metadata effects at render time. The key challenge is properly communicating cross-layer dependencies to AE's caching system so changes to materials trigger re-renders of the expensive global illumination computation while still benefiting from disk caching when unrelated elements change.

*Tags: `aegp`, `caching`, `layer-checkout`, `params`, `rendering`, `smart-render`*

---

### Is it possible to create a multi-layer global illumination plugin that discovers material metadata effects on other layers and properly communicates dependencies to After Effects' caching system?

This is a complex architectural question posed by fad regarding whether a 'renderer' effect can enumerate other layers, discover per-layer material definition effects, check out layer pixels/masks/transforms, and have those dependencies properly tracked by AE's dependency tracking and disk cache system. While Tobias Fleischer's experience suggests multi-layer effects are viable, the specific mechanism for communicating hidden dependencies (non-explicit UI parameter dependencies) to AE's caching system to trigger rerenders when materials change remains an open design challenge requiring careful SDK integration and likely custom dependency management.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `memory`, `render-loop`, `smartfx`*

---

### Can I use checkout_layer() during PF_Cmd_SMART_PRE_RENDER to access arbitrary composition layers not exposed as effect parameters?

According to maxon developer Alex Bizeau, you can only grab layer params directly during SMART_PRE_RENDER. However, you can use AEGP_RenderAndCheckoutLayerFrame with any layer IDs, and both sync and async versions exist. For multiple layers, a workaround is to use hidden layer parameters (up to 999) assigned via a button script that updates the comp, so checkout_layer() can access them. You may also handle dynamic layer order/removal by looping through the comp in SmartPreRender and mixing layer order into GuidMixInPtr() for dependency tracking.

*Tags: `aegp`, `caching`, `layer-checkout`, `params`, `smart-render`*

---

### How can I make a multi-layer effect that depends on material properties defined by effects on other layers while maintaining smart render efficiency?

The most robust approach is to use hidden layer parameters (you can have up to 999) and a button parameter that triggers a script to assign layers from the composition to those parameters one by one. Alternatively, loop through the comp in SmartPreRender and mix layer order into GuidMixInPtr() for dependency tracking. This allows the renderer effect to check out all layers efficiently while maintaining AE's caching and smart render system. You only need to click an 'Update Comp' button when you add layers; layer order changes or removals can be handled automatically by the SmartPreRender logic.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `params`, `smart-render`*

---

### Can I pass a layer receipt GUID to GuidMixInPtr() to register layer dependencies without rendering pixels during SmartPreRender?

You can retrieve a GUID from an AEGP_FrameReceiptH using AEGP_GetReceiptGuid and pass it to extra->cb->GuidMixInPtr(). However, AEGP_RenderAndCheckoutLayerFrame is the expensive call that renders the entire layer, so calling it during SmartPreRender defeats the performance benefit. The suggested approach is to use layer parameters instead, which allow AE to track dependencies without rendering during the SmartPreRender phase.

*Tags: `aegp`, `caching`, `compute-cache`, `smart-render`*

---

### How can you efficiently cache data when scraping parameters from dummy effects on multiple layers?

When implementing a control scheme using layer parameters on the main effect with dummy effects on other layers, call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects. This ensures After Effects caches everything properly and maintains good performance even when reading all layers one by one.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`, `params`*

---

### How can you synchronize layer data changes in an After Effects plugin using AEGP?

According to Alex Bizeau from maxon, you need to periodically check the project timestamp for changes, then gather your layers and check children for potential updates. This approach is computationally expensive but is noted as the proper synchronization alternative available with AEGP.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`*

---

### How do 3D Camera Tracker and Warp Stabilizer output progress and live update without purging cache?

According to an After Effects developer, these built-in effects use internal APIs that are not available to plugin developers. The effects "cheat" and do not operate within the constraints of a normal AE Effect API plug-in. Developers have attempted workarounds involving refresh kicks, but these have proven unreliable across macOS and Windows. A feature request was made in 2023 to expose this capability to third-party developers, but it remains unavailable through the public API.

*Tags: `aegp`, `caching`, `cross-platform`, `debugging`, `render-loop`, `smart-render`*

---

### How can you iterate through pixel data when working with rowbytes in After Effects or Premiere?

You can iterate through pixel data yourself using rowbytes. Be careful with negative values in Premiere, as they can occur and need special handling.

*Tags: `caching`, `memory`, `mfr`, `premiere`*

---

### Why is there no performance improvement when calling suite functions repeatedly inside iterate_generic callbacks?

Repeatedly acquiring suites inside iteration callbacks creates significant overhead that negates parallel execution benefits. Suite acquisition should be done once from the main thread before the iteration, and then pointers to the callbacks should be passed to the iteration function. Pre-acquiring suites can improve performance by orders of magnitude compared to acquiring them for every iteration.

*Tags: `caching`, `optimization`, `performance`, `smartfx`, `threading`*

---

### How can I setup an After Effects plugin to asynchronously receive video frames from an external pipe?

You can setup a separate C++ thread (independent of the AE SDK) to handle reading/writing data to a pre-allocated memory buffer. Use a mutex to safely access this memory from both the async pipe thread and AE's render calls. To trigger the async pipe thread to process data promptly, use AEGP_CauseIdleRoutinesToBeCalled(). However, reconciling AE's random frame access model with sequential async pipe operations is challenging—you may need to either stall the render until required frames are available or cache frames to a file and fetch them on demand. For fetching source frames without blocking the UI, use AEGP_CheckoutOrRender_ItemFrame_AsyncManager or AEGP_CheckoutOrRender_LayerFrame_AsyncManager.

*Tags: `aegp`, `async`, `caching`, `memory`, `threading`*

---

### How can I access decoded video frames in a specific order for face detection in an After Effects plugin?

You can request any project item's image at any time using AEGP_RenderAndCheckoutFrame and AEGP_RenderSuite4. However, After Effects renders frames on-demand and in random order by design. For sequential processing requirements, consider two approaches: (1) fast sequential pre-processing upfront at the cost of user experience, or (2) slower random-access processing for simplicity. Examples like AE's camera tracker and Lockdown perform sequential background processing while keeping the UI responsive. You may also need to use AEIO_InqNextFrameTime() to determine frame order and build a custom data structure to track the sequence.

*Tags: `aegp`, `caching`, `render-loop`, `sequence-data`, `smart-render`, `ui`*

---

### What is the primary rendering philosophy of After Effects for plugin developers?

After Effects uses an on-demand, random-access rendering scheme rather than sequential frame processing. This design prioritizes user experience by rendering only requested frames in any order. Plugins that need sequential processing must work against this architecture, but can implement background processing with progress indication (like the camera tracker) to maintain UI responsiveness while performing pre-processing tasks.

*Tags: `caching`, `render-loop`, `smart-render`, `ui`*

---

### How should I store time-dependent curve data for a custom parameter and ensure the UI updates as the user draws?

Use an arbitrary parameter (PF_ADD_ARBITRARY2) to store your data. The workflow is: user interacts with UI → data is stored in a LOCAL array → local array is serialized and stored in the arb parameter → AE automatically invalidates cached frames and re-renders. Before displaying the UI, read the value from the arb param, deserialize it into your local array, and draw accordingly. This also ensures undo/redo operations reflect correctly in the UI. You don't need to convert data to a string; serialize it into one continuous block of memory using whatever method works for your data type.

*Tags: `arb-data`, `caching`, `params`, `render-loop`, `ui`*

---

### Why does setting PF_OutFlag_NON_PARAM_VARY cause memory usage to increase dramatically?

Setting PF_OutFlag_NON_PARAM_VARY or PF_OutFlag_WIDE_TIME_INPUT causes After Effects to call FrameSetup>Render>FrameSetDown for each frame instead of once, which can appear to increase memory usage significantly. However, this is often normal behavior where After Effects is caching frames. The memory will be released when purging AE's memory cache or when the memory is needed for new frame renders. Ensure you implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup, preventing accumulation across multiple frame renders.

*Tags: `aegp`, `caching`, `memory`, `render-loop`*

---

### What is the most performant way to iterate through pixels when converting 8-bit to 16-bit data in After Effects plugins?

Use IterateGeneric to iterate through each line instead of each pixel, which provides much better performance compared to iterating pixel-by-pixel. Additionally, consider using PF_COPY for the actual pixel conversion operation, and refer to methods from the Transformer example that convert PF_Pixel to PF_Pixel16 if needed.

*Tags: `aegp`, `caching`, `memory`, `performance`, `sdk`*

---

### How should you properly store and reuse a frame snapshot across multiple renders in an AE plugin?

You can create a temporary EffectWorld and store it in sequence_data, then use it as the output for your effect across multiple frames. Alternatively, you can access the pixel data directly via effect_worldP->data (the base address of the world's buffer), save it to a file using a library like tinypng, and then import it back as needed. You should avoid copying data directly onto a layer's source buffer because AE may refresh that buffer without the plugin's knowledge, and the buffer may have already been cached elsewhere in the pipeline.

```cpp
effect_worldP->data  // base address of the world's buffer for direct pixel access
```

*Tags: `aegp`, `caching`, `memory`, `render-loop`, `sequence-data`*

---

### Why doesn't changing a plugin parameter invalidate cached frames in After Effects?

This is actually normal After Effects behavior. When a parameter changes, the composition render cache is invalidated and the displayed result is correct. However, After Effects intentionally does not release the RAM of previously cached frames—it keeps old results in memory in case the user undoes changes, until RAM is needed for something else. To verify this, purge After Effects' RAM and the memory usage will drop back to the original level, confirming the RAM accumulation is AE's design and not a memory leak.

*Tags: `caching`, `memory`, `params`, `render-loop`*

---

### Why does memory consumption increase drastically when playing an After Effects plugin and not return to normal after removing the effect layers?

The memory increase is likely not a leak but rather After Effects caching rendered frames. To verify this, purge RAM after playing or previewing the composition—if memory returns to the pre-play level after purging, it confirms the memory was being held by the cache, not leaked by the plugin. If a parameter change should invalidate cached frames but doesn't, you can force invalidation by changing the value of an invisible parameter.

*Tags: `caching`, `debugging`, `memory`, `render-loop`*

---

### How can you trigger cache purging from within a C++ plugin when a user changes a parameter?

You can use JavaScript execution via the C API to trigger the cache purge command. Execute `app.executeCommand(2370)` through the ExecuteScript function. However, this approach is strongly discouraged because purging RAM forces users to re-cache all other timelines or portions of the current timeline. Instead, ensure that parameter changes properly invalidate cached frames automatically, or use an invisible parameter change to force invalidation if needed.

```cpp
app.executeCommand(2370);
```

*Tags: `caching`, `memory`, `params`, `scripting`*

---

### Why does my C++ plugin show the wrong initial frame when loading, and how do I fix it?

The issue is likely caused by AE's dynamic resolution feature, which automatically reduces the composition's resolution to maintain interactive speeds. You can verify this by disabling it in the composition window settings (lightning icon). However, the real solution is to implement smartFX and report a GUID during SmartPreRender to tell AE when a frame differs from what it has cached. Additionally, you must account for the downsample factors passed in the in_data struct, which is common practice for AE plugin developers.

*Tags: `caching`, `debugging`, `render-loop`, `smartfx`*

---

### Why do I see artifacts and previously rendered shapes appearing in my plugin output when modifying parameters?

The issue stems from not properly accounting for AE's dynamic resolution feature and downsample factors. When drawing with cairo and copying to the AE buffer, you need to handle the downsample factors from the in_data struct. Additionally, ensure you're properly clearing the canvas at the beginning of each render pass and not relying on cached buffers. The enlargement of shapes during computation is typically a symptom of dynamic resolution being applied without proper downsample factor handling in your pixel copy operations.

```cpp
PF_FILL(NULL, NULL, output); // Clean the canvas at the beginning
cairoRefcon.data = cairo_image_surface_get_data(surface);
cairoRefcon.stride = cairo_image_surface_get_stride(surface);
cairoRefcon.output = *output;
ERR(suites.IterateSuite1()->AEGP_IterateGeneric(output->height, &cairoRefcon, cairoCopy8));
```

*Tags: `caching`, `debugging`, `memory`, `params`, `render-loop`*

---

### How can I load a file into a C++ plugin just once instead of every frame in the render function?

Use global_data or sequence_data to store the loaded dictionary outside the render function. global_data stores information common to all instances of an effect throughout an AE session and is not saved with the project, while sequence_data stores information specific to one applied instance of the effect and can be saved with the project. You can put any data you want in these memory handles. Refer to the "supervisor" sample project in the SDK for implementation details on how to use global_data.

*Tags: `aegp`, `caching`, `memory`, `reference`*

---

### How should an importer plugin interact with an effect plugin to share custom file data?

An importer plugin allows custom file types to be imported into After Effects and placed in compositions. There are two main scenarios for importer-to-effect interaction: (1) The importer parses the file on demand and populates an image buffer, which can then be read by the effect like any other layer source. After Effects can cache the result for faster access on subsequent frame renders. This approach expands usability for custom file types. (2) The effect uses the layer selector only to get the selected layer's ID, retrieves the source project item and file path from it, and then parses the file directly in the effect. This approach allows for data files without graphic interpretation and enables selective storage of data rather than loading all file data into RAM. Both scenarios are valid, and you can combine both approaches—parsing in the importer while also allowing the effect to fetch and parse the file path independently.

*Tags: `aegp`, `arb-data`, `caching`, `importer`, `layer-checkout`*

---

### Why does my After Effects plugin cause memory to balloon to 4-5 GB when changing parameters?

This can happen for two main reasons: (1) After Effects caching images as a user preference, which generally shouldn't be a concern, or (2) your plugin uses PF_NewWorld, which in some AE versions doesn't free memory even when PF_DisposeWorld is called. The memory is only freed when the user manually purges all caches through the menu. To fix this, consider allocating memory through the memory suite instead of PF_NewWorld, and release it after processing. Alternatively, you can point a PF_EffectWorld's 'data' parameter to arbitrarily allocated memory by filling in the other relevant data fields.

*Tags: `aegp`, `caching`, `debugging`, `memory`, `pf_newworld`*

---

### What happens if a plugin doesn't check for interrupt with PF_ABORT during rendering?

After Effects does not forcibly kill plugins in mid-call. Instead, AE tells the plugin it wants to abort and waits for the render call to return. The call should return either PF_ErrNONE or PF_Interrupt_CANCEL. If PF_ErrNONE is returned, AE might cache that result. If PF_Interrupt_CANCEL is returned, AE will discard the output buffer content as it has been reported as invalid.

*Tags: `aegp`, `caching`, `mfr`, `render-loop`*

---

### How can I force a rerender after PF_Cmd_SEQUENCE_RESETUP or PF_Cmd_SEQUENCE_SETUP when effect state changes?

Use GuidMixInPtr to incorporate custom state flags (such as a watermarked or license status flag) into the state that After Effects uses to determine whether a cached frame is valid. When you push the relevant flag into the mix, After Effects will automatically trigger a re-render when that state changes. This feature is documented in the CC2015 SDK and later versions.

*Tags: `caching`, `compute-cache`, `render-loop`, `sdk`, `sequence-data`*

---

### Why does my After Effects plugin display a white screen after parameter changes, requiring a purge to render correctly?

This is typically a buffer clearing issue. You need to fill the output buffer with empty pixels using PF_FillMatteSuite2()->fill(). Another common cause is accidentally writing data to the input buffer and then using it as an intermediate processing buffer, which causes After Effects to cache the tampered data. Always clear AEGP worlds before using them, as After Effects may reuse the same RAM block from a previous world call, which could contain junk data or previous frame data.

```cpp
PF_FillMatteSuite2()->fill()
```

*Tags: `aegp`, `caching`, `memory`, `output-rect`, `smart-render`*

---

### How can a plugin detect when a RAM preview starts in After Effects?

Use command_hook with command number 2285 to detect when a RAM preview begins. Command 2415 can be used to detect Play (spacebar) events. Note that PF_Cmd_RENDER may not be called for cached frames during RAM preview.

*Tags: `aegp`, `caching`, `debugging`, `render-loop`*

---

### How can a plugin clear the frame cache in After Effects?

Use AEGP_DoCommand() with command number 2372 to clear the frame cache. This allows subsequent playback to trigger PF_Cmd_RENDER for every frame instead of using cached frames.

*Tags: `aegp`, `caching`, `compute-cache`*

---

### Why does memory usage increase continuously when using AEGP_EffectCallGeneric to send mesh data between effects?

Memory leaks when using AEGP_EffectCallGeneric typically stem from improper ownership and disposal of allocated data structures. If you're allocating worlds or buffers to send data and not properly disposing of them after use, AE's memory footprint will continuously grow. Ensure that for any AEGP_WorldH you create with WorldSuite3()->AEGP_New(), you call WorldSuite3()->AEGP_Dispose() exactly once when done. If the issue persists even with empty data, verify that your sequence data management isn't holding references to disposed memory and that you're not allocating new structures on every frame without cleanup.

*Tags: `aegp`, `arb-data`, `caching`, `memory`*

---

### In what order does After Effects render frames when using the Render button?

After Effects does not necessarily render frames in sequential order when you hit the Render button. The application uses an internal algorithm to determine which frames should be rendered based on what is already cached in memory, which can result in non-sequential frame rendering order.

*Tags: `caching`, `render-loop`, `smart-render`*

---

### How can I check out only a portion of a frame at a different time instead of the entire frame?

You can use checkout_layer_pixels() which takes a request info structure as an argument that allows you to request a smaller rectangle. Alternatively, use the render suite to request the source item directly and set an ROI (region of interest) to get only the part you need. The render suite method doesn't hint AE in advance about your intentions, so AE won't cache items in advance—test both methods to see which is faster for your use case, as getting a potentially cached full frame might sometimes be faster than requesting a partial frame without caching.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`, `output-rect`, `render-loop`*

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

### How do you force After Effects to re-render when custom UI parameters change but the effect's cached result is stale?

Use PF_GetCurrentState and PF_HasParamChanged during a draw event to detect if the plug-in's parameter state has changed since the last draw call. If parameters have changed, set the evt_out_flags to include PF_OutFlag_FORCE_RERENDER to force a fresh render. Simply setting change_flags or PF_EO_HANDLED_EVENT alone is not sufficient to overcome AE's caching behavior.

*Tags: `caching`, `params`, `smart-render`, `ui`*

---

### Can you access the comp overlay buffer during a render call using Drawbot?

No, you cannot access the comp overlay buffer during a render call, regardless of whether you're using Drawbot. Instead, you must cache whatever you want to render during a "draw" event call, and then use that cache during the render call. Drawbot's data structures are opaque and don't offer tools for reading data back out. If you need to draw during render, consider using third-party drawing tools or OS-level drawing tools instead, such as OpenGL.

*Tags: `caching`, `drawbot`, `render-loop`, `ui`*

---

### What causes incomplete rendering when stacking multiple plugin instances and reloading a project?

The issue is likely caused by confused world buffer handling, especially with temporary buffers that might get overwritten by stacked effects. Carefully trace world creation to ensure input, output, and temporary buffers are not confused. A specific trap: temporary world buffers obtained from AE may actually point to previous frame output buffers in memory. Solution: clean buffer pixels before using temporary buffers, or ensure proper buffer allocation and usage patterns.

*Tags: `caching`, `debugging`, `memory`, `smart-render`*

---

### What are the differences between PF_CHECKOUT_PARAM and the checkout_layer/checkout_layer_pixels methods for accessing layer parameters in After Effects plugins?

Both methods are functionally correct but differ in important ways. The checkout_layer/checkout_layer_pixels method (used with PreRender and Render callbacks) is more resource-efficient as it allows smartFX to plan ahead and use RAM and buffers more efficiently for the entire rendering process. However, smartFX is not supported by all hosts, so if portability is important, this should be considered. A key difference: when checking out the same layer you're on, PF_CHECKOUT_PARAM gives a pre-effects image, while checkout_layer_pixels gives a post-effects image (including previous effects in the stack), even when sampling times other than the current render frame. For float images, the smartFX checkout_layer/checkout_layer_pixels combination is required.

```cpp
// Method 1: PF_CHECKOUT_PARAM
PF_ParamDef def;
AEFX_CLR_STRUCT(def);
ERR(PF_CHECKOUT_PARAM(in_data, layerID, in_data->current_time, in_data->time_step, in_data->time_scale, &(def)));
PF_EffectWorld myLayer = def.u.ld;
ERR2(PF_CHECKIN_PARAM(in_data, &(def)));
PF_COPY(&myLayer, output, NULL, NULL);

// Method 2: checkout_layer with PreRender callback
extraP->cb->checkout_layer(in_dataP->effect_ref, layerID, layerID, &req, in_dataP->current_time, in_dataP->time_step, in_dataP->time_scale, &ck_result);

// Method 2: checkout_layer_pixels with Render callback
PF_EffectWorld* myLayer;
ERR(extraP->cb->checkout_layer_pixels(in_data->effect_ref, layerID, &myLayer));
PF_COPY(myLayer, output, NULL, NULL);
extraP->cb->checkin_layer_pixels(in_data->effect_ref, layerID);
```

*Tags: `caching`, `layer-checkout`, `memory`, `params`, `smartfx`*

---

### Should you dispose of stream references acquired with AEGP_GetNewStreamRefForLayer, and when?

Yes, you must dispose of the stream reference using AEGP_DisposeStream to release the streamPH you acquired. Do not use AEGP_DeleteStream, which is only for deleting streams like paint strokes. Every getStream call must be balanced by a disposeStream call. It is recommended to dispose of the stream as soon as you're done with it, or at minimum before returning from the call that triggered the process.

*Tags: `aegp`, `caching`, `memory`*

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
