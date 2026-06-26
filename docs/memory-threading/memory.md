# Memory Management

> 229 Q&As · source: AE plugin dev community Discord

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

### If a resource fails to allocate, does it need to be disposed of?

No, if a resource fails to allocate, it does not need to be disposed of since it was never successfully created in the first place.

*Tags: `debugging`, `memory`*

---

### What happens if you try to dispose of a world that failed to allocate due to insufficient memory?

If allocation fails, do not attempt to dispose of the world. Attempting to dispose of a world you didn't successfully allocate will trigger an error warning stating you should not dispose of a world you didn't allocate. This error appears on top of the original out of memory warning, creating a confusing user experience.

*Tags: `aegp`, `debugging`, `memory`*

---

### What needs to be updated when using PF_COPY buffer operations?

When using PF_COPY or other buffer operations, the source and destination rectangles (src and dst rects) will need to be updated to reflect the operation.

*Tags: `memory`, `mfr`, `output-rect`*

---

### How do plugins like Trapcode Particular cache and calculate multiple frames over time rather than just the last frame?

Plugins that need to cache multiple frames typically use their own internal state management system separate from the standard Sequence Data approach. While Sequence Data can store the last frame, plugins handling fluid simulations or particle systems that need to jump to arbitrary times in a composition use custom caching mechanisms. These plugins maintain their own frame cache, calculate values for multiple frames, and store them in memory structures that can be accessed when jumping to different times in the composition. The exact implementation depends on the plugin's architecture, but it generally involves managing a larger state buffer that holds multiple frames of simulation data rather than relying solely on the sequential frame-to-frame Sequence Data mechanism.

*Tags: `caching`, `memory`, `render-loop`, `sequence-data`*

---

### Can I save frames to arbitrary file formats and read them back in when caching Effect_World in sequence data?

Yes, you can save frames into any file format you want and read them back in. This approach allows you to work around sequence data storage limitations when caching multiple frames, similar to using a .dat file example.

*Tags: `caching`, `memory`, `sequence-data`*

---

### What algorithm does the Matte/Simple Choker Effect use to find and create reduced/expanded edges on alpha channels?

The questioner describes their own implementation which analyzes each transparent pixel and looks in 8 directions to find nearby opaque pixels to identify outline/choker matte pixels. However, no answer was provided in the conversation about the actual method used by Adobe's Matte/Simple Choker Effect.

*Tags: `gpu`, `memory`, `mfr`, `optimization`, `render-loop`*

---

### What algorithm does the Matte/Simple Choker Effect use to find and create reduced/expanded edges in alpha channels?

The user describes their current implementation which analyzes each transparent pixel and looks in 8 directions to find nearby opaque pixels to identify outline/choker matte pixels. However, this approach is very slow (30+ seconds). The conversation does not provide a definitive answer about what method Adobe's Choker effect actually uses, only describes the user's current slow implementation.

*Tags: `gpu`, `memory`, `mfr`, `optimization`, `render-loop`*

---

### What algorithm does the Matte/Simple Choker Effect use to create reduced/expanded edges on alpha channels?

The Matte/Simple Choker Effect likely uses a distance field approach rather than checking neighboring pixels. A distance field uses an intermediate 2D buffer of singular distance values (such as int16_t for signed distance values) to exchange memory for speed. The Jump Flooding Algorithm (JFA) is a fast method to generate distance fields and is well-suited for this purpose, avoiding the O(n²) complexity of checking neighbors for each pixel.

*Tags: `gpu`, `memory`, `optimization`, `render-loop`*

---

### Why is the neighbor-checking algorithm for creating choker mattes too slow and what is a better approach?

The neighbor-checking approach is too slow because checking 8 directions for each transparent pixel to find nearby opaque pixels has O(n²) complexity when expanding outlines by larger amounts (e.g., 30 pixels). A distance field approach is significantly faster as it uses a 2D buffer of distance values to trade memory for speed, with Jump Flooding Algorithm being a recommended fast implementation.

*Tags: `memory`, `optimization`, `render-loop`*

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

### How does the VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension avoids the need to allocate a staging buffer entirely. It allows you to turn a CPU-side void* directly into a VkDeviceMemory object, enabling direct data copying from an EffectWorld onto the GPU and allowing the GPU to write into it as well. This provides significant gains in both memory and speed, especially for large 4K frames. However, MoltenVK does not currently support this extension, though it is worth conditionally taking advantage of on Windows.

*Tags: `cross-platform`, `gpu`, `memory`, `vulkan`, `windows`*

---

### Is it normal to receive thousands of calls to PF_Arbitrary_DISPOSE_FUNC on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls on shutdown is likely due to MFR (Multi-Frame Rendering) duplicating arbitrary parameters for each thread, causing many copy and dispose cycles as parameters change and threads are cleaned up during shutdown.

*Tags: `arb-data`, `memory`, `mfr`, `threading`*

---

### Why does device-to-device buffer transfer in Vulkan have the same speed as host-to-device transfer when using external CUDA memory?

The transfer likely implicitly bounces off host memory or the CUDA buffer may actually be on host memory rather than device memory, even though the transfer is performed device-to-device. This can happen when Vulkan cannot reliably determine how to handle the caching of imported CUDA memory handles.

*Tags: `compute-cache`, `cuda`, `gpu`, `memory`, `vulkan`*

---

### Why does an arbitrary data struct work with int but fail with std::string?

std::string cannot be used in ARB data because it cannot be flattened or serialized to be saved on disk. Use char arrays (char[xxx] where xxx is the max size) instead for storing strings in arbitrary data.

*Tags: `arb-data`, `memory`, `params`*

---

### What causes PF_Interrupt_CANCEL to occur in the middle of a render in After Effects?

PF_Interrupt_CANCEL can occur during rendering when certain conditions are met, such as when CAPS LOCK is off and the composition has not been previously rendered to the RAM cache. The issue appears to be related to some interaction with keyboard state or cache status, as the interrupt does not occur when CAPS LOCK is on or when the composition has been fully previewed in the RAM cache beforehand.

*Tags: `caching`, `debugging`, `memory`, `render-loop`*

---

### How can you access the previous frame result during the compute thread in After Effects?

The user asked this question but indicated that despite reading the documentation, a full example would be more helpful. No complete answer was provided in the conversation.

*Tags: `compute-cache`, `memory`, `render-loop`, `smartfx`*

---

### How can C++ plugin code communicate with a CEP panel?

C++ can open a CEP by executing scripting that calls the CEP. For CEP to plugin communication, store values to transfer in JavaScript memory using global variables. For large data sizes that are expensive in memory, use a temp file instead. Alternatively, set a hidden checkbox that is activated by script to catch the value. Another solution is to run a hidden panel as a server that receives requests from the plugin and adjusts values in CEP.

*Tags: `cep`, `memory`, `params`, `scripting`*

---

### How should you handle thread safety when using global variables in a plugin?

Use mutexes to protect access to shared global variables. A mutex locks all threads from accessing the protected variable simultaneously - only one thread can write while holding the mutex. Atomic bools can also work but may cause errors in some cases. Initialize global variables only once since After Effects runs on two threads (UI and render) since CC 2014, and you need consistent values across threads.

```cpp
static int stuff;
// In code modifying the variable:
mtx.lock();
stuff = new_value;
mtx.unlock();
```

*Tags: `memory`, `threading`*

---

### What should you avoid doing with globalData in After Effects 2022 and later?

Do not write to globalData outside of globalsetup/setdown in AE 2022 and later, even for non-MFR plugins. After Effects has stricter requirements about globalData immutability. Instead, store mutable data in sequence data (with appropriate callbacks) or static variables.

*Tags: `memory`, `mfr`, `params`*

---

### How can a plugin render frame N that depends on custom data from previous frames without checking out all 100 previous frames in memory during pre-render?

This is a known challenge with recursive operations in After Effects plugins. The issue is that smart render checks needed frames during pre-render, which forces checking out all intermediate frames. The checkout param solution doesn't account for previous effects. AEGP_CacheAndCheckoutFrame doesn't trigger the pre_render/smart render thread, so it doesn't solve the problem. A potential approach is to use layer checkout parameters during compute cache operations, but this requires careful design to avoid excessive memory usage and to ensure previous effects are properly evaluated.

*Tags: `aegp`, `compute-cache`, `layer-checkout`, `memory`, `smart-render`*

---

### Why is there a memory leak when using AEGP_memorysuite on Mac but not Windows during export?

The memory leak appears to be platform-specific (Mac only) and related to how AEGP_memorysuite handles memory operations. The issue is less severe with the mem_QUIET flag. Using standard C++ memory allocation (new/delete) instead of AEGP_memorysuite resolves the leak, and replacing memcpy with strncpy partially solves the problem (from mem pointer to buffer works, but not in the reverse direction). This suggests the issue may be related to how memcpy interacts with AEGP's memory management on macOS, particularly when memory is freed from a different thread than where it was allocated.

*Tags: `aegp`, `debugging`, `macos`, `memory`, `threading`*

---

### How should memory be freed differently on Mac versus Windows?

Memory should be freed later in another thread on Mac but not on Windows, due to platform-specific threading and memory management differences.

*Tags: `macos`, `memory`, `threading`, `windows`*

---

### What is a recommended approach for tracking memory allocations and deallocations in C++ plugins?

Wrap memory operations in custom optionally-logging wrappers to get a definitive record of what happened. Additionally, have all C++ classes descend from a base class like ObjectBase that keeps track of live instances of each type, allowing you to report current counts for allocations versus frees.

*Tags: `caching`, `debugging`, `memory`*

---

### Is there a limitation with AEGP_memsuite when allocating and freeing memory during the render thread?

The user observed that memory allocated using aegp_memHandle during the compute cache thread is not being freed when the freemem function is called, even though instruments show no leaks. This suggests there may be specific limitations or requirements for using AEGP_memsuite during the render thread that differ from other threads.

*Tags: `aegp`, `compute-cache`, `debugging`, `memory`, `render-loop`, `threading`*

---

### Why might a plugin cause a third-party library like Cineware to crash during After Effects project loading even when the plugin code isn't executing at the crash point?

The plugin may be defining a symbol that clashes with the third-party library's loading process, or it may be causing some state corruption in After Effects that affects library initialization later. The fact that preloading Cineware before opening the project allows it to work suggests a symbol collision or initialization order issue rather than a direct code bug.

*Tags: `debugging`, `memory`, `windows`*

---

### Why doesn't macOS Instruments detect memory leaks from PF_EffectWorlds that are allocated but never disposed?

PF_EffectWorlds are likely allocated deep within the AE engine rather than directly within the plugin code, so memory leak detection tools like Instruments don't recognize them as true leaks since they're allocated at a different level in the AE memory management hierarchy.

*Tags: `aegp`, `debugging`, `macos`, `memory`*

---

### How can you improve memory management for AESDK objects in After Effects plugins?

Create C++ wrappers around AESDK objects to enable automatic memory management. There are already some C++ wrappers of portions of the SDK available on GitHub that can be used for this purpose.

*Tags: `aegp`, `build`, `memory`*

---

### Why doesn't macOS Instruments detect memory leaks from undisposed PF_EffectWorlds even though After Effects runs out of memory?

PF_EffectWorlds allocated in After Effects plugins may not be detected by Instruments leak detection because After Effects itself may be managing the memory lifecycle through its own memory management systems, or the allocations may be attributed to After Effects' internal memory pools rather than the plugin's direct allocations. Instruments can detect direct memory leaks (e.g., data allocated in pre-render that isn't disposed), but large framework-managed allocations like PF_EffectWorlds may be tracked differently and not flagged as leaks even when they cause memory exhaustion.

*Tags: `debugging`, `macos`, `memory`, `plugin-leak-detection`*

---

### How should a plugin handle GPU out-of-memory errors differently for queue renders versus interactive scrubbing?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects may retry with lower thread counts during MFR queue renders, allowing the frame to succeed. However, during interactive scrubbing or non-queue renders, After Effects may not warn the user and simply display a black frame. The plugin should only show an out_data warning message about resolution/bitdepth requirements if the user is not rendering from the queue. Unfortunately, there is no documented way to distinguish between a render request from the queue versus interactive timeline scrubbing in the plugin API.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `render-loop`*

---

### How should a plugin handle GPU out-of-memory errors and warn users appropriately?

When a plugin runs out of GPU memory, it should return PF_Err_OUT_OF_MEMORY. During MFR (Multi-Frame Rendering) queue renders, After Effects may retry with lower thread counts. However, for interactive scrubbing in the composition, After Effects may not warn users and they see black frames instead. The challenge is that there's no standard way to distinguish between queue renders and interactive scrubbing to conditionally show warning messages in out_data without failing the render. A workaround would be to return the error code and accept that After Effects' handling may be limited, or document the resolution and bitdepth requirements for users to manually adjust.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `ui`*

---

### How should a plugin handle GPU memory exhaustion errors during render versus interactive scrubbing?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects may not warn the user if they're scrubbing the timeline interactively (only black frames appear). During MFR queue renders, AE may retry with lower thread counts. One approach is to use a C++ text library to render an "Out of GPU memory" message directly onto the frame output, making the error visible to the user regardless of render context. However, there is currently no known way to distinguish between queue renders and interactive scrubbing to conditionally show warnings in out_data.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `render-loop`*

---

### Why does a plugin act weird on Media Encoder and render without the effect applied when queued from After Effects?

The issue is likely related to static global variables in the plugin or elements defined in globalData or during global initialization thread. These can cause problems across different Media Encoder versions.

*Tags: `debugging`, `memory`, `premiere`, `threading`*

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

### What are the scenarios where nullptr checks on extraP and its members fail?

The conversation does not provide a specific answer to this question. The asker acknowledges the issue with 'that sucks' but no detailed explanation of failure scenarios is given.

*Tags: `debugging`, `memory`*

---

### What are the size limitations for storing custom metadata in AE projects using xmpPacket from ExtendScript?

There is a practical size limit of less than 500kb for xmpPacket data. AE itself stores more than 500kb in its own XMP Packet for reference. Tools like AE Viewer can remove XMP packets to significantly reduce file sizes (e.g., from 50mb down to 50kb).

*Tags: `arb-data`, `memory`, `scripting`*

---

### How should cv::Mat pixel data be correctly copied into an After Effects PF_LayerDef structure?

When copying OpenCV Mat data to After Effects PF_LayerDef, you should not manually allocate layerDef->data since After Effects manages PF_LayerDef creation and destruction through PF_NewWorld() and PF_DisposeWorld(). The layerDef->data should already be allocated by After Effects and should not be nullptr. You should use the pre-allocated buffer and copy your OpenCV pixel data directly into it, being careful about channel ordering (OpenCV uses BGRA while After Effects uses ARGB format).

```cpp
// Correct approach: use AE-allocated buffer
PF_Pixel8* pixelData = reinterpret_cast<PF_Pixel8*>(layerDef->data);
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
        PF_Pixel8& aePixel = pixelData[y * width + x];
        aePixel.alpha = pixel[3];  // OpenCV stores alpha at index 3
        aePixel.red = pixel[2];    // Red at index 2
        aePixel.green = pixel[1];  // Green at index 1
        aePixel.blue = pixel[0];   // Blue at index 0
    }
}
```

*Tags: `layer-checkout`, `memory`, `render-loop`*

---

### Should I manually allocate memory in effect plugins or follow the sample plugins' approach?

You should follow what the sample plugins are doing rather than manually allocating memory. Manual allocation is problematic because After Effects won't know whether to use delete[] or free() for deallocation, and AE won't be able to track memory usage if you allocate it yourself.

*Tags: `build`, `memory`, `plugin`*

---

### Can I directly modify the output PF_LayerDef pointer or do I need to use PF_NewWorld?

You can directly adjust the output PF_LayerDef data from the Render command without needing PF_NewWorld. However, you must not modify the layer settings (width, height, rowbytes) directly as these are defined by the host app. Instead of directly assigning mat.data to layerDef->data, you should copy the data from your matrix into layerDef->data using a loop while respecting the layerDef->rowbytes alignment. The proper approach is shown in the skeleton plugin example, which uses the Iterate Suite to write into output->data.

```cpp
static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        for (int x = 0; x < layer->width; ++x) {
            cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
            PF_Pixel& aePixel = *sampleIntegral32(*layer, x, y);
            aePixel.alpha = pixel[3];
            aePixel.red = pixel[2];
            aePixel.green = pixel[1];
            aePixel.blue = pixel[0];
        }
    }
}
```

*Tags: `memory`, `mfr`, `output-rect`, `pipl`*

---

### How should I properly handle pixel data alignment when copying a matrix into a layer?

When copying pixel data into a PF_LayerDef, you must respect the layer's rowbytes alignment. Use a helper function like sampleIntegral32 that calculates the pixel address as (char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)). Ensure that mat.data has the same alignment as the layer rowbytes. You can optimize the copy by using memcpy() to copy entire rows at a time instead of individual pixels.

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data +
    (y * def.rowbytes) +
    (x * sizeof(PF_Pixel)));
}
```

*Tags: `memory`, `mfr`, `output-rect`*

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

### Can cv::mixChannels be used to convert ARGB back to BGRA to enable memcpy of entire rows?

Yes, cv::mixChannels can be used to convert between channel formats. However, a more efficient approach is to use OpenCV's Mat constructor with the step parameter (rowbytes) to create a Mat that points directly to the layer's pixel data without copying. Use cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes). Additionally, you should benchmark different approaches - mixChannels followed by memcpy might be slower than using IterateSuite (one thread per row) and doing manual channel swizzling when copying from the matrix to the layer.

```cpp
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Tags: `gpu`, `layer-checkout`, `memory`, `render-loop`*

---

### What are the practical limits for working with a composition containing thousands of layers in After Effects?

After Effects struggles significantly even at 512 layers. As a user reports, working with over 500 layers becomes a nightmare in terms of performance and usability. The software has inherent limitations that make working with very large numbers of layers (such as 10,000) impractical, even if they are instances of only a small number of source files.

*Tags: `caching`, `memory`, `performance`, `render-loop`*

---

### What causes slowness when displaying a timeline with many layers in After Effects?

The slowness is primarily in the UI rendering rather than the plugin rendering itself. The bottleneck is likely in After Effects internal calls such as 'get layer info' and 'get layer sprite'. Even with many layers hidden by default, performance issues persist. After Effects also has poor memory management, making it inefficient with large numbers of layers (10k+).

*Tags: `aegp`, `memory`, `performance`, `ui`*

---

### Is it expected to receive 5000+ calls to arb dispose when shutting down After Effects?

Yes, this is expected behavior. After Effects allocators generally do not dispose of memory until either available memory is saturated, threads are destroyed, or the application exits. You can test this behavior by artificially setting the available memory to a low value like 2GB, which will trigger more destruction calls during runtime.

*Tags: `arb-data`, `debugging`, `memory`*

---

### What is the purpose of the rowbytes parameter in layer data?

Rowbytes exists as an optimization so that layers can be cropped without having to allocate extra memory. You can crop a layer by simply updating width and height and making the data pointer point to the new top-left pixel, leaving rowbytes untouched while keeping a reference to the old data for proper deallocation. AE does this often, such as when dragging a layer partially off the composition.

*Tags: `layer-checkout`, `memory`, `output-rect`*

---

### Can the width-to-rowbytes ratio reliably determine bit depth?

While theoretically rowbytes can be anything, in reality padding is usually minimal, so the width-to-rowbytes ratio should reliably tell you the bit depth with certainty in practice. However, edge cases may exist and Adobe could change this behavior in the future.

*Tags: `memory`, `params`*

---

### What is the purpose of rowbytes in After Effects layer data?

rowbytes exists as an optimization so that layers can be cropped without allocating extra memory. You can crop a layer by simply updating width and height and making the data pointer reference the new top-left pixel, leaving rowbytes untouched while keeping a reference to the old data for proper cleanup. After Effects does this often, such as when dragging a layer partially off the composition.

*Tags: `caching`, `layer-checkout`, `memory`, `output-rect`*

---

### Does rowbytes change when a layer is moved partially off-comp and then cache is purged?

When you load a layer and move it partially off-comp, rowbytes will be larger than the minimum required. However, after purging cache, rowbytes will likely return to 4*width (or 4*4*width for 32-bit data), indicating the cache optimization is recalculated.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `memory`*

---

### Why might rowbytes differ in an 8-bit EffectWorld when the layer is cropped?

Rowbytes can differ based on cropping and alignment. For example, in an 8bpc EffectWorld that is cropped 25% horizontally, rowbytes could be calculated as 32*4*width, reflecting the stride needed for the allocated buffer dimensions rather than just the visible dimensions.

*Tags: `gpu`, `memory`, `output-rect`*

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

### Do you need to lock the handle when calling GetConstSequenceData for sequence data?

Yes, you need to lock the handle even when reading only. Use PF_LOCK_HANDLE(const_seq) to lock the handle before accessing it, and unlock it after use. This is necessary in multi-frame render (MFR) scenarios with multiple plugin applications, and the lack of locking can cause nullptr returns and crashes. While Adobe documentation doesn't clearly document this requirement, it appears to be essential for thread safety.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `aegp`, `debugging`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### Do PF_LOCK_HANDLE and host_lock_handle functions actually do anything in After Effects?

According to Adobe engineers, PF_LOCK_HANDLE and host_lock_handle functions are dummy functions that do nothing at all. As stated by Jason Bartell from Adobe in a 2022 discussion: 'Executive summary is handle locking and unlocking has not done anything in After Effects since roughly CS6.'

*Tags: `aegp`, `debugging`, `memory`*

---

### Does the old GLator sample have memory leaks in its GL rendering and texture download functions?

Yes, the GLator sample has major memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, meaning bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBO's.

*Tags: `debugging`, `gpu`, `memory`, `opengl`*

---

### What causes the generic exception handler in Adobe suites manager to appear, and why is it more frequent in newer AE versions?

The generic exception handler/catch-all in the Adobe suites manager appears when double freeing or releasing a suite pointer. It has existed since CS2 or CS3 days. It appears more often in newer AE versions likely due to MFR (Multi-Frame Rendering) with global suite handles or suite pointers being shared between threads.

*Tags: `aegp`, `memory`, `mfr`, `threading`*

---

### How should error handling be structured when developing After Effects plugins?

Use std::expected for function returns instead of traditional error codes, allowing for proper .then chaining. Wrap all AE handles in classes with proper construction and destruction via smart pointers. Implement an allocation class manager that clears allocations when exiting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug mode, use a cache validator to detect extra memory still allocated at endpoints.

*Tags: `aegp`, `debugging`, `error-handling`, `memory`*

---

### Is the path value converted properly for 16/32 bit processing?

Eric was investigating whether path values are being converted correctly between 8-bit and 16/32-bit processing paths. The issue was occurring during a blur function and was resolved by precomposing the layer with the effect first, suggesting it may be a bit-depth conversion detail.

*Tags: `debugging`, `memory`, `params`*

---

### What were the root causes of bugs in FastBoxBlur?

Two critical bugs were identified: (1) Incorrect bit depth detection causing memory corruption, and (2) Improper stride handling for padded buffers. The solution involved fixing bit depth detection to use rowbytes calculation and implementing proper stride handling throughout the blur algorithm.

*Tags: `debugging`, `memory`, `render-loop`*

---

### How slow is CPU to GPU upload and has it been benchmarked?

The question was asked but not directly answered in the conversation. Jonah asked for benchmarking data on CPU→GPU upload performance, but no specific benchmark results were provided.

*Tags: `gpu`, `memory`, `performance`*

---

### What GPU APIs are recommended for After Effects plugins?

For GPU work in After Effects plugins, several options are viable: Vulkan is recommended for maximum control over performance and memory transfers, with MoltenVK providing Metal support on macOS and KosmicKrisp coming as a successor. WebGPU is easier to use and cross-platform but may not allow platform-specific optimizations. OpenGL is mature but deprecated and implemented via Metal/DirectX/Vulkan under the hood. A practical approach is to implement a CPU plugin that does GPU rendering underneath. Vulkan interoperability features allow importing textures from CUDA, OpenCL, DirectX, Metal, or OpenGL.

*Tags: `cross-platform`, `cuda`, `gpu`, `memory`, `metal`, `opengl`, `vulkan`*

---

### Why does GPU mode still run slow when the work is being done on GPU?

The performance issue occurs when critical operations fall back to CPU processing. For example, in UV unwrapping for 3D models, if a mask is present in GPU mode, the data passed is post-mask which breaks the mathematical requirements, forcing the hard work back to the CPU path, negating GPU performance benefits.

*Tags: `gpu`, `memory`, `performance`, `render-loop`*

---

### What is the best way to pass data from an AEGP plugin to an effects plugin?

Use an AEGP plugin with an idle hook to process a queue. Every frame, push your string result into a queue, and the idle hook processes the queue occasionally. The AEGP plugin and FX plugin are separate binaries, so you'll need a C interface to push strings into the queue.

*Tags: `aegp`, `memory`, `render-loop`, `threading`*

---

### What are the risks of encoding data into dead pixels and retrieving them with sampleImage?

The solution is dependent on render format which can be tricky. CPU and GPU have different byte orders in After Effects (ARGB vs RGBA), and GPU formats like CUDA may use BGRA. Any swizzeling or color format conversion could mangle the data. Additionally, after threading changes in AE, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering yet.

*Tags: `cuda`, `gpu`, `memory`, `render-loop`, `threading`*

---

### How can you ensure byte order doesn't affect encoding data into pixel values?

sampleImage always returns RGBA. As long as you encode each color separately and limit it to one byte per color, byte order or even bit depth won't matter.

*Tags: `gpu`, `memory`, `output-rect`*

---

### How should you handle caching when scraping data from dummy effects on other layers?

Call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects, and AE will cache everything fine.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`*

---

### What alignment guarantees does PF_LayerDef::rowbytes have in After Effects?

rowbytes will always be a multiple of sizeof(PF_Pixel8), which is 4 bytes, but 16-byte alignment is not guaranteed. The minimum guarantee is that rowbytes must be a multiple of the alignment requirement of the pixel type being used. Since After Effects effects don't have pixel types smaller than ARGB8, rowbytes will always be at least 4-byte aligned in practice. Misaligned pointers would result in undefined behavior according to the C specification.

*Tags: `aegp`, `memory`, `params`, `reference`*

---

### What does the C specification say about pointer alignment and undefined behavior?

According to the C specification (section 6.3.2.3, paragraph 7 of the C11 standard), a pointer to an object type may be converted to a pointer to a different object type, but if the resulting pointer is not correctly aligned for the referenced type, the behavior is undefined. Apple documents this issue in their Xcode documentation on misaligned pointers: https://developer.apple.com/documentation/xcode/misaligned-pointer

*Tags: `cross-platform`, `debugging`, `memory`, `reference`*

---

### How do you efficiently copy image data row by row between Premiere and Vulkan/OpenGL using memcpy?

For copying from Premiere to 3D APIs (Vulkan/OpenGL), use row-by-row memcpy by calculating pointers for each row: cast the data pointers to char, offset by row byte counts, and copy one row at a time. The formula is: Void PrPtr = (Char)PRData + yPrRowByte; Void ApiPtr = (Char)ApiData + yAPIRowByte; Memcpy(ApiPtr, PrPtr, APIRowByte). The APIRowByte is calculated as width * 4 (for number of channels) * sizeOfPixel (1 for 8-bit, 4 for 32-bit). The PrRowByte comes from Premiere's definitions and can be negative. This approach works across all bitdepths. However, when copying from 3D APIs back to Premiere, crashes occur during memcpy because Premiere image buffers are 16-byte aligned with potential padding at the end of each line, so the reverse copy requires accounting for this buffer layout.

```cpp
Void PrPtr = (Char)PRData + yPrRowByte;
Void ApiPtr = (Char)ApiData + yAPIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `gpu`, `memory`, `opengl`, `premiere`, `vulkan`*

---

### How do you efficiently copy pixel data row by row when converting between Premiere and Vulkan/OpenGL?

In Premiere, use memcpy with row-based pointers: cast both Premiere and API data pointers to char, add the row offset (yPrRowByte for Premiere, yAPIRowByte for API), then memcpy the row. The APIRowByte calculation is width × NUM_channels × sizeOfPixel (1 for 8-bit, 4 for 32-bit). The PrRowByte comes from Premiere definitions and may be negative. Note that image buffers are 16-byte aligned with potential padding at line ends. In After Effects, the row-byte should already include 16-byte SIMD padding. When copying from 3D API back to Premiere causes crashes, verify the API pointer is correctly mapped and use vkGetImageSubresourceLayout in Vulkan to ensure proper source/destination image layout before copying.

```cpp
Void PrPtr = (Char)PRData + yPrRowByte;
Void ApiPtr = (Char)ApiData + yAPIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `gpu`, `memory`, `opengl`, `premiere`, `vulkan`*

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

### If a resource fails to allocate, does it still need to be disposed?

No, if a resource fails to allocate, it does not need to be disposed of. Only successfully allocated resources require cleanup and disposal.

*Tags: `debugging`, `memory`, `resource-management`*

---

### How can you test whether memory disposal is working correctly in an After Effects plugin?

One practical approach is stress-testing by rendering a long video (e.g., 5 hours) to identify memory leaks. If the plugin runs without crashing and memory usage remains stable throughout the render, it indicates that necessary resources have been properly disposed. This method relies on observation rather than formal memory profiling tools.

*Tags: `debugging`, `disposal`, `memory`, `testing`*

---

### What is the current behavior of the After Effects Memory Suite functions?

According to discussions on the aescripts Slack channel, the Memory Suite functions originally worked as intended in earlier versions of After Effects. However, in current versions, these functions essentially just call new and delete, suggesting they may no longer provide the specialized memory management they once did. This is important context when deciding how to handle memory allocation and deallocation in plugins.

*Tags: `aegp`, `memory`, `reference`*

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

### How can you persist sequence data without consuming memory?

You can dump sequence data to a file (encoded or not) and read it back as needed. This approach allows you to save a sequence of data without loading it all into memory at once.

*Tags: `caching`, `file-io`, `memory`, `sequence-data`*

---

### What is an efficient algorithm for creating edge outlines or choker mattes based on alpha channels?

Instead of checking neighbors for each transparent pixel (which results in O(n^2) complexity), use a distance field approach. Create an intermediate 2D buffer of signed distance values (e.g., int16_t) to exchange memory for speed. The Jump Flooding Algorithm (JFA) is a fast method for generating distance fields and is well-suited for this exact use case.

*Tags: `algorithm`, `gpu`, `matte`, `memory`, `performance`*

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

### Should layer checkout and check-in be done immediately after use in a render loop, or can check-in be deferred?

Layer checkout and check-in should be handled carefully in render loops. For each frame checked out, you should check-in immediately after use, even if an error occurs. The recommended pattern is: for each frame, checkout the frame, perform work if no error occurs, then check-in the frame (using Err2 or equivalent error handling). Alternatively, you can defer check-in until the end of the render thread, but every checked-out layer must be checked in whether it was used or not. Error handling during checkout must also be monitored.

```cpp
For X
    Checkout frame x
    If (no err) do job
    Err2(check-in frame x)
End of loop
```

*Tags: `aegp`, `debugging`, `layer-checkout`, `memory`, `render-loop`*

---

### How can VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension (available on Windows) avoids the need to allocate a staging buffer by allowing a CPU-side void* to be turned directly into a VkDeviceMemory object. This enables copying data directly from an EffectWorld onto the GPU and allows the GPU to write back into it without redundant upload/download copies, resulting in significant gains in both memory and speed, especially for large 4K frames.

*Tags: `gpu`, `memory`, `optimization`, `vulkan`, `windows`*

---

### Is it normal to receive thousands of PF_Arbitrary_DISPOSE_FUNC calls on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls during shutdown is likely due to Multi-Frame Rendering (MFR) duplicating arbitrary parameters for each thread, combined with parameter changes triggering dispose/copy cycles.

*Tags: `arb-data`, `memory`, `mfr`, `shutdown`, `threading`*

---

### Why is device-to-device buffer transfer between Vulkan and CUDA no faster than host-to-device transfer?

When importing a CUDA handle into Vulkan for external memory, the transfer may implicitly bounce through host memory if Vulkan cannot reliably determine how to handle the memory's caching. Additionally, the AE_cuda buffer may be allocated on the host rather than the device, which would explain why device-to-device transfer speeds match host-to-device speeds. Check whether the buffer is actually allocated on the device or if it's a host-side allocation.

*Tags: `cross-platform`, `cuda`, `gpu`, `memory`, `vulkan`*

---

### Why does using std::string in an arbitrary parameter data structure cause errors in After Effects plugins?

std::string cannot be flattened or serialized to disk, which is required for After Effects to save and restore arbitrary parameter data when projects are saved and reopened. Use fixed-size char arrays (char[max_size]) instead to store string data in Arb parameter structures.

*Tags: `arb-data`, `memory`, `params`*

---

### What are the constraints on writing to sequence_data in After Effects CC2022 and later?

In CC2022 and later, you cannot write to sequence_data whenever you want. Writing is only allowed during specific events: sequence setup, sequence resetup, user changed param, do dialog, and external dependencies. For unrestricted modifications, use a global static variable initialized once instead. Note that in_data->sequence_data is for reading, while out_data->sequence_data is for writing.

*Tags: `memory`, `params`, `sequence-data`*

---

### How should global variables be used in multi-threaded plugin development?

Define global variables at the top of the main .cpp file, initialized only once, to ensure the same value is accessed across both UI and render threads (which have run separately since CC 2014). Use atomic bools for thread-safe access, though mutexes may be needed in some cases. Avoid using static variables with MFR (Multi-Frame Rendering), but non-MFR plugins should be fine. Pass pointers to these globals carefully, as passing pointers (e.g., bool*) can cause crashes—use value types (e.g., bool) instead.

```cpp
static bool global_variable;
// In PF_cmd or prerender:
if(extra->cb->GuidMixInPtr) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(bool), reinterpret_cast<void*>(&global_variable));
}
```

*Tags: `memory`, `smart-render`, `threading`*

---

### How can a plugin implement recursive operations where frame N depends on computed data from previous frames without checking out all intermediate frames during pre-render?

This is a known challenge when combining smart render with compute caches that require sequential frame dependencies. The user attempted several approaches: (1) using a compute cache with smart render, but this requires checking out all previous frames during pre-render when rendering frame 100, which is memory-inefficient; (2) using checkout params during compute cache, which doesn't account for previous effects; (3) using AEGP_CacheAndCheckoutFrame, which doesn't trigger pre_render/smart render callbacks on checked-out frames. A solution may involve manually managing frame dependencies outside the smart render system or using layer checkout strategically to load only necessary intermediate results rather than relying on automatic frame dependency detection.

*Tags: `aegp`, `compute-cache`, `layer-checkout`, `memory`, `sequence-data`, `smart-render`*

---

### Why is there a memory leak when using AEGP_memorysuite on macOS but not Windows during export?

A user reported a memory leak when using AEGP_memorysuite to allocate, lock, memcpy to/from, and unlock memory handles during export (not preview render). The leak was macOS-specific and did not occur on Windows. Testing with different flags (mem_NONE, mem_CLEAR, mem_QUIET) showed the leak was less severe with mem_QUIET. Replacing the AEGP memory handle with a temporary pointer using new/delete eliminated the leak. Additionally, replacing memcpy with strncpy solved the issue when copying from the mem pointer to a buffer, but not in the opposite direction. This suggests a potential platform-specific issue with how AEGP_memorysuite handles memory during the export pipeline, possibly related to threading or memory alignment differences between macOS and Windows.

*Tags: `aegp`, `debugging`, `macos`, `memory`, `threading`, `windows`*

---

### How can you track and debug memory allocation and deallocation issues in After Effects plugins?

Wrap memory allocation and deallocation in custom logging wrapper functions to get a definitive record of what happened. Additionally, create a base class (like ObjectBase) that all C++ classes descend from to keep track of live instances of each type, which helps identify memory leaks and allocation mismatches. This approach also makes it easy to report current counts of allocations versus frees.

*Tags: `debugging`, `macos`, `memory`, `threading`, `windows`*

---

### Is there a memory leak when using aegp_memHandle during the compute cache thread that persists even after calling the free memory function?

A developer reported that instruments showed no leaks, but memory allocated using aegp_memHandle during the compute cache thread was not being freed in reality when the freemem function was called. This issue occurred specifically during render operations, suggesting there may be a limitation with AEGP_memsuite during the render thread that prevents proper memory deallocation.

*Tags: `aegp`, `compute-cache`, `debugging`, `memory`, `render-loop`*

---

### Is it safe to allocate memory in one thread and free it in another thread in After Effects plugins?

It is generally not recommended to allocate memory in one thread and free it in another thread in After Effects plugins. AE makes few threading guarantees beyond pairing SETUP and SETDOWN calls. Even using mutex locks does not reliably solve cross-thread allocation/deallocation issues, as the plugin framework does not provide clear guarantees about thread lifecycle and frequency.

*Tags: `aegp`, `debugging`, `memory`, `threading`*

---

### Why doesn't Instruments detect memory leaks from PF_EffectWorlds allocated in After Effects plugins?

Memory leaks from PF_EffectWorlds may not be detected by Instruments because these objects are allocated deep within the After Effects engine rather than directly within the plugin code. The leak detection tools only see allocations at the plugin level, not at the internal AE engine memory allocation level where EffectWorlds are actually created. This means that even significant memory leaks from unmanaged EffectWorlds won't show up in Instruments despite causing the application to run out of memory.

*Tags: `aegp`, `debugging`, `macos`, `memory`*

---

### What is a good approach to manage memory safety for After Effects SDK objects?

Create C++ wrapper classes around AESDK objects to enable automatic memory management. This approach leverages C++ features like constructors and destructors to ensure proper allocation and deallocation of SDK objects. Additionally, there are already existing C++ wrappers for portions of the After Effects SDK available on GitHub that can be referenced or reused for this purpose.

*Tags: `aegp`, `build`, `memory`, `open-source`, `reference`*

---

### How can you manage memory for AESDK objects in C++ plugins?

Create C++ wrappers around all AESDK objects to get automatic memory management. This approach helps prevent memory leaks and ensures proper cleanup of After Effects SDK resources.

*Tags: `aegp`, `cpp`, `memory`, `plugin-development`*

---

### Why does macOS Instruments fail to detect memory leaks from undisposed PF_EffectWorlds in After Effects plugins?

James Whiffin reported that while Instruments successfully detected memory leaks from data allocated in pre-render functions that weren't disposed, it failed to detect leaks from PF_EffectWorlds that were allocated but never disposed. Despite these allocations consuming gigabytes and causing After Effects to run out of memory, they didn't appear in Instruments' leak detection output even when sorted by size. This suggests that PF_EffectWorlds may be allocated through memory management mechanisms that Instruments cannot track, or that After Effects' memory management for effect worlds operates outside standard system memory tracking.

*Tags: `aegp`, `debugging`, `macos`, `memory`*

---

### How can a plugin distinguish between a render queue render and a user scrubbing the timeline in the composition to handle GPU memory errors appropriately?

James Whiffin raised this as an unsolved problem in AE plugin development. When a plugin returns PF_Err_OUT_OF_MEMORY, AE may not warn the user if they're scrubbing the timeline (resulting in a black frame), but the plugin cannot reliably distinguish between queue renders and interactive scrubbing. While returning the error code prevents render queue failures (allowing AE to retry with lower thread counts), there's currently no documented method to detect the render context and conditionally display user warnings via out_data. This remains a limitation when plugins need multiple 32bpc GPU buffers at high resolutions.

*Tags: `debugging`, `gpu`, `memory`, `mfr`*

---

### How should a plugin handle GPU out-of-memory errors differently for interactive scrubbing versus render queue operations?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects handles it differently depending on context. During MFR (Multi-Frame Rendering) queue renders, AE may retry with a lower thread count. However, during interactive timeline scrubbing, AE doesn't warn the user and may display a black frame instead. The challenge is that PF_Err_OUT_OF_MEMORY warnings in out_data will fail the render, and there is currently no documented method to distinguish between a render queue request versus interactive scrubbing. Plugins must accept that they cannot reliably warn users about resolution/bitdepth being too high for available GPU memory in interactive contexts without failing the frame.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `smartfx`*

---

### What is the best practice for handling GPU out-of-memory errors in After Effects plugins?

When a plugin runs out of GPU memory, it should return PF_Err_OUT_OF_MEMORY. During MFR (multi-frame rendering), After Effects may retry the frame with a lower thread count. For interactive scrubbing where the user isn't rendering from the queue, one approach is to use a C++ text library to render an error message directly onto the frame (e.g., "Out of GPU memory") so the user understands why a black frame appeared. However, there is currently no reliable way to distinguish between a render queue request and interactive timeline scrubbing to conditionally show warnings in out_data.

*Tags: `gpu`, `memory`, `mfr`, `output-rect`, `render-loop`*

---

### Why does a plugin sometimes render without the effect applied when queued to Media Encoder from After Effects?

Static global variables in the plugin and elements defined in globalData or during global initialization thread can cause issues with Media Encoder compatibility. These should be reviewed and potentially refactored to avoid state persistence issues when the plugin runs in ME's different execution context.

*Tags: `debugging`, `media-encoder`, `memory`, `threading`*

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

### What suites are unsafe to call from the Death Hook in After Effects plugins?

There are restrictions on which suites can be safely called from the Death Hook. The Utility Suite, specifically ReportInfo, has been reported to cause unhandled exceptions when called from Death Hook. This suggests that certain suites with external dependencies or state management are not safe to invoke during plugin shutdown, and developers should be cautious about suite usage in Death Hook contexts.

*Tags: `aegp`, `debugging`, `memory`, `threading`*

---

### How should you properly copy pixel data from an OpenCV cv::Mat to an After Effects PF_LayerDef structure?

When copying from cv::Mat to PF_LayerDef, you should not manually allocate layerDef->data yourself—AE manages PF_LayerDef creation and destruction through PF_NewWorld()/PF_DisposeWorld(). The layerDef->data pointer should already be allocated by AE. The pixel data copy itself needs to respect both memory layout and channel ordering: OpenCV uses BGRA order (blue at index 0, green at 1, red at 2, alpha at 3) while AE's PF_Pixel8 struct has ARGB fields. Additionally, ensure you're using the correct row byte calculation and pixel indexing that accounts for the actual rowbytes value, not just width * sizeof(PF_Pixel8), as AE may add padding.

```cpp
PF_Pixel8* pixelData = reinterpret_cast<PF_Pixel8*>(layerDef->data);
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
        PF_Pixel8& aePixel = pixelData[y * (layerDef->rowbytes / sizeof(PF_Pixel8)) + x];
        aePixel.alpha = pixel[3];
        aePixel.red = pixel[2];
        aePixel.green = pixel[1];
        aePixel.blue = pixel[0];
    }
}
```

*Tags: `aegp`, `memory`, `params`, `render-loop`*

---

### Should I manually allocate memory in After Effects effect plugins, or follow the sample plugins' approach?

You should not manually allocate memory in effect plugins. Instead, follow what the sample plugins are doing. Manual allocation is problematic because: (1) After Effects won't know whether to deallocate using delete[] or free(), (2) AE won't be able to track memory usage for optimization and management, and (3) it can lead to memory leaks or corruption. Let AE manage memory allocation and deallocation through its own APIs.

*Tags: `best-practices`, `effect-plugin`, `memory`, `reference`*

---

### Do I need to use PF_NewWorld to modify the output PF_LayerDef in the Render function, or can I directly adjust its data?

You should not modify the layer settings (width, height, rowbytes) directly as they are defined by the host app. Instead of directly assigning mat.data to layerDef->data, you should copy the data from your matrix into the layerDef->data buffer using a loop while respecting layerDef->rowbytes. The proper approach is demonstrated in the skeleton plugin, which uses the Iterate Suite to write into output->data rather than allocating output itself.

```cpp
static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        for (int x = 0; x < layer->width; ++x) {
            cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
            PF_Pixel& aePixel = *sampleIntegral32(*layer, x, y);
            aePixel.alpha = pixel[3];
            aePixel.red = pixel[2];
            aePixel.green = pixel[1];
            aePixel.blue = pixel[0];
        }
    }
}
```

*Tags: `aegp`, `layer-checkout`, `memory`, `output-rect`*

---

### How do I properly sample and write pixels to a PF_LayerDef respecting rowbytes alignment?

Use pointer arithmetic to calculate pixel addresses based on rowbytes. The formula is: (char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)). This ensures proper memory alignment. Reference: https://ae-plugins.docsforadobe.dev/effect-details/tips-tricks.html?highlight=center%20of%20a%20pixel#sampling-pixels-at-x-y

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data +
    (y * def.rowbytes) +
    (x * sizeof(PF_Pixel)));
}
```

*Tags: `aegp`, `memory`, `output-rect`, `reference`*

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

### How can you efficiently convert an After Effects layer to OpenCV Mat format while handling pixel format conversion and ROI?

When converting a PF_LayerDef to cv::Mat, use PF_GET_PIXEL_DATA8 to get the pixel data pointer, then create a Mat with the proper stride parameter (rowbytes). Use cv::mixChannels with a fromTo mapping array to handle channel reordering (e.g., ARGB to BGRA: fromTo[] = {0, 3, 1, 2, 2, 1, 3, 0}). The Mat constructor accepts a step parameter for row stride: cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes). This allows proper handling of extent_hint for ROI-based processing.

```cpp
static cv::Mat ConvertLayerToMat(PF_LayerDef* layerDef, PF_InData* in_data) {
    PF_Pixel8* pixelData = nullptr;
    PF_GET_PIXEL_DATA8(layerDef, NULL, &pixelData);
    int width = layerDef->width;
    int height = layerDef->height;
    cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
    cv::Rect roi(
        in_data->extent_hint.left - in_data->output_origin_x,
        in_data->extent_hint.top - in_data->output_origin_y,
        width, height
    );
    if (roi.x < 0 || roi.y < 0 || roi.x + roi.width > width || roi.y + roi.height > height) {
        throw std::runtime_error("ROI goes outside the image dimensions.");
    }
    cv::Mat argbRoi = argb(roi);
    cv::Mat bgra(height, width, CV_8UC4);
    int fromTo[] = {0, 3, 1, 2, 2, 1, 3, 0};
    cv::mixChannels(&argbRoi, 1, &bgra, 1, fromTo, 4);
    return bgra;
}
```

*Tags: `layer-checkout`, `memory`, `opencv`, `output-rect`, `pixel-format`*

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

### What is the OpenCV Mat step parameter and how does it work?

The step parameter in OpenCV's Mat constructor specifies the number of bytes between consecutive rows in the matrix. This is useful when working with data that has custom row alignment or stride, such as pixel buffers from After Effects. See: https://docs.opencv.org/4.x/d3/d63/classcv_1_1Mat.html#a51615ebf17a64c968df0bf49b4de6a3a

*Tags: `memory`, `opencv`, `reference`*

---

### What are the limitations of working with thousands of layers in After Effects, and how do plugins like Newton handle this?

After Effects struggles with performance even at 512 layers. The user is asking whether Newton (a physics simulation plugin) encounters the same scaling issues when simulating many layers, and considering whether switching to GPU-based sprite rendering would be necessary to maintain flexibility while handling 10,000 layer instances.

*Tags: `ae-limits`, `gpu`, `memory`, `performance`, `render-loop`*

---

### What are the common performance bottlenecks when working with large numbers of layers in After Effects plugins?

Performance issues with large layer counts (800+ layers) typically stem from UI rendering rather than the render engine itself. The bottleneck often comes from repeated After Effects API calls like 'get layer info' and 'get layer sprite', not from Artisan optimization. Additionally, After Effects has poor memory management with large layer counts, making it inefficient to handle thousands of layers.

*Tags: `aegp`, `debugging`, `memory`, `performance`, `ui`*

---

### How do you store a struct in global_data in an After Effects plugin?

You can store a struct in global_data by creating a pointer to it in the global_data structure, then in PF_GlobalSetup allocate it with `globaldata->structPointer = new StructName();`. While you can optionally delete it in PF_GlobalSetdown, this is not strictly necessary since the OS will clear the memory when After Effects closes.

```cpp
// In globaldata definition:
StructName* structPointer;

// In PF_GlobalSetup:
globaldata->structPointer = new StructName();

// Optional in PF_GlobalSetdown:
delete globaldata->structPointer;
```

*Tags: `aegp`, `memory`, `params`*

---

### Can you use smart pointers in After Effects global data objects?

No, you should not use smart pointers in global data objects. Instead, use new/malloc directly since After Effects does funky things with that pointer. You can do delete/free on the pointer during global setup/teardown, but it's optional.

*Tags: `aegp`, `memory`, `plugin-architecture`*

---

### Is it expected to receive thousands of arb dispose calls when shutting down After Effects?

Yes, this is expected behavior. After Effects' allocators don't dispose of resources until available memory is saturated or on thread destruction/exit. You can test this behavior by artificially setting the available memory to a low value like 2GB, which will trigger more destruction calls during runtime.

*Tags: `aegp`, `arb-data`, `debugging`, `memory`*

---

### What is the purpose of the rowbytes field in layer data structures?

The rowbytes field exists as an optimization to allow layers to be cropped without allocating extra memory. A layer can be cropped by simply updating width and height and making the data pointer reference the new top-left pixel, while leaving rowbytes untouched. After Effects uses this technique frequently, for example when dragging a layer partially off the composition.

*Tags: `layer-checkout`, `memory`, `optimization`*

---

### What is the purpose of rowbytes in After Effects layer data, and how does it enable memory optimization?

Rowbytes exists as an optimization so that layers can be cropped without allocating extra memory. You can crop a layer by updating width and height and making the data pointer reference the new top-left pixel, while leaving rowbytes unchanged and maintaining a reference to the original data for proper deallocation. After Effects uses this optimization frequently, such as when dragging a layer partially off the composition. Rowbytes is calculated as 4*width for 32-bit data, but may be larger if a layer is loaded and moved partially off-comp; however, purging cache typically resets it back to 4*width.

*Tags: `caching`, `memory`, `optimization`, `output-rect`*

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

### Why do lock/unlock functions still exist in the After Effects SDK if they weren't ported to 64-bit?

According to Tobias Fleischer, the lock/unlock functions were not ported when the AE codebase moved to 64-bit in CS6 (2011), yet they still exist in the current SDK and sample code calls them. While rowbyte notes these functions are somewhat redundant in a 64-bit address space, they still provide a safe way to dereference handles (which are effectively double pointers in 64-bit) without confusion. The SDK provides a DH macro as an alternative for direct dereferencing.

*Tags: `64-bit`, `aegp`, `memory`, `reference`, `sdk`*

---

### Does the old GLator sample have memory leaks in its GL rendering code?

Yes, the GLator sample has significant memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, which means bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBOs.

*Tags: `deprecated`, `gpu`, `memory`, `opengl`, `reference`*

---

### What is a good resource for understanding scope guards and their implementation in C++?

Alex Bizeau from maxon mentioned that scope guards are smart pointers for anything that can use lambda functions. They're valuable memory safety tools because they handle deletion automatically on throwing and return statements without requiring explicit deletion handling at every return point. Maxon has their own implementation, with examples available in their codebase.

*Tags: `debugging`, `memory`, `open-source`, `reference`, `tool`*

---

### What is a good tool for managing resource cleanup and memory safety in C++ plugin code?

Scope guards are a useful pattern for memory safety in C++ code. They act like smart pointers for any resource, allowing you to use lambda functions to handle cleanup on scope exit. This is particularly valuable for handling exceptions and early returns without explicitly managing deletion at every exit point. Alex Bizeau from maxon recommended the scope_guard library as a reference implementation: https://github.com/ricab/scope_guard/blob/main/scope_guard.hpp. Scope guards work well with both smart pointers and even malloc/free patterns, making them a favorite tool for resource management.

*Tags: `cpp`, `memory`, `reference`, `resource-management`, `smart-pointer`, `tool`*

---

### What causes error code 1397908844 when rendering with multiple effects in After Effects?

According to an Adobe Community post (https://community.adobe.com/t5/after-effects-discussions/error-while-rendering-with-multiple-effects-with-code-1397908844-using-pf-newworld-pf-disposeworld/td-p/14285656), this error occurs when a plugin is accidentally double-releasing a suite. The error has been documented since at least 2007 and can also occur in other Adobe applications like Illustrator.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

### What causes the generic exception handler/catch-all error in Adobe After Effects, and why does it appear more frequently in newer versions?

The generic exception handler/catch-all in the Adobe suites manager is triggered by issues like double freeing or releasing a suite pointer. This error has existed since CS2 or CS3 days. It appears more often in newer AE versions due to Multi-Frame Rendering (MFR) with global suite handles or suite pointers being shared between threads, which increases the likelihood of memory management conflicts.

*Tags: `aegp`, `memory`, `mfr`, `threading`*

---

### What is a modern approach to error handling in After Effects SDK development?

Use std::expected for function returns instead of traditional error codes. This allows for easier error handling and proper chaining with .then() methods. Create type aliases like ExpectedAEGP and ExpectedPF to wrap the expected types for different AE APIs. Additionally, wrap all AE handles in classes with proper construction and destruction in smart pointers, and implement an allocation class manager that clears allocations when exiting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug builds, implement a cache validator to detect extra memory still allocated at endpoints.

```cpp
fn get_thing() -> Result<Thing, Error>;
```

*Tags: `aegp`, `debugging`, `error handling`, `memory`, `resource management`, `smart pointers`*

---

### What best practices should be followed for managing After Effects handles in C++ plugins?

Wrap all AE handles into classes with proper construction and destruction in destructors, and use smart pointers to manage them. Additionally, implement a greater allocation class manager that clears allocations when quitting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug mode, use a cache validator to verify no extra memory remains allocated at endpoints.

*Tags: `aegp`, `best-practices`, `debugging`, `memory`, `smart-pointers`*

---

### Why does a blur function work differently between 8-bit and 16/32-bit color depths in After Effects?

When implementing blur effects that support multiple bit depths (8-bit, 16-bit, and 32-bit), subtle differences in how path values are converted between depth formats can cause discrepancies. One workaround is to precompose the layer with the effect first, which ensures consistent processing. The issue likely stems from improper conversion of path values when switching between 8-bit and 16/32-bit processing paths.

*Tags: `debugging`, `memory`, `mfr`, `render-loop`*

---

### What were the root causes of bugs in the FastBoxBlur implementation?

Two critical bugs were identified in FastBoxBlur: (1) Incorrect bit depth detection causing memory corruption, and (2) Improper stride handling for padded buffers. The solution involved fixing bit depth detection to use rowbytes calculation and implementing proper stride handling throughout the blur algorithm.

*Tags: `debugging`, `memory`, `output-rect`, `render-loop`*

---

### How can a plugin effect communicate implicit dependencies on other layers and effects to After Effects' caching system?

When building multi-layer effects that discover and depend on other layers' properties at runtime, you need to explicitly communicate these dependencies to AE's render dependency tracking system. This involves checking out the layers you depend on and notifying AE of those checkouts so the cache knows to invalidate when those layers change. The renderer effect should enumerate dependent layers, check out their pixels/masks/shapes/transforms, and use the appropriate AEGP calls to register these dependencies so AE's smart render and disk cache systems understand the effect needs to re-render when those layers are modified.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `memory`, `smart-render`*

---

### Is it possible to create a multi-layer global illumination plugin that discovers material metadata effects on other layers and properly communicates dependencies to After Effects' caching system?

This is a complex architectural question posed by fad regarding whether a 'renderer' effect can enumerate other layers, discover per-layer material definition effects, check out layer pixels/masks/transforms, and have those dependencies properly tracked by AE's dependency tracking and disk cache system. While Tobias Fleischer's experience suggests multi-layer effects are viable, the specific mechanism for communicating hidden dependencies (non-explicit UI parameter dependencies) to AE's caching system to trigger rerenders when materials change remains an open design challenge requiring careful SDK integration and likely custom dependency management.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `memory`, `render-loop`, `smartfx`*

---

### How can you efficiently cache data when scraping parameters from dummy effects on multiple layers?

When implementing a control scheme using layer parameters on the main effect with dummy effects on other layers, call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects. This ensures After Effects caches everything properly and maintains good performance even when reading all layers one by one.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`, `params`*

---

### How can you synchronize layer data changes in an After Effects plugin using AEGP?

According to Alex Bizeau from maxon, you need to periodically check the project timestamp for changes, then gather your layers and check children for potential updates. This approach is computationally expensive but is noted as the proper synchronization alternative available with AEGP.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`*

---

### Is PF_LayerDef::rowbytes always guaranteed to be a multiple of 4?

This question was asked but not answered in the conversation. No response or clarification was provided.

*Tags: `aegp`, `memory`, `reference`*

---

### What alignment guarantees does PF_LayerDef::rowbytes provide in After Effects?

PF_LayerDef::rowbytes is guaranteed to always be a multiple of sizeof(PF_Pixel8), which is 4 bytes. 16-byte alignment is not always guaranteed. The rowbytes value is arbitrary and your code should be prepared to deal with any stride. Additionally, a buffer might be a sub-reference to another buffer, so you should not write into the bytes outside your image reference frame into the rowbytes gutter.

*Tags: `aegp`, `memory`, `output-rect`, `params`*

---

### Why is rowbytes alignment important for pointer dereferencing in After Effects plugins?

Dereferencing a pointer that is not correctly aligned for the referenced type results in undefined behavior according to the C specification (6.3.2.3 7 in the C spec https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3047.pdf). If rowbytes was not a multiple of alignof(Pixel Type), you would be dereferencing misaligned pointers everywhere, which is UB. Apple documents this issue at https://developer.apple.com/documentation/xcode/misaligned-pointer.

*Tags: `aegp`, `debugging`, `macos`, `memory`*

---

### How do you efficiently copy pixel data row-by-row between Premiere and Vulkan/OpenGL APIs?

In Premiere, use memcpy with row-based copying by calculating pointers for each row: `void* PrPtr = (char*)PRData + y*PrRowByte; void* ApiPtr = (char*)ApiData + y*APIRowByte; memcpy(ApiPtr, PrPtr, APIRowByte);` where APIRowByte is calculated as `width * 4 * sizeOfPixel` (1 for 8-bit, 4 for 32-bit). PrRowByte comes from Premiere and may be negative. This approach works for reading from Premiere, but copying back to Premiere can cause crashes. For Vulkan specifically, use `vkGetImageSubresourceLayout()` to get the actual `VkSubresourceLayout` of source/destination images to ensure correct copying, as the layout may not always be packed. In After Effects, row-byte already includes 16-byte SIMD padding, so verify buffer alignment and that API pointers are correctly mapped to avoid access violations.

```cpp
void* PrPtr = (char*)PRData + y*PrRowByte;
void* ApiPtr = (char*)ApiData + y*APIRowByte;
memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `gpu`, `memory`, `opengl`, `premiere`, `vulkan`*

---

### How can you iterate through pixel data when working with rowbytes in After Effects or Premiere?

You can iterate through pixel data yourself using rowbytes. Be careful with negative values in Premiere, as they can occur and need special handling.

*Tags: `caching`, `memory`, `mfr`, `premiere`*

---

### How can you retrieve the original parameter value with full precision before downsampling is applied by in_data->downsample_x and downsample_y?

Use AEGP_GetNewStreamValue instead of the plain checkout method to retrieve parameter values. This approach preserves the original precision of the parameter value before any downsampling by in_data->downsample_x and in_data->downsample_y is applied, which is critical for maintaining double precision in 3D point parameters.

*Tags: `aegp`, `memory`, `params`*

---

### Can you use transform_world multiple times in a loop with the same output buffer as input for subsequent transformations?

No, you cannot use the same buffer as both input and output for transform_world, as it reads and overwrites simultaneously, resulting in corrupted output. Instead, create a temporary buffer and alternate between buffers: transform input to temp, temp to output, output back to temp, repeating as needed. Never overwrite the original input buffer that After Effects caches, as this causes bizarre bugs.

```cpp
// Correct approach for repeated transformations
// 1. create a new buffer "temp"
// 2. transform the input to temp
ERR(in_data->utils->transform_world(in_data->effect_ref, in_data->quality, PF_MF_Alpha_STRAIGHT, in_data->field, input, &composite_mode, NULL, &matrix1, 1L, TRUE, &output->extent_hint, temp));
// 3. transform temp to output
ERR(in_data->utils->transform_world(in_data->effect_ref, in_data->quality, PF_MF_Alpha_STRAIGHT, in_data->field, temp, &composite_mode, NULL, &matrix2, 1L, TRUE, &output->extent_hint, output));
// 4. repeat by transforming output back to temp for next iteration
```

*Tags: `aegp`, `debugging`, `memory`, `render-loop`, `transform_world`*

---

### What is the recommended way to store global data like arrays or vectors in After Effects plugins?

While global variables are technically allowed and commonly used, the best practice for After Effects plugins is to allocate memory and store it in a pointer on your global_data handle. This approach is more RAM-friendly and respects AE's memory allocation and prioritization. Avoid using std::vector directly since it uses system malloc/new rather than AE's memory management. If you must use std::vector, write a custom allocator that uses AE's memory allocation functions. Global variables should only be used for constant data that doesn't change once set, and be aware that such memory persists until AE shuts down and cannot be properly released after global_setdown.

*Tags: `aegp`, `global-state`, `memory`*

---

### Why are global variables considered bad practice in plugin development?

Global variables are discouraged for several reasons: (1) They reduce code readability because function behavior depends on data outside their parameters; (2) They can cause CPU cache inefficiency as the processor must fetch data from different memory segments; (3) In multi-threaded scenarios, they require mutexes which impose performance penalties and make code non-thread-safe; (4) They are generally considered poor programming practice. However, they are widely used in practice, especially for constant data that is set once and never changes.

*Tags: `debugging`, `memory`, `threading`*

---

### How can I predict the required buffer size for string conversions in Windows?

Use the Windows API function MultiByteToWideChar with a null pointer for the output buffer parameter. This will return the required buffer size in wide characters without actually performing the conversion. This allows you to accurately allocate the necessary buffer before conversion.

*Tags: `memory`, `sdk`, `string-handling`, `windows`*

---

### What are the memory safety considerations when using AEGP_RenderAndCheckoutLayerFrame_Async?

According to community discussion with Adobe's AE team, there is a known memory release problem when using the async version of renderAndCheckout. As a result, developers have reverted to using the synchronous version instead, rendering one frame at a time on each idle cycle to avoid UI lag while properly managing memory.

*Tags: `aegp`, `async`, `memory`, `render-loop`, `threading`*

---

### How can you safely implement async layer frame rendering in an AEGP plugin?

Create an AsyncManager class that wraps AEGP_RenderAndCheckoutLayerFrame_Async with proper callback handling. The async call is thread-safe as long as only one thread calls it at a time. Use a static callback function that receives the request via refcon, retrieves the world from the frame receipt, and invokes a user-provided callback function. Store the callback as a heap-allocated std::function pointer passed through the refcon parameter, then delete it after use to prevent memory leaks.

```cpp
class AsyncRenderManager {
public:
  void renderAsync(LayerRenderOptionsPtr optionsH, std::function<void(WorldPtr)> callbackF) {
    auto callbackPtr = new std::function<void(WorldPtr)>(callbackF);
    auto ret = std::async(std::launch::async, [optionsH, callbackPtr]() {
      RenderSuite().renderAndCheckoutLayerFrameAsync(optionsH, callback,
        reinterpret_cast<AEGP_AsyncFrameRequestRefcon>(callbackPtr));
    });
  }
  
  static A_Err callback(AEGP_AsyncRequestId request_id, A_Boolean was_canceled, A_Err error,
    AEGP_FrameReceiptH receiptH, AEGP_AsyncFrameRequestRefcon refconP0) {
    auto callbackPtr = reinterpret_cast<std::function<void(WorldPtr)> *>(refconP0);
    if (callbackPtr && *callbackPtr) {
      FrameReceiptPtr ptr = std::make_shared<AEGP_FrameReceiptH>(receiptH);
      auto world = RenderSuite().getReceiptWorld(ptr);
      (*callbackPtr)(world);
      delete callbackPtr;
    }
    return error;
  }
};
```

*Tags: `aegp`, `async`, `memory`, `render-loop`, `threading`*

---

### How can I setup an After Effects plugin to asynchronously receive video frames from an external pipe?

You can setup a separate C++ thread (independent of the AE SDK) to handle reading/writing data to a pre-allocated memory buffer. Use a mutex to safely access this memory from both the async pipe thread and AE's render calls. To trigger the async pipe thread to process data promptly, use AEGP_CauseIdleRoutinesToBeCalled(). However, reconciling AE's random frame access model with sequential async pipe operations is challenging—you may need to either stall the render until required frames are available or cache frames to a file and fetch them on demand. For fetching source frames without blocking the UI, use AEGP_CheckoutOrRender_ItemFrame_AsyncManager or AEGP_CheckoutOrRender_LayerFrame_AsyncManager.

*Tags: `aegp`, `async`, `caching`, `memory`, `threading`*

---

### Should I use global variables to store custom parameter data that needs to be accessed from multiple functions?

No. Instead of using global variables, store user input in a LOCAL array during UI interactions (like DrawEvent), then serialize that local array into an arbitrary parameter. This allows the data to persist, be invalidated properly by AE, and trigger re-renders automatically without relying on global state.

*Tags: `arb-data`, `memory`, `params`, `ui`*

---

### Can I save sequence data during SequenceSetdown?

No. SequenceSetdown is AE's mechanism for telling your plugin that data needs to be destructed and freed. The data saved with the project is the last handle used or flattened (depending on whether you have set the flattening flag). SequenceSetdown is not the appropriate place to save data, as the logic is that the data may not just be a simple handle to be freed, but rather a structure with pointers that need to be separately destructed and freed.

*Tags: `aegp`, `memory`, `sdk`, `sequence-data`*

---

### How can I create an After Effects effect that only provides settings without changing the layer visually?

There are several approaches: (1) Use AE's utils->Copy() function to copy the input buffer to the output buffer, which is fast and straightforward. (2) Set the PF_OutFlag_AUDIO_EFFECT_ONLY flag to prevent AE from calling your effect for image rendering—it will only be called for audio, which you must implement by copying input to output. This is faster than copying the image buffer. (3) Try setting the output rect size to 0 or negative during pre-render to make AE skip rendering, though this approach has inconsistent results. The simplest approach is using utils->Copy() to pass through the input unchanged.

```cpp
static PF_Err
Render(	PF_InData		*in_data,
		PF_OutData		*out_data,
		PF_ParamDef		*params[],
		PF_LayerDef		*output)
{
	PF_Err	err	= PF_Err_NONE;
	// Use utils->Copy() to copy input to output
	return err;
}
```

*Tags: `aegp`, `memory`, `output-rect`, `render-loop`*

---

### Why does setting PF_OutFlag_NON_PARAM_VARY cause memory usage to increase dramatically?

Setting PF_OutFlag_NON_PARAM_VARY or PF_OutFlag_WIDE_TIME_INPUT causes After Effects to call FrameSetup>Render>FrameSetDown for each frame instead of once, which can appear to increase memory usage significantly. However, this is often normal behavior where After Effects is caching frames. The memory will be released when purging AE's memory cache or when the memory is needed for new frame renders. Ensure you implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup, preventing accumulation across multiple frame renders.

*Tags: `aegp`, `caching`, `memory`, `render-loop`*

---

### How should I handle memory allocation when rendering multiple frames with NON_PARAM_VARY flag?

When using PF_OutFlag_NON_PARAM_VARY to render multiple frames, you must implement PF_Cmd_FRAME_SETDOWN to properly free memory allocated during PF_Cmd_FRAME_SETUP. This prevents memory accumulation as FrameSetup and FrameSetDown are called for each frame. Without implementing FRAME_SETDOWN, allocated memory from each frame setup will accumulate rather than being released between frames.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

### What is the correct way to extract arbitrary data from a stream value in After Effects plugins?

The correct approach is to cast the stream value's arbH handle to a PF_Handle, then use the HandleSuite1 to lock the handle and access the arbitrary data. However, it's critical to ensure you either complete all data operations before releasing the stream value, or copy the data before release. Otherwise, you may be left with an invalidated pointer that could cause crashes or undefined behavior. The stream value should be disposed with AEGP_DisposeStreamValue after you're done with it.

```cpp
PF_Handle arbH = reinterpret_cast<PF_Handle>(streamVal.val.arbH);
arbData *arbP = reinterpret_cast<arbDataP>(suites.HandleSuite1()->host_lock_handle(arbH));
// Use arbP data here before disposing
suites.StreamSuite5()->AEGP_DisposeStreamValue(&streamVal);
```

*Tags: `aegp`, `arb-data`, `memory`, `stream-data`*

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

### What is a good library for saving EffectWorld pixel data to image files in AE plugins?

tinypng is a popular library for saving pixel buffer data to image files when working with AE plugin EffectWorld objects. You can access the pixel data via effect_worldP->data and then use tinypng to serialize it to disk.

*Tags: `memory`, `open-source`, `reference`, `tool`*

---

### Why isn't drawing to a newly created Effect World displaying correctly when using transform_world?

When drawing pixel-by-pixel to a newly created Effect World using manual pixel manipulation, ensure you are drawing directly to the output parameter rather than an intermediate world buffer. The issue in this case was a logic error in the broader code context, not the pixel-drawing approach itself. The technique of using sampleIntegral32() to get a pointer to a pixel and then modifying its RGBA values is correct, but verify the world is being used in the right scope and that PF_Fill works as a control test to confirm the world itself is valid.

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

static PF_Err drawPixel32(int x, int y, int size, PF_EffectWorld *output) {
  PF_Err err = PF_Err_NONE;
  int u, v;
  for (u=0; u < size; u++) {
    for (v=0; v < size; v++) {
      PF_Pixel *myPixel = sampleIntegral32(*output, x+u, y+v);
      myPixel->red = 0;
      myPixel->green = 0;
      myPixel->blue = 255;
      myPixel->alpha = 255;
    }
  }
  return err;
}
```

*Tags: `aegp`, `debugging`, `memory`, `output-rect`*

---

### Why doesn't changing a plugin parameter invalidate cached frames in After Effects?

This is actually normal After Effects behavior. When a parameter changes, the composition render cache is invalidated and the displayed result is correct. However, After Effects intentionally does not release the RAM of previously cached frames—it keeps old results in memory in case the user undoes changes, until RAM is needed for something else. To verify this, purge After Effects' RAM and the memory usage will drop back to the original level, confirming the RAM accumulation is AE's design and not a memory leak.

*Tags: `caching`, `memory`, `params`, `render-loop`*

---

### What causes an access violation error when calling AEGP_GetActiveItem() in a C++ plugin?

An access violation when calling AEGP_GetActiveItem() is most likely caused by an invalid in_data pointer or an invalid pica_basicP passed to the AEGP_SuiteHandler. Ensure that the function is being called during a valid AE command execution where in_data is guaranteed to be valid, and verify that the passed in_data pointer has not been corrupted or used outside its valid scope.

*Tags: `aegp`, `debugging`, `memory`*

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

### Why does AEGP_GetLayerToWorldXform return zeros when called from multiple threads using Iterate8Suite1?

Most AEGP functions are not thread-safe. AEGP_GetLayerToWorldXform should only be called from the main thread that After Effects called your plug-in with. Calling it from other threads will return zeros or cause internal verification failures. Only a few AEGP functions are documented as thread-safe.

*Tags: `aegp`, `memory`, `sdk`, `threading`*

---

### Why does a plugin's blur effect keep increasing when changing iteration parameters?

The plugin was writing into the input layer buffer and corrupting it. After Effects provides the original input buffer, not a copy that can be modified. When iterating the blur in a loop and changing the iteration parameter, the corrupted input from the previous operation causes cumulative blur effects. The issue stems from assigning the input layer directly (PF_EffectWorld tmpWrld = params[0]->u.ld) which creates a reference rather than a true copy.

```cpp
// Incorrect - creates reference:
PF_EffectWorld tmpWrld = params[0]->u.ld;

// Correct approach - allocate new buffer:
// AEGP_WorldSuite3->AEGP_New()
// AEGP_FillOutPFEffectWorld()  // creates a PF_EffectWorld wrapper for the allocated AEGP_WorldH
```

*Tags: `aegp`, `memory`, `params`, `render-loop`*

---

### How do you create a proper copy of an input buffer in an After Effects plugin instead of a reference?

Use AEGP_WorldSuite3->AEGP_New() to allocate a new world buffer, then call AEGP_FillOutPFEffectWorld() to create a PF_EffectWorld wrapper for the allocated AEGP_WorldH. This prevents the common mistake of assigning the input layer directly (which only creates a reference) and corrupting the original input data when iterating operations.

*Tags: `aegp`, `buffer-allocation`, `memory`*

---

### How should arbitrary data reallocation be handled when it depends on non-animatable parameters like 'rows' and 'cols' in a C++ effect plugin?

Arbitrary parameter handlers in After Effects are stateless functions that operate without access to a specific effect instance, effect reference, or sequence data. Instead of trying to fetch parameter values during HandleArbitrary() calls (where effect_ref is usually null and checkout/checkin rarely work), you should reallocate and store allocation-related metadata like 'rows' and 'cols' in your arbitrary data structure itself during UserChangedParam() or the render function. This way, HandleArbitrary() can process the data independently without querying system or instance state. The render function should fetch raw arbitrary data and other relevant pieces available during render time, then process them there rather than during arbitrary parameter handling calls.

*Tags: `arb-data`, `memory`, `params`, `skeleton`*

---

### How do you prevent video memory leaks when using OpenGL in After Effects plugins, especially during UI parameter changes?

The issue is typically caused by exceptions thrown within OpenGL calls during an interrupt, which causes the code to skip glDeleteTextures calls and jump to catch blocks. To fix this, ensure that glDeleteTextures is called before checking for interrupts using PF_ABORT(), and put breakpoints in catch blocks to verify if exceptions are being thrown during user interaction. The proper order should be: perform GPU operations, delete textures, unbind framebuffers and textures, then check for abort conditions and release suites.

```cpp
glBindFramebuffer(GL_FRAMEBUFFER, 0);
glBindTexture(GL_TEXTURE_2D, 0);
glDeleteTextures(1, &inputFrameTexture);
glDeleteTextures(1, &inputFrameTexture2);
// Then check for abort after cleanup
ERR(PF_ABORT(in_data));
```

*Tags: `debugging`, `gpu`, `memory`, `opengl`, `plugin-development`*

---

### How should you handle frame interruption in After Effects plugins to avoid crashes when advancing frames quickly?

When a user interrupts frame processing (by advancing to another frame, clicking buttons, etc.), After Effects does not forcefully kill the rendering thread. Instead, the plugin must call PF_ABORT() during the render call (not pre-render) to check if AE wants processing to stop. If PF_ABORT returns true, you should return a PF_Interrupt_CANCEL error message to tell AE the output buffer should not be used. Crashes during interruption are often caused by re-entrancy issues—if your crash occurs when accessing shared data structures like matrices, ensure thread-safety by using mutexes or allocating structures on the stack rather than globally. Call PF_ABORT periodically during lengthy processing to maintain responsive interactivity while scrubbing or changing parameters.

```cpp
if (err = PF_ABORT(in_dataP)) {
  err = PF_Interrupt_CANCEL;
  return err;  // Must return after setting the error
}
```

*Tags: `aegp`, `debugging`, `memory`, `render-loop`, `threading`*

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

### How do you copy pixel data from a loaded BMP file into a PF_EffectWorld?

Once you have loaded the BMP pixel data and allocated an effect world, you can access the effect world's pixel data using pointer arithmetic: PF_Pixel *p = world->data + x + y * (world->rowbytes / sizeof(PF_Pixel)). Then copy individual color channels from the BMP data to the effect world pixel, such as p->red = bmpPixel[0], p->green = bmpPixel[1], p->blue = bmpPixel[2], and p->alpha = 255. Be aware that the channel order and bit depth may differ between the BMP format and the effect world format, so value conversion may be required.

```cpp
PF_Pixel *p = world->data + x + y * (world->rowbytes / sizeof(PF_Pixel));
p->red = bmpPixel[0];
p->green = bmpPixel[1];
p->blue = bmpPixel[2];
p->alpha = 255;
```

*Tags: `aegp`, `memory`, `mfr`, `output-rect`*

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

### How can I create off-screen image buffers in After Effects SDK and draw to them?

It is possible to create off-screen image buffers in After Effects SDK. You can use intermediate buffers by working with the output buffer handed to the effect. The buffer has a 'data' pointer for the base address of image pixels and a 'rowbytes' variable for the number of bytes per row (stride). You can draw to custom buffers and then copy the result into the output buffer. After Effects provides native buffers called 'worlds' in the SDK documentation that can be used as intermediates.

*Tags: `memory`, `output-rect`, `render-loop`, `sdk`*

---

### Are effect_ref and AEGP_EffectRefH the same thing and interchangeable?

No, effect_ref and AEGP_EffectRefH are not interchangeable. While they may have similar values because both denote RAM addresses, they hold different kinds of data. The effect_ref is used for very few things like GetNewEffectForEffect(), while AEGP_EffectRefH is used for a wide range of AEGP callbacks. Additionally, you cannot rely on the validity of either beyond the scope of a single AE call—after a call ends, AE may change its dataset and invalidate the references. To safely use AEGP functions from an idle hook, store your effect's index on the layer, layer ID, and comp item ID instead, then look up a fresh AEGP_EffectRefH during each idle hook call.

*Tags: `aegp`, `memory`, `params`, `threading`*

---

### Why is arbitrary data always showing default values in PF_cmd_Render and PF_cmd_SmartRender, but correct values in PF_cmd_User_changed_params?

The issue is typically in how the arbitrary data handle is being allocated and passed back to After Effects. Arbitrary data should be flat—the entire data should be one big handle, not a handle containing other handles. When allocating new arbitrary data, create a single handle of the required size (data size + null termination), copy the data into it directly, and pass it to AE. When you pass the handle as an arb value, it becomes owned by After Effects, so do not unlock or release it. Also, use AEGP_GetNewStreamValue instead of PF_checkout_param to retrieve parameter values.

```cpp
PF_Handle newArbH = PF_NEW_HANDLE(newString.length() + 1);
A_char* newExpMemP = (char*)newArbH;
strcpy(newExpMemP, newString.c_str());
params[SKELETON_ARBITRARY_EXPRESSION]->u.arb_d.value = newArbH;
// Do not unlock/release - handle now belongs to AE
```

*Tags: `arb-data`, `memory`, `params`, `render-loop`*

---

### How can I debug intermittent C++ exceptions that occur between PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_RENDER?

According to community expert shachar carmi, intermittent exceptions at the same memory address are likely caused by stack overflow or buffer overrun. The recommended debugging strategy is to use binary search: disable half your code and test to see if the problem manifests, then progressively narrow down the suspect code by testing the remaining halves. This methodical approach helps isolate the problematic section without relying on traditional debuggers, which may not be effective for these types of memory issues.

*Tags: `debugging`, `memory`, `render-loop`, `windows`*

---

### Why does AEGP_RenderAndCheckoutLayerFrame cause a hard crash when trying to get layer pixels?

The issue was caused by incorrect pointer declaration. Instead of declaring AEGP_FrameReceiptH *receiptPH (pointer to handle), declare it as AEGP_FrameReceiptH receiptPH (handle itself) and pass it as &receiptPH to the function. Additionally, pass null values for the cancel function parameters instead of uninitialized variables.

```cpp
AEGP_FrameReceiptH receiptPH;
ERR(suites.RenderSuite5()->AEGP_RenderAndCheckoutLayerFrame(optionsH, NULL, NULL, &receiptPH));
ERR(suites.RenderSuite4()->AEGP_GetReceiptWorld(receiptPH, &worldH));
```

*Tags: `aegp`, `debugging`, `layer-checkout`, `memory`*

---

### Why does my After Effects plugin cause memory to balloon to 4-5 GB when changing parameters?

This can happen for two main reasons: (1) After Effects caching images as a user preference, which generally shouldn't be a concern, or (2) your plugin uses PF_NewWorld, which in some AE versions doesn't free memory even when PF_DisposeWorld is called. The memory is only freed when the user manually purges all caches through the menu. To fix this, consider allocating memory through the memory suite instead of PF_NewWorld, and release it after processing. Alternatively, you can point a PF_EffectWorld's 'data' parameter to arbitrarily allocated memory by filling in the other relevant data fields.

*Tags: `aegp`, `caching`, `debugging`, `memory`, `pf_newworld`*

---

### How can I efficiently access neighboring pixel values during CCU iteration without using sampleIntegral32 on every pixel?

You can use pointer math to access adjacent pixels directly from the input buffer pointer. If inP is a pointer to the current pixel, you can get the next pixel on the same line with `PF_Pixel *next = inP + 1;`. However, for better performance when needing multiple integer samples of other pixels, use iterate_generic instead of the regular iteration suites, as it allows much more optimization potential than acquiring and releasing the sample suite on every pixel.

```cpp
PF_Pixel *next = inP + 1; // gives you the next pixel on the same line
```

*Tags: `aegp`, `ccu`, `iteration`, `memory`, `mfr`, `performance`*

---

### What is the performance impact of acquiring and releasing the sample suite on every pixel during iteration?

Acquiring and releasing the sample suite on every pixel significantly weighs down performance. For optimal results when you need to sample many adjacent pixels, it's recommended to access the input and output buffers directly and use iterate_generic instead, which allows each thread to handle a single buffer line and eliminates the overhead of repeated suite acquisition/release operations.

*Tags: `ccu`, `memory`, `optimization`, `performance`, `threading`*

---

### How can data be shared between render and UI threads if sequence_data synchronization is not reliable?

Global data is shared between threads and can be used as an alternative solution for render-to-UI-thread communication when sequence_data synchronization is not reliable. However, relying on updating UI sequence_data from the render function is not supported by design in CC 2015 and later versions.

*Tags: `memory`, `render-loop`, `threading`, `ui`*

---

### How should arbitrary data parameters be saved and loaded in After Effects plugins?

When dealing with an ARB (arbitrary) parameter, you should honor the PF_Arbitrary_UNFLATTEN_FUNC and PF_Arbitrary_FLATTEN_FUNC callbacks. The sequence data commands (PF_Cmd_SEQUENCE_FLATTEN and PF_Cmd_SEQUENCE_RESETUP) have nothing to do with ARB parameter persistence. ARB data may get flattened/unflattened during various events such as copy/paste, not only during save/load. If data doesn't reconstruct correctly after honoring these callbacks, the issue is likely with how the flatten/unflatten operations are implemented.

*Tags: `aegp`, `arb-data`, `memory`, `params`*

---

### Why does data passed via AEGP_IdleRefcon become corrupted when retrieved in the idle hook?

The issue occurs when you save the address of a locally-defined variable in the entry point function and cast it to AEGP_IdleRefcon. Once the function scope ends, that stack memory becomes invalid and may be reused by other code, causing the data to appear corrupted when accessed later in the idle hook. Solution: Use dynamic allocation (new/delete or preferably AE's memory suite) to allocate the data structure on the heap, which remains valid until explicitly freed. Always deallocate the memory in global setdown.

```cpp
// Wrong - local variable on stack:
my_global_data myStruct;
myStruct.my_int = 5;
my_global_dataP globP = &myStruct;  // invalid after function ends

// Correct - heap allocation:
my_global_dataP globP = new my_global_data();
globP->my_int = 5;
AEGP_IdleRefcon idleRefcon = reinterpret_cast<AEGP_IdleRefcon>(globP);
// Later in global setdown:
delete globP;
```

*Tags: `aegp`, `debugging`, `memory`, `refcon`*

---

### How should plugins handle non-thread-safe code in the render function?

While there is no way to disable multithreading entirely, if your code is not thread-safe, you can use a mutex at the top of your rendering function to serialize access and allow only one thread at a time to execute the critical section. However, this approach significantly reduces performance benefits of multithreading. The better solution is to refactor your plugin to make the render function properly thread-safe and multi-threadable.

*Tags: `memory`, `render-loop`, `threading`*

---

### How can you exchange data between an effect plugin and an IdleHook callback function?

While an IdleHook can reside in an effect plugin, the callback doesn't have direct access to the effect's in_data structure or parameters. The recommended approach is to allocate and lock a shared piece of RAM during global setup, store it in the global data structure, and pass it as the IdleRefCon to AEGP_RegisterIdleHook. AE will then pass this shared memory to the hook handling function on each idle call. Since this RAM is shared between all instances and threads, use a mutex to prevent race conditions. During idle calls, you can read/write parameter values using AEGP_GetNewStreamValue(), passing the instance reference data through the shared RAM.

*Tags: `aegp`, `memory`, `params`, `threading`*

---

### Why does my After Effects plugin display a white screen after parameter changes, requiring a purge to render correctly?

This is typically a buffer clearing issue. You need to fill the output buffer with empty pixels using PF_FillMatteSuite2()->fill(). Another common cause is accidentally writing data to the input buffer and then using it as an intermediate processing buffer, which causes After Effects to cache the tampered data. Always clear AEGP worlds before using them, as After Effects may reuse the same RAM block from a previous world call, which could contain junk data or previous frame data.

```cpp
PF_FillMatteSuite2()->fill()
```

*Tags: `aegp`, `caching`, `memory`, `output-rect`, `smart-render`*

---

### Should I use AEGP_World for floating-point processing in After Effects plugins?

It is not necessary to use AEGP_World for 32-bit floating-point processing. Even in non-smart plugins that only support 8 and 16-bit, you can create a 32-bit PF_EffectWorld internally for processing in floating point and avoid the complexity and potential issues associated with AEGP worlds.

*Tags: `aegp`, `memory`, `smart-render`*

---

### Can you keep AEGP handles like AEGP_ItemH, AEGP_CompH, and AEGP_LayerH across multiple plugin calls?

No, you cannot store these handles across plugin calls. AEGP_ItemH, AEGP_CompH, AEGP_LayerH, AEGP_StreamRefH, and AEGP_EffectRefH are only valid during a single call from After Effects to your effect. Once your plugin returns and AE regains control, these handles may become invalid. To track items across calls, store the item ID instead and search for it again on the next call. The same applies for layer handles using layer IDs.

*Tags: `aegp`, `memory`, `reference`*

---

### How should you debug a plugin that stops rendering prematurely when After Effects runs out of memory?

When a plugin doesn't properly handle out-of-memory conditions, first clarify whether it crashes or just stops rendering. If you cannot reproduce the problem in a debug session, write data to file from strategic points in the code and reproduce the error in release mode. This file-based logging approach will help pinpoint where memory is not being freed properly.

*Tags: `ae-plugin`, `debugging`, `memory`*

---

### Why might an After Effects plugin continue increasing memory usage instead of freeing memory after each frame?

When a plugin fails to use After Effects' memory suites correctly, it may not properly deallocate memory after processing frames. This causes memory usage to accumulate across multiple frames during preview or rendering. The key is to ensure you are using the AE SDK memory suites as documented to properly manage allocation and deallocation of temporary buffers used during frame processing.

*Tags: `ae-plugin`, `memory`, `memory-management`*

---

### What are the best practices for running computationally intensive algorithms during the PF_Cmd_RENDER call in After Effects plugins?

During a render call, you can execute any code except project modifications. For memory allocation, use After Effects' memory and handle suites rather than standard system allocation, otherwise you compete with AE for resources and risk crashes. If you get a hard crash error like 'Crash in progress', it often indicates failed memory allocation. Debug your code during an AE session to identify issues. Ensure your algorithm works correctly and uses AE's memory management APIs.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

### Why is my image appearing smeared when passing vector<int> pixel data through an iteration function in After Effects?

The issue is likely related to incorrect pixel data ordering or indexing assumptions. The user was storing pixels in one order (x0y0, x0y1, x0y2... x1y0, x1y1, x1y2...) but the iteration function was expecting them in a different order (x0y0, x1y0, x2y0... x0y1, x1y1, x2y1...), causing the image to appear distorted or on its side. To debug, simplify the project to verify that a basic pixel copy (outP = *inP) works correctly first, then incrementally add complexity to identify where the indexing breaks down.

```cpp
static PF_Pixel8
*getXY(PF_EffectWorld &def, int x , int y){
  return (PF_Pixel*)def.data + y * (def.rowbytes / sizeof(PF_Pixel)) + x;
}
```

*Tags: `debugging`, `memory`, `output-rect`, `render-loop`*

---

### How should pixel data be correctly indexed when extracting values from a PF_EffectWorld and storing them in a vector for later retrieval?

When extracting pixel data from a PF_EffectWorld, use proper pointer arithmetic accounting for rowbytes: cast the data pointer to PF_Pixel, then offset by (y * (rowbytes / sizeof(PF_Pixel)) + x). Store pixels in a consistent order (RGBA for each pixel sequentially). When retrieving from the vector in the iteration function, ensure the indexing matches: for pixel at (x, y), access indices starting at ((y * width + x) * 4) for RGBA values in order. The key is maintaining consistency between storage order and retrieval order.

```cpp
// Storage in vector
for(int i = 0; i < tInfo.width; i++){
  for(int j = 0; j < tInfo.height; j++){
    PF_Pixel8 currentPixel = *getXY(*tInfo.input, i, j);
    data.push_back(currentPixel.red);
    data.push_back(currentPixel.green);
    data.push_back(currentPixel.blue);
    data.push_back(currentPixel.alpha);
  }
}

// Retrieval in iteration function
int idx = (yL * tInfo.width + xL) * 4;
outP->red = tInfo->data[idx];
outP->green = tInfo->data[idx + 1];
outP->blue = tInfo->data[idx + 2];
outP->alpha = tInfo->data[idx + 3];
```

*Tags: `debugging`, `memory`, `output-rect`, `render-loop`*

---

### What is the best way to store all pixels from a source layer into an array for processing?

Use the MemorySuite to allocate an int array of any size, then lock the memory handle onto an int pointer which will behave as an array. You can pass a pointer to your custom data as the refcon parameter to the Iterate8Suite, cast it in your iteration function, and fetch your int data according to the x,y coordinates provided by the suite. Alternatively, manually iterate through pixels using getXY() helper function to access PF_Pixel data by calculating the memory offset using rowbytes.

```cpp
static PF_Pixel *getXY(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

// In Render function:
for(int i = 0; i < tInfo->width; i++){
  for(int j = 0; j < tInfo->height; j++){
    PF_Pixel currentPixel = *getXY(*tInfo->input, i, j);
    int alpha = currentPixel.alpha;
    int red = currentPixel.red;
    int green = currentPixel.green;
    int blue = currentPixel.blue;
  }
}
```

*Tags: `aegp`, `memory`, `params`, `reference`*

---

### How should you properly manage and copy pixel buffer data when passing worlds between effects using AEGP_EffectCallGeneric?

When passing world data between effects, you need to understand ownership. AE-owned worlds (input/output buffers during render) will be released after the render call returns, so you must copy the pixel data using PF_COPY(). For plugin-owned worlds, you can pass the AEGP_WorldH directly. To copy: (1) create a new AEGP_WorldH with WorldSuite3()->AEGP_New() at the same dimensions, (2) wrap it with WorldSuite3()->AEGP_FillOutPFEffectWorld(), (3) copy pixels using PF_COPY(). In the receiving plugin, assign the AEGP_WorldH to sequence data and call WorldSuite3()->AEGP_Dispose() when done. Only dispose once, and let wrapped PF_EffectWorld structures go out of scope naturally.

```cpp
// Create and copy a world
AEGP_WorldH newWorldH;
suites.WorldSuite3()->AEGP_New(inputP->width, inputP->height, &newWorldH);

PF_EffectWorld newWorld;
suites.WorldSuite3()->AEGP_FillOutPFEffectWorld(newWorldH, &newWorld);

PF_COPY(&inputWorld, &newWorld, NULL, NULL);

// Send to other plugin via AEGP_EffectCallGeneric
suites.EffectSuite2()->AEGP_EffectCallGeneric(pluginID, effectH, &timeT, (void*)newWorldH);

// In receiving plugin's sequence data
seqDataWorldH = receivedWorldH;
```

*Tags: `aegp`, `arb-data`, `layer-checkout`, `memory`*

---

### Why does memory usage increase continuously when using AEGP_EffectCallGeneric to send mesh data between effects?

Memory leaks when using AEGP_EffectCallGeneric typically stem from improper ownership and disposal of allocated data structures. If you're allocating worlds or buffers to send data and not properly disposing of them after use, AE's memory footprint will continuously grow. Ensure that for any AEGP_WorldH you create with WorldSuite3()->AEGP_New(), you call WorldSuite3()->AEGP_Dispose() exactly once when done. If the issue persists even with empty data, verify that your sequence data management isn't holding references to disposed memory and that you're not allocating new structures on every frame without cleanup.

*Tags: `aegp`, `arb-data`, `caching`, `memory`*

---

### How can I access and modify pixels in DrawSparseFrame of an AEIO without using Iterate8Suite?

You can create a PF_InData struct and fill it with whatever data you have available, which should allow you to use the iterate suites. Alternatively, you can iterate through the buffer's pixels directly using nested loops to access individual pixels via sampleIntegral32. However, be careful with coordinate ordering—ensure you're passing coordinates in the correct order (x,y) rather than swapped (y,x), and account for thumbnail resolution differences which can cause crashes or unexpected behavior.

```cpp
for (A_long i = 0; i < 1000; i++)
for (A_long j = 0; j < 1000; j++){
  PF_Pixel *pixel = sampleIntegral32(wP, i, j);
  pixel->alpha = PF_MAX_CHAN8;
  pixel->red = PF_MAX_CHAN8;
  pixel->green = PF_MAX_CHAN8;
  pixel->blue = PF_MAX_CHAN8;
}
```

*Tags: `aegp`, `aeio`, `debugging`, `memory`, `output-rect`*

---

### How can I iterate through 16-bit and 32-bit per channel pixels directly without using the iterate suites?

You can iterate 16bpc and 32bpc pixels directly by casting the buffer data to the appropriate pixel type. For 16-bit pixels, use: `PF_Pixel16 pix = (PF_Pixel16)buffer->data + xCount + yCount * (buffer->rowbytes / sizeof(Pixel16));`. Convert similarly for 32-bit pixels. The rest of the process is the same as in the CCU sample. Replace all PF_Pixel8 references with PF_Pixel16 or PF_Pixel32 as needed, and update constants like PF_MAX_CHAN8 to PF_MAX_CHAN16 (or 1.0f for 32-bit). For 16-bit support, set PF_OutFlag_DEEP_COLOR_AWARE in out_data->out_flags. For 32-bit support, use SmartFX (see the smartyPants sample).

```cpp
PF_Pixel16 pix = (PF_Pixel16)buffer->data + xCount + yCount * (buffer->rowbytes / sizeof(Pixel16));
```

*Tags: `memory`, `mfr`, `output-rect`, `render-loop`*

---

### How can I check out only a portion of a frame at a different time instead of the entire frame?

You can use checkout_layer_pixels() which takes a request info structure as an argument that allows you to request a smaller rectangle. Alternatively, use the render suite to request the source item directly and set an ROI (region of interest) to get only the part you need. The render suite method doesn't hint AE in advance about your intentions, so AE won't cache items in advance—test both methods to see which is faster for your use case, as getting a potentially cached full frame might sometimes be faster than requesting a partial frame without caching.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`, `output-rect`, `render-loop`*

---

### Why does calling AEGP_ExecuteScript from within an effect plugin crash After Effects?

Executing script commands that modify the scene state (like applying a preset to a layer) from within an effect plugin causes a crash because the script invalidates locked/checked-out memory objects that are currently passed to the effect plugin via in_data. The effect is in mid-call when the scene state changes, creating memory conflicts. You cannot add an effect to a layer while your effect is currently executing. The solution is to defer scene modifications until after the effect has finished execution.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

### How do I rasterize a mask and use it to restrict pixel operations in my effect plugin?

You can rasterize a mask by filling a dummy buffer with color and applying the mask using the MaskWorldWithPath function from the mask suite. However, this requires the mask mode to be set to something other than NONE. A workaround is to use a temporary buffer for rasterization without affecting the project state. Note that calling AEGP_SetMaskMode is undoable and will prompt users to save changes. For querying whether pixels are contained in a mask, rasterize the mask extent into a temporary buffer and query it directly.

*Tags: `aegp`, `mask`, `memory`, `render-loop`*

---

### How can I allocate memory for a Windows GDI+ Bitmap object in an After Effects plugin without using new/delete to avoid crashes?

Use placement new syntax to construct an object in pre-allocated memory. First allocate memory using host_new_handle(), lock it with host_lock_handle(), then use the placement new operator: `Bitmap *tmpImg = new(memV) Bitmap(width, height, bitmap_bytes_per_rowL, PixelFormat32bppPARGB, reinterpret_cast<BYTE *>(bitmap_dataP));` This avoids mixing memory allocation strategies between the host and the plugin, which can cause crashes especially in Windows release builds.

```cpp
PF_Handle bmpH = suites.HandleSuite1()->host_new_handle(sizeof(Bitmap));
void *memV = suites.HandleSuite1()->host_lock_handle(bmpH);
Bitmap *tmpImg = new(memV) Bitmap(width, height, bitmap_bytes_per_rowL, PixelFormat32bppPARGB, reinterpret_cast<BYTE *>(bitmap_dataP));
```

*Tags: `aegp`, `debugging`, `memory`, `windows`*

---

### What is placement new and what are its uses?

Placement new is a C++ feature that allows you to construct an object in memory that has already been allocated. Instead of `new Type()` which allocates memory and constructs, placement new uses the syntax `new (existing_memory) Type()` to only construct the object in the already-allocated space. This is useful when you need to manage memory allocation separately from object construction. See: http://stackoverflow.com/questions/222557/what-uses-are-there-for-placement-new

```cpp
Bitmap *somePointer = new (someMemoryAlreadyAllocated) Bitmap();
```

*Tags: `memory`, `reference`*

---

### How do I properly handle AEGP_EffectRefH and AEGP_StreamRefH disposal when SmartRender operations are interrupted?

Every Get() call must be balanced by a Dispose() call, even when render is interrupted. The key issue is that if an interrupt causes your error variable to have a value, all commands wrapped in the ERR() macro will be skipped, preventing proper disposal. Solution: either set an interrupt flag and nullify the err var (remembering to return the canceled code to AE), or use ERR2() for interrupt checks. Additionally, ensure PF_ABORT is called during iteration and proper disposal happens before returning on abort.

```cpp
if (err = PF_ABORT(in_data)){
  if(outputWorld) (suites.WorldSuite3()->AEGP_Dispose(outputWorld));
  if(effectWorld) (suites.WorldSuite3()->AEGP_Dispose(effectWorld));
  if(streamH) (suites.StreamSuite4()->AEGP_DisposeStream(streamH));
  if(effectH) (suites.EffectSuite2()->AEGP_DisposeEffect(effectH));
  return err;
}
```

*Tags: `aegp`, `debugging`, `memory`, `smart-render`*

---

### What causes incomplete rendering when stacking multiple plugin instances and reloading a project?

The issue is likely caused by confused world buffer handling, especially with temporary buffers that might get overwritten by stacked effects. Carefully trace world creation to ensure input, output, and temporary buffers are not confused. A specific trap: temporary world buffers obtained from AE may actually point to previous frame output buffers in memory. Solution: clean buffer pixels before using temporary buffers, or ensure proper buffer allocation and usage patterns.

*Tags: `caching`, `debugging`, `memory`, `smart-render`*

---

### How do you properly calculate row bytes and gutters when converting pixel iteration code from 8-bit to 16-bit in After Effects?

When converting from 8-bit to 16-bit rendering, you need to adjust your row byte calculations to account for the different pixel sizes. For 8-bit (PF_Pixel8), divide rowbytes by sizeof(PF_Pixel8) to get the row width in pixels. The same principle applies to 16-bit (PF_Pixel16), but you must use sizeof(PF_Pixel16) instead. The gutter is calculated as: gutter = (rowbytes / pixelSize) - width. Common issues arise from not properly accounting for the pixel size difference between bit depths, which can cause row byte calculations to be incorrect and lead to rendering failures.

```cpp
A_long sizeOfPF_Pixel8 = sizeof(PF_Pixel8);
A_long sizeInputRowBytes = (input_worldP->rowbytes / sizeOfPF_Pixel8);
A_long sizeOutputRowBytes = (output_worldP->rowbytes / sizeOfPF_Pixel8);
in_gutterL = sizeInputRowBytes - input_worldP->width;
out_gutterL = sizeOutputRowBytes - output_worldP->width;
```

*Tags: `memory`, `mfr`, `reference`, `render-loop`*

---

### Why does buffer height calculation blow up to around 500 when iterating through rows with zero gutter values?

The issue was caused by incorrect casting of the world as the base address for pixels. The correct approach is to use AEGP_GetBaseAddr32(worldH, &baseAddress) to properly obtain the base address for pixel access, rather than casting the world handle directly.

```cpp
ERR(ws3P->AEGP_GetBaseAddr32(worldH, &baseAddress));
```

*Tags: `aegp`, `buffer`, `debugging`, `memory`, `render-loop`*

---

### What flag should be used when creating a new world buffer for deep color pixel processing?

When creating a new world for processing, use the PF_NewWorldFlag_DEEP_PIXELS flag instead of PF_NewWorldFlag_NONE to properly allocate buffers for 16-bit and 32-bit color depths. This ensures the world is correctly initialized for deep color operations.

```cpp
ERR(in_data->utils->new_world(in_data->effect_ref, width, height, PF_NewWorldFlag_DEEP_PIXELS, &outputWorld));
```

*Tags: `aegp`, `memory`, `params`, `render-loop`*

---

### What are the best approaches for accessing and sampling specific pixels from a buffer in After Effects?

There are two main approaches: (1) Use the Sample Suite to grab color values from a pixel, subpixel, or area within a buffer - see the Shifter sample; (2) Access pixel data directly in RAM and retrieve the data yourself - see the CCU sample. Choose based on whether you need convenience (Sample Suite) or direct memory access performance.

*Tags: `layer-checkout`, `memory`, `reference`, `sdk`*

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

### What does host_lock_handle do and how should memory be allocated in After Effects plugins?

host_lock_handle tells After Effects that a memory chunk is in use and should not be moved by the operating system. In After Effects plugins, you should use the memory suites API (host_create_handle + host_lock_handle) rather than malloc/new, to allow AE to make RAM allocation decisions at the application scope. The pattern host_create_handle + host_lock_handle is functionally equivalent to malloc. For handles you allocate, you must lock, unlock, and dispose of them. Some handles are allocated and locked for you by AE (like those from ExecuteScript), while others like global_data and sequence_data are managed by AE automatically when the plugin is called.

*Tags: `aegp`, `memory`, `plugin-api`*

---

### Does host_lock_handle provide thread safety and is it recursive?

host_lock_handle does NOT provide thread safety—it only prevents AE from moving the memory chunk. Regarding recursiveness (multiple locks on the same handle), the behavior is unclear. Calling host_lock_handle twice followed by a single host_unlock_handle may or may not leave the handle locked, depending on whether the locking mechanism is reference-counted. It's safest to maintain a 1:1 ratio of lock to unlock calls per handle.

*Tags: `aegp`, `memory`, `threading`*

---

### How can an effect force full resolution rendering when the user has downsampled the composition?

Use the RenderAndCheckoutFrame AEGP function to render at full resolution. However, be careful to render the composition your effect is applied IN (not TO) to avoid infinite loops. You can check the downsample factors using AEGP_GetDownsampleFactor and conditionally render full resolution only when downsampling is detected. Make sure to use AEGP_GetEffectLayer instead of AEGP_GetActiveLayer to get your effect's layer, as 'active' doesn't necessarily refer to your layer.

```cpp
if (1 != in_data->downsample_x.num || 1 != in_data->downsample_x.den || 1 != in_data->downsample_y.num || 1 != in_data->downsample_y.den) {
  AEGP_ItemH itemH = NULL;
  AEGP_LayerH layerH = NULL;
  suites.EffectSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
  // Set downsample factor to 1,1 for full resolution
  suites.RenderOptionsSuite2()->AEGP_SetDownsampleFactor(roH, 1, 1);
  suites.RenderSuite2()->AEGP_RenderAndCheckoutFrame(roH, NULL, NULL, &receiptH);
}
```

*Tags: `aegp`, `layer-checkout`, `memory`, `smart-render`*

---

### Why am I getting black pixels (0,0,0,0 RGBA) when rendering a full resolution frame with RenderAndCheckoutFrame?

The issue is likely that you're using AEGP_GetActiveItem and AEGP_GetActiveLayer, which don't necessarily refer to your effect's layer. Use AEGP_GetEffectLayer instead, which is guaranteed to return your effect's actual layer. The black pixels suggest the render is not finding the correct layer to render.

*Tags: `aegp`, `debugging`, `memory`, `smart-render`*

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

### How can I render only a single color channel (R, G, B, or A) from a PF_EffectWorld?

There are several approaches to isolate and display a single channel from a PF_EffectWorld without subpixel sampling. The iteration suite is recommended as it is multi-threaded and easy to manage—you iterate through input and output buffers and selectively zero out unwanted channels while preserving the desired one. Alternatively, you can manually access the world's base address and directly manipulate individual PF_Plane channels in memory for maximum efficiency, though this sacrifices multi-threading support. A compositing approach can also work: create a new world filled with the desired color and composite it using composite_rect with multiply transfer mode to suppress other channels. If using SmartFX, you can declare which channels to use and let AE ignore the others, though this approach was noted as potentially risky.

```cpp
outP->alpha = inP->alpha;
outP->red = 0;
outP->green = 0;
outP->blue = inP->blue;
```

*Tags: `aegp`, `memory`, `render-loop`, `smartfx`*

---
