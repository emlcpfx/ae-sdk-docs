# Compute Cache

> 49 Q&As · source: AE plugin dev community Discord

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

### What is the AEGP memory leak issue on Mac during export with Compute Cache?

When using AEGP_MemorySuite to create memHandles during ComputeCache threads, the memory may not be properly freed during export on Mac (works fine on Windows). The used memory grows past the RAM limit. Using new/delete instead of AEGP memHandles avoids the leak. The issue is that memHandles allocated in one thread and freed in another may not actually release memory during the render thread. The AEGP tools report the memory as freed, but virtual memory keeps growing.

*Tags: `aegp-memory-suite`, `compute-cache`, `export`, `mac`, `memory-leak`, `threading`*

---

### What flags or methods are used to create a caching effect where rendering waits for frame caching to complete before updating?

This is a technique used in higher-end plugins like Particular where moving the Current Time Indicator (CTI) causes the plugin to wait for frame caching to complete before updating the render. This involves using smart render techniques and potentially compute cache mechanisms to defer rendering until cache state is ready.

*Tags: `caching`, `compute-cache`, `render-loop`, `smart-render`*

---

### Why does device-to-device buffer transfer in Vulkan have the same speed as host-to-device transfer when using external CUDA memory?

The transfer likely implicitly bounces off host memory or the CUDA buffer may actually be on host memory rather than device memory, even though the transfer is performed device-to-device. This can happen when Vulkan cannot reliably determine how to handle the caching of imported CUDA memory handles.

*Tags: `compute-cache`, `cuda`, `gpu`, `memory`, `vulkan`*

---

### Do you need to use AEGP for adding time values in the compute cache API?

The conversation indicates this is a question about the proper API approach for time value handling in the compute cache, but a definitive answer was not provided in the chat.

*Tags: `aegp`, `compute-cache`, `smartfx`*

---

### How can you access the previous frame result during the compute thread in After Effects?

The user asked this question but indicated that despite reading the documentation, a full example would be more helpful. No complete answer was provided in the conversation.

*Tags: `compute-cache`, `memory`, `render-loop`, `smartfx`*

---

### How can a plugin render frame N that depends on custom data from previous frames without checking out all 100 previous frames in memory during pre-render?

This is a known challenge with recursive operations in After Effects plugins. The issue is that smart render checks needed frames during pre-render, which forces checking out all intermediate frames. The checkout param solution doesn't account for previous effects. AEGP_CacheAndCheckoutFrame doesn't trigger the pre_render/smart render thread, so it doesn't solve the problem. A potential approach is to use layer checkout parameters during compute cache operations, but this requires careful design to avoid excessive memory usage and to ensure previous effects are properly evaluated.

*Tags: `aegp`, `compute-cache`, `layer-checkout`, `memory`, `smart-render`*

---

### How can frame N access custom data computed from the previous frame, accounting for previous effects in the chain?

The user explored using checkout Param during compute cache, but noted this approach doesn't account for previous effects. The solution involves retrieving data computed from the previous frame (same size as a frame) through the effect chain.

*Tags: `compute-cache`, `layer-checkout`, `render-loop`, `sequence-data`*

---

### Is there a limitation with AEGP_memsuite when allocating and freeing memory during the render thread?

The user observed that memory allocated using aegp_memHandle during the compute cache thread is not being freed when the freemem function is called, even though instruments show no leaks. This suggests there may be specific limitations or requirements for using AEGP_memsuite during the render thread that differ from other threads.

*Tags: `aegp`, `compute-cache`, `debugging`, `memory`, `render-loop`, `threading`*

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

### What causes memory issues when using the Compute Cache API on macOS during export?

On macOS, when caching large amounts of data (e.g., 100MB per frame) during export, the memory handles are not destroyed quickly enough, causing RAM to become overloaded and eventually crash on long exports. The issue is that the delete function may not be called frequently enough to free the cached memory handles during the export process.

*Tags: `compute-cache`, `deployment`, `macos`, `memory`*

---

### What are the advantages of compute cache compared to arbdata for storing pointers to heap objects that are flattened/unflattened?

Compute cache can be written and read from any thread (render or UI thread) using std::atomic for thread safety. Unlike arbdata, compute cache is not stored with the project. Compute cache is designed to get a value per frame, whereas arbdata requires a global context where you define arrays or vectors to store data for different frames. Compute cache replaced sequence data after MFR was introduced since sequence data can no longer be used.

*Tags: `arb-data`, `compute-cache`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### Can data from compute cache be transferred to arbdata to persist it in the project?

Yes, data from the compute cache can be transferred to arbdata if needed so it gets saved in the project. You can read the compute cache from arb or sequence thread if you want.

*Tags: `arb-data`, `compute-cache`, `sequence-data`*

---

### How can you implement a key generation function that requires AEGP_HashSuite1 when the function signature only provides AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP parameters?

This question was asked but not answered in the conversation.

*Tags: `aegp`, `compute-cache`, `debugging`*

---

### How do you properly access compute cache data and retrieve the basic suite pointer?

Create an access_cache_data pointer from optionsP, verify in_data is not null, then extract the SPBasicSuite pointer from in_data->pica_basicP. Always check that bsuite is valid before using it to avoid allocation errors.

```cpp
access_cache_data* accessCacheData = static_cast<access_cache_data*>(optionsP);
if ( !accessCacheData->in_data ) {return A_Err_ALLOC;}
SPBasicSuite* bsuite;
bsuite = accessCacheData->in_data->pica_basicP;
if (!bsuite){return A_Err_ALLOC; }
```

*Tags: `aegp`, `compute-cache`, `memory`*

---

### What structure should be used to pass in_data and out_data to compute cache functions?

Define a struct access_cache_data that contains PF_InData* and PF_OutData* pointers to pass the required suite information to the cache. Create a new object and copy only the needed parts, then delete it after computing to avoid crashes and maintain independence from the render thread.

```cpp
struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
}
```

*Tags: `aegp`, `compute-cache`, `memory`, `threading`*

---

### How can you force After Effects to de-cache frames when changing a plugin's globaldata variable?

Setting PF_OutFlag_FORCE_RERENDER alone may not work reliably when mixing in a globaldata variable via extra->cb->GuidMixInPtr. One workaround is to "kick" AE by triggering an internal frame cache check, such as changing the composition background color and then changing it back. However, this is not ideal; using GuidMixInPtr for forcing rerenders can work properly if implemented correctly.

*Tags: `caching`, `compute-cache`, `params`*

---

### Can compute cache data be accessed in globalsetup or paramssetup?

Compute cache must be defined during global setup, and the first access possible is during param setup. However, compute cache cannot be flattened. For persisting data in project files, sequence data or arb data should be used instead.

*Tags: `compute-cache`, `globalsetup`, `params`*

---

### Does rowbytes change when a layer is moved partially off-comp and then cache is purged?

When you load a layer and move it partially off-comp, rowbytes will be larger than the minimum required. However, after purging cache, rowbytes will likely return to 4*width (or 4*4*width for 32-bit data), indicating the cache optimization is recalculated.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `memory`*

---

### How can a C++ plugin update a text layer every frame after rendering completes?

This is not cleanly possible due to AE's project locking during render operations. Possible workarounds include: (1) writing data to a compute cache or sequence data and reading it from the UI thread via the aegp_idle hook, (2) using AEGP_SetText from the UI thread instead of the render thread, (3) rendering text yourself in the render thread using an external library, or (4) saving to disk and having a ScriptUI panel with a timer read the data. The aegp_idle hook approach may not work without a GUI or in MediaEncoder. Setting text from the render thread causes a locked project error.

*Tags: `aegp`, `compute-cache`, `render-loop`, `sequence-data`, `threading`, `ui`*

---

### Is it possible to create a multi-layer 2D global illumination plugin where a renderer effect discovers and uses material properties from dummy effects on other layers?

This approach is theoretically possible with the After Effects SDK. You can use the AEGP suite to enumerate layers in a composition and checkout their properties. However, the key challenge is properly communicating dependencies to AE's internal dependency tracking system. You need to explicitly declare all layer and effect dependencies your renderer effect relies on, so that changes to materials or other layers trigger re-renders. This ensures the disk cache is used correctly—cached results are only used when all dependencies remain unchanged. The AEGP layer enumeration and property checkout APIs support this pattern, though it requires careful dependency management to avoid either missing updates or unnecessary re-renders.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `params`*

---

### How can a renderer effect communicate its dependencies on other layers and effects to AE's caching system?

Dependencies must be explicitly registered through the AEGP API when your renderer effect enumerates and checks out layers. When you call layer checkout functions to read pixel data, masks, shapes, and transforms from dependent layers, and query properties of effects on those layers, AE tracks these accesses as dependencies. To ensure proper cache invalidation, you should checkout all relevant layer properties at render time—if any of those checked-out properties change, AE will invalidate the cached result and trigger a re-render. This allows the disk cache to function correctly by only using cached data when all dependencies are identical.

*Tags: `aegp`, `caching`, `compute-cache`, `dependency`, `layer-checkout`*

---

### Is it possible to make a multi-layer 2D global illumination effect where a renderer effect discovers material metadata effects on other layers and communicates these dependencies to AE's caching system?

It is theoretically possible using AEGP_RenderAndCheckoutLayerFrame to enumerate layers and pass their GUIDs via GuidMixInPtr() to register dependencies. However, a more practical approach is to use hidden layer parameters (up to 999) with a script button that assigns layers from the composition to these parameters, allowing checkout_layer() to work efficiently with SmartPreRender. You can also mix in layer order information via GuidMixInPtr() to handle dynamic layer changes without manual updates.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `smart-render`*

---

### What is the Compute Cache useful for in After Effects plugin development?

The Compute Cache is a useful learning resource for understanding how to work with sequence data and plugin state management in After Effects plugins.

*Tags: `compute-cache`, `reference`, `sequence-data`*

---

### Is GLM compatible with Vulkan and can it be used for matrix operations in graphics pipelines?

Yes, GLM is compatible with Vulkan and is recommended for use with it. GLM provides matrix functions like glm::perspective and glm::lookAt that can be used for graphics transformations, though note that GLM's perspective function uses gluPerspective conventions which may differ from other implementations.

*Tags: `compute-cache`, `gpu`, `reference`, `vulkan`*

---

### Can I save cached frames to a custom file format and read them back in a plugin?

Yes, you can save frames to any file format you want and read them back in. This is a common approach when caching large amounts of frame data that won't fit in sequence data. You can use custom binary formats (like .dat files) or other formats suitable for your needs. The plugin can write frames to disk during processing and read them back as needed, avoiding the limitations of storing arbitrarily large numbers of frames directly in sequence data.

*Tags: `caching`, `compute-cache`, `mfr`, `sequence-data`*

---

### What flags or methods are used in high-end plugins like Particular to cache frames and delay render updates until caching is complete?

The user is asking about caching mechanisms used in professional plugins like Particular that intelligently manage frame rendering by waiting for frame cache completion before updating the CTI (Current Time Indicator). This typically involves smart render and compute cache APIs provided by After Effects to optimize performance when scrubbing the timeline.

*Tags: `caching`, `compute-cache`, `render-loop`, `smart-render`*

---

### How do you add time values when using the compute cache API in After Effects?

When adding time values to the compute cache, you need to use AEGP (After Effects General Plugin) APIs to properly interact with the time-based caching system.

*Tags: `aegp`, `caching`, `compute-cache`*

---

### How can a plugin implement recursive operations where frame N depends on computed data from previous frames without checking out all intermediate frames during pre-render?

This is a known challenge when combining smart render with compute caches that require sequential frame dependencies. The user attempted several approaches: (1) using a compute cache with smart render, but this requires checking out all previous frames during pre-render when rendering frame 100, which is memory-inefficient; (2) using checkout params during compute cache, which doesn't account for previous effects; (3) using AEGP_CacheAndCheckoutFrame, which doesn't trigger pre_render/smart render callbacks on checked-out frames. A solution may involve manually managing frame dependencies outside the smart render system or using layer checkout strategically to load only necessary intermediate results rather than relying on automatic frame dependency detection.

*Tags: `aegp`, `compute-cache`, `layer-checkout`, `memory`, `sequence-data`, `smart-render`*

---

### How can you retrieve custom data computed from a previous frame in After Effects plugins?

You can use checkout Param during compute cache to access data from previous frames. However, this approach has limitations as it doesn't account for previous effects in the chain. For frame-sized custom data, you need to handle the data flow through the compute cache mechanism while being aware that parameter checkout alone may not fully solve the problem if previous effects need to be considered.

*Tags: `arb-data`, `compute-cache`, `params`, `render-loop`*

---

### How do you access computed data from previous frames in a compute cache when using smart render?

When accessing computed data from frame N-1 in a compute cache, if that frame hasn't been computed yet, you need access to the layer at the previous frame. This creates a cascading dependency—if N-2 is also not calculated, you need the layer at N-2 as well. With smart render, pre-loading all needed layers during pre-render can be expensive, especially if the user sets the timeline cursor at an arbitrary time like 10 seconds. The solution is to use classical checkout_param during compute cache, though this loads layers without their effects applied.

*Tags: `compute-cache`, `layer-checkout`, `render-loop`, `smart-render`*

---

### Is there a memory leak when using aegp_memHandle during the compute cache thread that persists even after calling the free memory function?

A developer reported that instruments showed no leaks, but memory allocated using aegp_memHandle during the compute cache thread was not being freed in reality when the freemem function was called. This issue occurred specifically during render operations, suggesting there may be a limitation with AEGP_memsuite during the render thread that prevents proper memory deallocation.

*Tags: `aegp`, `compute-cache`, `debugging`, `memory`, `render-loop`*

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

### What are the advantages of compute cache compared to arbdata for storing pointers to heap data?

Compute cache offers several advantages over arbdata: (1) it can be written/read from any thread (render or UI) with optional thread-safety using std::atomic, (2) it's designed per-frame rather than requiring a global context like sequence or arb data, (3) it provides functions to specify whether to wait if another thread is computing the same key. However, arbdata gets saved with the project while compute cache does not. Compute cache replaces sequence data now that MFR is introduced and sequence data can no longer be used.

*Tags: `arb-data`, `compute-cache`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### Can compute cache data be transferred to arbdata for project persistence?

Yes, data from the compute cache can be transferred to arbdata if needed so it gets saved in the project. Additionally, you can read the compute cache from the arb or sequence thread if required.

*Tags: `arb-data`, `compute-cache`, `params`, `sequence-data`*

---

### How can you generate a compute cache key when the key generation function only accepts AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP but requires AEGP_HashSuite1 which needs in_data?

Tim Constantinov asked about the apparent conflict between the parameters accepted by the key generation function and the requirements for using AEGP_HashSuite1. The question highlights that to generate a key, you need access to in_data or in_data->pica_basicP, but the function signature seems to only provide AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP. This appears to be an unresolved technical question about the proper API usage pattern.

*Tags: `aegp`, `compute-cache`, `debugging`*

---

### How do you generate a key and create a pointer for computeIfNeeded or reading a function in After Effects plugins?

When you call computeIfNeeded or want to read a function, you create a pointer and must set in_data->pica_basicP in this pointer along with the other necessary values. The compute cache operates as something independent where you must send everything needed for the computation.

*Tags: `aegp`, `compute-cache`, `params`, `smart-render`*

---

### How do you properly access the compute cache in After Effects plugins and avoid crashes?

When accessing compute cache data, you must create a pointer to access_cache_data and extract the SPBasicSuite from in_data->pica_basicP. To avoid crashes and maintain independence from the render thread, create a new object and copy only the parts you need, then delete it after computing. This prevents thread safety issues and ensures the cache operates independently.

```cpp
access_cache_data* accessCacheData = static_cast<access_cache_data*>(optionsP);
if ( !accessCacheData->in_data ) {return A_Err_ALLOC;}
SPBasicSuite* bsuite;
bsuite = accessCacheData->in_data->pica_basicP;
if (!bsuite){return A_Err_ALLOC; }

struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
};
```

*Tags: `aegp`, `compute-cache`, `debugging`, `memory`, `threading`*

---

### How can I force After Effects to invalidate cached frames when I modify global data in my plugin?

Simply setting PF_OutFlag_FORCE_RERENDER in outflags while changing a GuidMixInPtr value in extra->cb may not always work reliably. One workaround is to "kick" AE by triggering an internal frame cache check, such as changing the composition background color and changing it back. However, if you properly use GuidMixInPtr for forcing rerenders, it should work without this workaround—if you're experiencing issues, it may be worth reviewing source code examples from developers who have successfully implemented this.

*Tags: `aegp`, `compute-cache`, `params`, `smartfx`*

---

### How can a C++ plugin update a text layer every frame during rendering without locking the project?

This is a fundamental limitation in After Effects: calling AEGP_SetText from the render thread causes the project to lock. Several workarounds exist: (1) Write data to sequence data or compute cache from the render thread, then read and apply it from the UI thread using the aegp_idle hook, which is called multiple times per second; (2) Use your own text rendering engine in the render thread with an external library; (3) Write to disk and have a ScriptUI panel with a timer read it back (though this is unreliable). However, none of these work reliably in MediaEncoder or headless rendering without a GUI. The fundamental issue is that modifying a text layer from render could create infinite loops, which is why AE locks the project.

*Tags: `aegp`, `compute-cache`, `render-loop`, `sequence-data`, `threading`, `ui`*

---

### How can a plugin effect communicate implicit dependencies on other layers and effects to After Effects' caching system?

When building multi-layer effects that discover and depend on other layers' properties at runtime, you need to explicitly communicate these dependencies to AE's render dependency tracking system. This involves checking out the layers you depend on and notifying AE of those checkouts so the cache knows to invalidate when those layers change. The renderer effect should enumerate dependent layers, check out their pixels/masks/shapes/transforms, and use the appropriate AEGP calls to register these dependencies so AE's smart render and disk cache systems understand the effect needs to re-render when those layers are modified.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `memory`, `smart-render`*

---

### Is it possible to create a multi-layer global illumination plugin that discovers material metadata effects on other layers and properly communicates dependencies to After Effects' caching system?

This is a complex architectural question posed by fad regarding whether a 'renderer' effect can enumerate other layers, discover per-layer material definition effects, check out layer pixels/masks/transforms, and have those dependencies properly tracked by AE's dependency tracking and disk cache system. While Tobias Fleischer's experience suggests multi-layer effects are viable, the specific mechanism for communicating hidden dependencies (non-explicit UI parameter dependencies) to AE's caching system to trigger rerenders when materials change remains an open design challenge requiring careful SDK integration and likely custom dependency management.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `memory`, `render-loop`, `smartfx`*

---

### How can I make a multi-layer effect that depends on material properties defined by effects on other layers while maintaining smart render efficiency?

The most robust approach is to use hidden layer parameters (you can have up to 999) and a button parameter that triggers a script to assign layers from the composition to those parameters one by one. Alternatively, loop through the comp in SmartPreRender and mix layer order into GuidMixInPtr() for dependency tracking. This allows the renderer effect to check out all layers efficiently while maintaining AE's caching and smart render system. You only need to click an 'Update Comp' button when you add layers; layer order changes or removals can be handled automatically by the SmartPreRender logic.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `params`, `smart-render`*

---

### Can I pass a layer receipt GUID to GuidMixInPtr() to register layer dependencies without rendering pixels during SmartPreRender?

You can retrieve a GUID from an AEGP_FrameReceiptH using AEGP_GetReceiptGuid and pass it to extra->cb->GuidMixInPtr(). However, AEGP_RenderAndCheckoutLayerFrame is the expensive call that renders the entire layer, so calling it during SmartPreRender defeats the performance benefit. The suggested approach is to use layer parameters instead, which allow AE to track dependencies without rendering during the SmartPreRender phase.

*Tags: `aegp`, `caching`, `compute-cache`, `smart-render`*

---

### How do I write the generate_key function for the compute cache API without access to in_data?

The generate_key function doesn't have an in_data pointer, but you only need SPBasicSuite to acquire suites, not the full in_data. You have two options: (1) store SPBasicSuite in a global variable (shown in some SDK samples), or (2) use the cleaner approach of passing SPBasicSuite through the AEGP_CCComputeOptionsRefconP struct that all compute cache callbacks receive. If you want to use suite handling macros, create a local copy of the in_data struct and assign the passed SPBasicSuite value to its pica_basicP member. Do not pass in_data itself as it's only valid for the duration of a single AE call.

```cpp
PF_Err AEFX_AcquireSuite( PF_InData *in_data, /* >> */
PF_OutData *out_data, /* >> */
const char *name, /* >> */
int32_t version, /* >> */
const char *error_stringPC0, /* >> */
void **suite) /* << */
{
  PF_Err err = PF_Err_NONE;
  SPBasicSuite *bsuite;
  bsuite = in_data->pica_basicP;
  if (bsuite) {
    (*bsuite->AcquireSuite)((char*)name, version, (const void**)suite);
    if (!*suite) {
      err = PF_Err_BAD_CALLBACK_PARAM;
    }
  }
  return err;
}
```

*Tags: `aegp`, `compute-cache`, `mfr`, `smartfx`*

---

### Is there an official Adobe documentation reference for the compute cache API?

Yes, Adobe's official documentation for the compute cache API is available at https://ae-plugins.docsforadobe.dev/effect-details/compute-cache-api.html, which includes details about AEGP_ComputeCacheCallbacks, the generate_key function, and the AEGP_HashSuite1 for computing hashes.

*Tags: `aegp`, `compute-cache`, `documentation`, `reference`*

---

### How can I force a rerender after PF_Cmd_SEQUENCE_RESETUP or PF_Cmd_SEQUENCE_SETUP when effect state changes?

Use GuidMixInPtr to incorporate custom state flags (such as a watermarked or license status flag) into the state that After Effects uses to determine whether a cached frame is valid. When you push the relevant flag into the mix, After Effects will automatically trigger a re-render when that state changes. This feature is documented in the CC2015 SDK and later versions.

*Tags: `caching`, `compute-cache`, `render-loop`, `sdk`, `sequence-data`*

---

### How can a plugin clear the frame cache in After Effects?

Use AEGP_DoCommand() with command number 2372 to clear the frame cache. This allows subsequent playback to trigger PF_Cmd_RENDER for every frame instead of using cached frames.

*Tags: `aegp`, `caching`, `compute-cache`*

---
