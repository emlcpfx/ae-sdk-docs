# Multi-Frame Rendering (MFR)

> 101 Q&As · source: AE plugin dev community Discord

### How much work is required to update a simple AE plugin for Multi-Frame Rendering (MFR)?

For simple plugins that don't use sequence data or caching, there is not much to do besides rebuilding. The complexity comes when you have GPU plugins, custom caching, or sequence data which require more significant refactoring for thread safety.

*Tags: `mfr`, `multi-frame-rendering`, `plugin-update`, `thread-safety`*

---

### What is the best practice when a plugin runs out of GPU memory during rendering?

Returning PF_Err_OUT_OF_MEMORY may cause AE to show a black frame without warning the user. Options: (1) Use a C++ text library to render 'Out of GPU memory' text directly onto the frame. (2) Set a static global boolean (protected by mutex) in the render thread, then display the warning in another thread like PF_UpdateParamUI. Avoid using out_data warning messages during renders as they would fail render queue operations and prevent MFR from retrying with fewer threads.

*Tags: `error-handling`, `gpu`, `mfr`, `out-of-memory`, `render-queue`*

---

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

### What causes AE error code 1397908844?

Error 1397908844 is a generic exception handler/catch-all in Adobe's suites manager, dating back to CS2/CS3. It commonly appears when double-freeing/releasing a suite pointer. It appears more often in newer AE versions due to MFR with global suite handles shared between threads. Known triggers include: problems with arb params, crashes after exporting RAM previews, and double-releasing suites. The underlying issue is usually in the plugin's suite management code.

*Tags: `debugging`, `double-free`, `error-code`, `mfr`, `suite-manager`*

---

### Why is PF_Cmd_SEQUENCE_FLATTEN called when an effect is first applied, and why are Sequence Resetup/Setdown called multiple times on different threads?

This is correct behavior when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set. AE flattens sequence data so it can be copied (for render thread copies). Since AE 2015, rendering uses separate threads with separate project copies. Resetup may receive unflat data, so tag your data to indicate flat/unflat state. Use PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA to allow AE to get the flat copy without losing the original unflat one, reducing unnecessary resetup calls. During sequence setup/resetup, don't assume the plugin can find its own layer/comp - AE sometimes constructs the instance before associating it to a layer. For unique instance IDs, flag sequence data as 'requires checking' on resetup, then scan for duplicates when the instance can be located.

*Tags: `flatten`, `instance-id`, `mfr`, `resetup`, `sequence-data`, `threading`*

---

### Should I use multithreading (iterate_generic) or MFR (Multi-Frame Rendering) in my plugin, or both?

Use both. MFR and multithreading can coexist because not all steps in the single-frame rendering pipeline are multi-threadable. MFR shines for sequential parts, while multithreaded parts (via iterate callbacks) are critical for performance. Mixing MFR with MT doesn't necessarily cause competition because MT-friendly code tends to be CPU cache friendly, while other parts may leave the CPU waiting for RAM. When using iteration callbacks, AE may prioritize between MFR and MT threads. The AE engineering team tested these scenarios extensively.

*Tags: `iterate-generic`, `mfr`, `multi-frame-rendering`, `multithreading`, `performance`*

---

### After upgrading a plugin from CS5 SDK to the 2021 SDK with MFR support, AE crashes when saving the project. How to debug?

The crash is likely related to sequence data flattening. Check if PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set in global setup - if so, you must handle PF_Cmd_SEQUENCE_FLATTEN properly. When modifying sequence data, be careful about whether you're changing data in the same handle or allocating a new one. Debug steps: (1) Purge Xcode intermediates and clean rebuild. (2) Test SDK sample projects to verify they work. (3) Comment out parts of your plugin to isolate the issue. (4) Copy your code onto a fresh sample project.

*Tags: `crash`, `debugging`, `mfr`, `save`, `sdk-upgrade`, `sequence-flatten`*

---

### What was required to update a simple plugin for MFR compatibility?

For simple plugins without sequence data or fancy features, MFR compatibility mainly requires rebuilding the plugin. No major refactoring is necessary if the plugin doesn't use advanced features.

*Tags: `build`, `mfr`*

---

### What are the main challenges with GPU plugins and MFR compatibility?

GPU plugins have significant stability issues with MFR compared to CPU plugins. The issues are hard to track down because crashes occur only after extended rendering periods. OpenGL-based plugins experience particular difficulties, especially on macOS where platform-specific issues arise.

*Tags: `debugging`, `gpu`, `mfr`, `opengl`*

---

### How can sequence_data be used to set values in the output data?

The out_data parameter has sequence_data as a property that can be used to set the values of sequence data. This functionality works in After Effects 2021 and will also work in 2022 if you don't set the MFR flag, though it may show a warning in 2022.

*Tags: `mfr`, `params`, `sequence-data`*

---

### Why is the plugin experiencing glitch behavior with MFR enabled when the Multithreaded flag is not set?

Different threads/CPUs are using different previous frames when MFR (Multi-Frame Rendering) is enabled, causing the glitch behavior. The issue resolves when MFR is turned off. The root cause appears to be that MFR is being invoked even though the Multithreaded flag is not explicitly set for the plugin.

*Tags: `debugging`, `mfr`, `threading`*

---

### What needs to be updated when using PF_COPY buffer operations?

When using PF_COPY or other buffer operations, the source and destination rectangles (src and dst rects) will need to be updated to reflect the operation.

*Tags: `memory`, `mfr`, `output-rect`*

---

### Why must X and Y positions be cast to FIX when using subpixel_sample_float, and is there a better sampling method for smoother displacement?

The subpixel_sample_float function requires FIX format for coordinate input despite being a float sampling function. FIX is a fixed-point format used by After Effects for precise sub-pixel positioning, not a simple integer rounding. For smoother displacement results comparable to vanilla AE or professional displacement plugins, ensure you are using the correct sampling suite and that your displacement values are calculated with sufficient precision before conversion. Consider reviewing the sampling parameters and whether alternative sampling methods or higher precision displacement calculations might improve quality.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
        INT2FIX(newX),
        INT2FIX(newY),
        &giP->inSamp_pb,
        &samplePixel));
```

*Tags: `debugging`, `displacement`, `mfr`, `quality`, `sampling`*

---

### Why must X and Y position arguments be cast to FIX when using subpixel_sample_float, and is there a better sampling method for smoother displacement?

The subpixel_sample_float function requires FIX format for coordinate arguments even though it returns float samples. This is part of AE's API design. For smoother displacement results comparable to vanilla AE or other displacement plugins, you should review sampling best practices in the Warbler plugin tutorial, which covers proper displacement sampling techniques.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
         INT2FIX(newX),
         INT2FIX(newY),
         &giP->inSamp_pb,
         &samplePixel));
```

*Tags: `debugging`, `displacement`, `gpu`, `mfr`, `sampling`*

---

### What algorithm does the Matte/Simple Choker Effect use to find and create reduced/expanded edges on alpha channels?

The questioner describes their own implementation which analyzes each transparent pixel and looks in 8 directions to find nearby opaque pixels to identify outline/choker matte pixels. However, no answer was provided in the conversation about the actual method used by Adobe's Matte/Simple Choker Effect.

*Tags: `gpu`, `memory`, `mfr`, `optimization`, `render-loop`*

---

### What algorithm does the Matte/Simple Choker Effect use to find and create reduced/expanded edges in alpha channels?

The user describes their current implementation which analyzes each transparent pixel and looks in 8 directions to find nearby opaque pixels to identify outline/choker matte pixels. However, this approach is very slow (30+ seconds). The conversation does not provide a definitive answer about what method Adobe's Choker effect actually uses, only describes the user's current slow implementation.

*Tags: `gpu`, `memory`, `mfr`, `optimization`, `render-loop`*

---

### Does changing image size or color depth affect the frame number where a cache-related crash occurs?

The suggestion is to test whether changing image size or color depth of the project changes the crash frame number, as this could indicate whether RAM access is being handled correctly with time-varying data, or if there's a conflict between sequence data writing and MFR.

*Tags: `caching`, `debugging`, `memory`, `mfr`, `sequence-data`*

---

### Can you write to sequence data during PF_update_param_ui with MFR?

No, you cannot write to sequence data during PF_update_param_ui with MFR. You can only do it in the user changed param callback. If you need to modify layer defaults, you may need to use AEGP instead.

*Tags: `aegp`, `mfr`, `params`, `sequence-data`*

---

### How do you extract scale values from a decomposed transformation matrix?

Use AEGP_MatrixDecompose4() on your result matrix to decompose it into position (posVP), scale (scaleVP), shear (shearVP), and rotation (rotVP) components. The scale values will be in the scaleVP output parameter.

```cpp
A_FloatPoint3 posVP, scaleVP, shearVP, rotVP;
A_Matrix4 resultMat4;
ERR(suites.MathSuite1()->AEGP_MatrixDecompose4(&resultMat4, &posVP, &scaleVP, &shearVP, &rotVP));
```

*Tags: `aegp`, `mfr`, `params`*

---

### Is it normal to receive thousands of calls to PF_Arbitrary_DISPOSE_FUNC on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls on shutdown is likely due to MFR (Multi-Frame Rendering) duplicating arbitrary parameters for each thread, causing many copy and dispose cycles as parameters change and threads are cleaned up during shutdown.

*Tags: `arb-data`, `memory`, `mfr`, `threading`*

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

### What should you avoid doing with globalData in After Effects 2022 and later?

Do not write to globalData outside of globalsetup/setdown in AE 2022 and later, even for non-MFR plugins. After Effects has stricter requirements about globalData immutability. Instead, store mutable data in sequence data (with appropriate callbacks) or static variables.

*Tags: `memory`, `mfr`, `params`*

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

### What are the advantages of compute cache compared to arbdata for storing pointers to heap objects that are flattened/unflattened?

Compute cache can be written and read from any thread (render or UI thread) using std::atomic for thread safety. Unlike arbdata, compute cache is not stored with the project. Compute cache is designed to get a value per frame, whereas arbdata requires a global context where you define arrays or vectors to store data for different frames. Compute cache replaced sequence data after MFR was introduced since sequence data can no longer be used.

*Tags: `arb-data`, `compute-cache`, `memory`, `mfr`, `sequence-data`, `threading`*

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

### When do I need to implement PF_Cmd_FRAME_SETUP to customize output size?

You only need to implement PF_Cmd_FRAME_SETUP for non-SmartFX effects if your effect expands the output buffer size, such as with glow or drop shadow effects. For effects that maintain the same output dimensions as the input, this command is not necessary.

*Tags: `mfr`, `output-rect`, `smartfx`*

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

### Do you need to lock the handle when calling GetConstSequenceData for sequence data?

Yes, you need to lock the handle even when reading only. Use PF_LOCK_HANDLE(const_seq) to lock the handle before accessing it, and unlock it after use. This is necessary in multi-frame render (MFR) scenarios with multiple plugin applications, and the lack of locking can cause nullptr returns and crashes. While Adobe documentation doesn't clearly document this requirement, it appears to be essential for thread safety.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `aegp`, `debugging`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### What causes the generic exception handler in Adobe suites manager to appear, and why is it more frequent in newer AE versions?

The generic exception handler/catch-all in the Adobe suites manager appears when double freeing or releasing a suite pointer. It has existed since CS2 or CS3 days. It appears more often in newer AE versions likely due to MFR (Multi-Frame Rendering) with global suite handles or suite pointers being shared between threads.

*Tags: `aegp`, `memory`, `mfr`, `threading`*

---

### Why does the precomp result show only a masked region as the result rect with everything else black?

When the precomp is created, the result rect is determined by the masked layer inside it. Since the precomp is full comp size but contains only a masked layer, only the area around the mask is included in the result rect, leaving everything else black.

*Tags: `layer-checkout`, `mfr`, `output-rect`*

---

### How do 3D Camera Tracker and Warp Stabilizer output progress over frames with live updates without cache purging?

According to an After Effects developer, these built-in tools use internal APIs that are not available to regular plugin developers. The Warp Stabilizer effect 'cheats' and does not operate within the constraints of a normal AE Effect API plug-in. Adobe's internal team has access to secret suites that allow this functionality, which is not exposed to third-party developers. A feature request was made in 2023 to expose this capability to plugin developers, but it remains unavailable.

*Tags: `aegp`, `caching`, `debugging`, `mfr`, `smart-render`*

---

### How should you handle null inoutworld parameters in Premiere Pro plugins?

If you set a condition to check if inoutworld is null and return an error, Premiere Pro will redo the render. This is a valid strategy for handling null world pointers in render functions.

```cpp
if (inoutworld == null) return err;
```

*Tags: `debugging`, `mfr`, `premiere`, `render-loop`*

---

### How do you define ARGB_8u color space in globalSetup for After Effects plugins?

When using ARGB_8u as your color space, you need to properly define it in the globalSetup function. The specific definition depends on your plugin architecture and color space requirements.

*Tags: `build`, `mfr`, `params`*

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

### How can you set sequence data values in After Effects plugins?

You can use the out_data property which has sequence_data as a property that can be used to set the values of sequence data. This approach works in After Effects 2021 and 2022 (in 2022, it will work without the MFR flag set, though a warning may be shown).

*Tags: `aegp`, `mfr`, `sequence-data`*

---

### Why is a plugin exhibiting glitch behavior with Multi-Frame Rendering (MFR) even though the Multithreaded flag is not set?

MFR glitches can occur when different threads/CPUs are using different previous frames, causing inconsistent rendering behavior. Even without the Multithreaded flag explicitly set, the plugin may still attempt MFR, which can lead to these issues. Disabling MFR resolves the problem. The root cause appears to be improper frame state management across threads during multi-frame rendering.

*Tags: `debugging`, `mfr`, `render-loop`, `threading`*

---

### What is the behavior of After Effects regarding multi-frame rendering (MFR) and thread safety in plugins?

Even if plugins don't specify they are MFR safe, After Effects still calls them from different threads and requests frames out of sequence (for example, frame 5, then frame 3, then frame 6). The only way to prevent this non-sequential, multi-threaded behavior is to disable MFR completely.

*Tags: `mfr`, `render-loop`, `threading`*

---

### How do you expand the output buffer with smartFX to prevent outlines from being clipped by layer bounds?

When using smartFX in After Effects, if you're generating outlines around an alpha image and they're getting cut off by the layer bounding box, you need to expand the output buffer. This is typically done by modifying the output rectangle in your smartFX implementation to be larger than the layer's current bounds, ensuring that generated content (like outlines) isn't clipped.

*Tags: `mfr`, `output-rect`, `render-loop`, `smartfx`*

---

### Why must X and Y coordinates be cast to FIX when using subpixel_sample_float, and what sampling method should be used for smoother displacement results?

When using the SamplingFloatSuite1()->subpixel_sample_float() function, coordinates are cast to FIX format (fixed-point representation) rather than remaining as floats. This is part of the After Effects API's internal coordinate system. The FIX format provides the necessary precision for the sampling algorithm while maintaining compatibility with AE's rendering pipeline. For smoother displacement results comparable to After Effects' native displacement or professional plugins, ensure you're using subpixel_sample_float with proper coordinate conversion, and consider whether your displacement values are being calculated with sufficient precision before being passed to the sampling function. The quality difference may also stem from how displacement offsets are computed rather than the sampling function itself.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
        INT2FIX(newX),
        INT2FIX(newY),
        &giP->inSamp_pb,
        &samplePixel));
```

*Tags: `aegp`, `displacement`, `mfr`, `output-rect`, `sampling`*

---

### How can you cache frame data that's too large to fit in RAM?

You can save binary data directly to disk for each frame (e.g., 0000.dat, 0001.dat, etc.), similar to how Houdini handles large computation results. After Effects may have a cache API for this, but if not, you'll need to implement your own disk-based caching system to store frame data when it exceeds available memory.

*Tags: `caching`, `memory`, `mfr`, `render-loop`*

---

### Can I save cached frames to a custom file format and read them back in a plugin?

Yes, you can save frames to any file format you want and read them back in. This is a common approach when caching large amounts of frame data that won't fit in sequence data. You can use custom binary formats (like .dat files) or other formats suitable for your needs. The plugin can write frames to disk during processing and read them back as needed, avoiding the limitations of storing arbitrarily large numbers of frames directly in sequence data.

*Tags: `caching`, `compute-cache`, `mfr`, `sequence-data`*

---

### How does caching behavior work by default in the render function, and how can you update progress indication?

Caching calculations taking place in the render function will have default caching behavior where the output won't be updated until the plugin has populated it. For updating the loading bar specifically, use PF_Progress.

*Tags: `caching`, `mfr`, `render-loop`*

---

### How can you diagnose whether a crash is related to RAM caching issues in After Effects plugins?

Try changing the size of the image or color depth of the project to see if the crash frame number changes. This can help determine if the issue is related to RAM access handling, particularly with time-varying data or conflicts between sequence data writing and MFR (Multiprocessing Framework).

*Tags: `caching`, `debugging`, `memory`, `mfr`, `sequence-data`*

---

### Can you write to sequence data during PF_update_param_ui callback?

No, you cannot write to sequence data during PF_update_param_ui. Writing to sequence data is only possible in the user changed param callback. With MFR, you also cannot write to sequence data during PF_update_param_ui. However, in certain cases you may be able to accomplish similar goals without AEGP by using PF_LayerDefault_NONE or similar approaches, though AEGP is often the recommended solution.

*Tags: `aegp`, `mfr`, `params`, `sequence-data`*

---

### Why does Multi-Frame Rendering (MFR) start with multiple threads but reduce to a single thread on longer compositions?

James Whiffin observed that MFR often initiates rendering with 6-10 threads but consistently drops to single-threaded operation when compositions are sufficiently long. This behavior persists even when custom effects are disabled and only first-party Adobe effects are present, suggesting it is a characteristic of MFR itself rather than third-party plugin behavior.

*Tags: `mfr`, `performance`, `render-loop`, `threading`*

---

### Is it normal to receive thousands of PF_Arbitrary_DISPOSE_FUNC calls on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls during shutdown is likely due to Multi-Frame Rendering (MFR) duplicating arbitrary parameters for each thread, combined with parameter changes triggering dispose/copy cycles.

*Tags: `arb-data`, `memory`, `mfr`, `shutdown`, `threading`*

---

### What should you assume about the data pointer in pF_effectworld when using GPU mode in After Effects plugins?

According to the SDK documentation, when using GPU mode (CUDA, Metal, or OpenCL), you must assume that the data pointer is NULL in the pF_effectworld structure. This means the pixel data is not available in system memory and you need to work with GPU-resident data instead.

*Tags: `cuda`, `gpu`, `metal`, `mfr`, `opencl`, `reference`*

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

### How can I check out the current layer with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT to check out the INPUT layer at different times, it returns the effectworld before all effects are applied, including effects applied before the current effect. However, this is possible to achieve with all effects applied, as demonstrated by the Echo effect. The solution involves understanding the layer checkout mechanism and potentially using different checkout strategies or timing parameters to access the fully composited layer rather than the raw input.

*Tags: `aegp`, `layer-checkout`, `mfr`, `render-loop`*

---

### What are the advantages of compute cache compared to arbdata for storing pointers to heap data?

Compute cache offers several advantages over arbdata: (1) it can be written/read from any thread (render or UI) with optional thread-safety using std::atomic, (2) it's designed per-frame rather than requiring a global context like sequence or arb data, (3) it provides functions to specify whether to wait if another thread is computing the same key. However, arbdata gets saved with the project while compute cache does not. Compute cache replaces sequence data now that MFR is introduced and sequence data can no longer be used.

*Tags: `arb-data`, `compute-cache`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### How does transform application differ between SmartFX and standard pipelines in After Effects?

In SmartFX, if a layer is continuously rasterized, you receive the input buffer with transforms already applied. If not continuously rasterized, the transforms are applied after your effect populates the output buffer. This means you won't know if transforms changed in the standard pipeline unless using SmartFX.

*Tags: `mfr`, `output-rect`, `render-loop`, `smartfx`*

---

### Is there a reference repository documenting After Effects SDK structures and flags?

The virtualritz/after-effects repository on GitHub contains useful references for understanding PF_LayerDef structures and world_flags behavior, including documented examples of how to extract bit depth information from layer definitions.

*Tags: `mfr`, `open-source`, `reference`, `tool`*

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

### What causes the generic exception handler/catch-all error in Adobe After Effects, and why does it appear more frequently in newer versions?

The generic exception handler/catch-all in the Adobe suites manager is triggered by issues like double freeing or releasing a suite pointer. This error has existed since CS2 or CS3 days. It appears more often in newer AE versions due to Multi-Frame Rendering (MFR) with global suite handles or suite pointers being shared between threads, which increases the likelihood of memory management conflicts.

*Tags: `aegp`, `memory`, `mfr`, `threading`*

---

### What flag should be set in global setup to handle premultiplied alpha correctly?

Set the flag PF_outflags2_REVEALS_ZERO_ALPHA in global setup to properly handle cases where the plugin reveals zero alpha, which is related to the premultiplied alpha of the input layer.

*Tags: `alpha`, `mfr`, `params`, `pipl`*

---

### Why does a blur function work differently between 8-bit and 16/32-bit color depths in After Effects?

When implementing blur effects that support multiple bit depths (8-bit, 16-bit, and 32-bit), subtle differences in how path values are converted between depth formats can cause discrepancies. One workaround is to precompose the layer with the effect first, which ensures consistent processing. The issue likely stems from improper conversion of path values when switching between 8-bit and 16/32-bit processing paths.

*Tags: `debugging`, `memory`, `mfr`, `render-loop`*

---

### What approach did reduxFX use to handle multi-layer compositing in After Effects plugins?

Instead of using a plugin-per-layer approach (which caused render order issues), reduxFX implemented a single plugin with 10 layer parameters that could pull in pixels from multiple layers, apply lighting/material options, and composite everything together within that single plugin. While AE's rigid parameter interface made it somewhat clunky to use, this approach worked surprisingly well and was later ported to node-based hosts like Nuke and Natron where the multi-input approach worked more naturally.

*Tags: `mfr`, `params`, `render-loop`, `ui`*

---

### Is it possible to achieve progressive rendering in an After Effects plugin where samples render incrementally and display results to the user in real-time?

gabgren mentioned porting functionality from outside AE where progressive sampling over time with continuous result display is possible, making the interaction feel more responsive. They noted that the current AE plugin workflow requires waiting for all samples to render before seeing results on screen, which feels slow. They're exploring whether AE plugins can support this type of progressive rendering approach.

*Tags: `mfr`, `performance`, `render-loop`, `smart-render`, `ui`*

---

### What is an effective architecture for creating interactive material systems with child plugins on layers controlled by a master adjustment layer?

One approach is to use an adjustment layer with a master plugin that loops over child layers and inspects their effects for specifically-named control instances. Rather than creating custom effects, you can use built-in Slider Control and Checkbox Control effects renamed in the UI, with the master plugin scanning for these named controls. The master effect reads their values to drive the material interactions. This keeps material-specific controls directly on the affected layers rather than centralizing all parameters in the master effect, improving UX. A more polished approach would involve creating dummy effects that don't render but provide a cleaner interface for users to specify layer properties without manually adding and renaming slider controls.

*Tags: `aegp`, `mfr`, `params`, `ui`*

---

### Why does a plugin work in the Plugins folder but not the Mediacore folder on Mac After Effects 26.0.0?

Multiple developers reported this issue with After Effects 26.0.0 on Mac where plugins load correctly from the Plugins folder but fail to load from the Mediacore folder. The Gaussian Splat team issued an update to address AE 26 compatibility, suggesting there are specific compatibility changes in AE 26 affecting Mediacore plugin loading on Mac. The issue also affects Premiere Pro 2026, where plugins in Mediacore don't appear at all, even though they work in earlier Premiere versions (2023-2025) and in AE 2026.

*Tags: `cross-platform`, `debugging`, `deployment`, `macos`, `mfr`*

---

### How can you iterate through pixel data when working with rowbytes in After Effects or Premiere?

You can iterate through pixel data yourself using rowbytes. Be careful with negative values in Premiere, as they can occur and need special handling.

*Tags: `caching`, `memory`, `mfr`, `premiere`*

---

### What is the issue with the no_params_vary flag and how does it differ between After Effects and Premiere?

The no_params_vary flag was identified as the origin of a problem in plugin behavior. In After Effects, this flag's behavior can be replaced using MIX_GUI during the pre_render thread. However, in Premiere, the flag does nothing, indicating different parameter variation handling between the two applications.

*Tags: `mfr`, `params`, `premiere`, `render-loop`*

---

### How can I make a plugin support 8-bit, 16-bit, and 32-bit projects while sharing the same internal codebase without duplicating logic?

Use C++ templates to create multiple instances of your algorithm with different data types and range constants. This allows you to maintain a single shared codebase while handling different bit depths. Instead of letting After Effects automatically convert bit depths (which introduces color noise), you can manually handle the conversion within your plugin by instantiating template versions for 8-bit, 16-bit, and 32-bit processing.

*Tags: `build`, `mfr`, `params`, `reference`*

---

### Why is PF_Cmd_SEQUENCE_FLATTEN sent when an effect is first applied, and why are SEQUENCE_RESETUP and SEQUENCE_SETDOWN called multiple times on different threads?

PF_Cmd_SEQUENCE_FLATTEN is sent when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set during global setup. Flattening allows plugins to serialize complex data structures (containing pointers or other non-copyable elements) into a single copyable piece of memory. Multiple SEQUENCE_RESETUP calls occur because AE's rendering architecture separates UI and rendering threads, each maintaining their own copy of sequence data. The PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA flag allows returning a flattened copy without losing the original, reducing some resetup calls on the UI thread. The PF_OutFlag2_MUTABLE_RENDER_SEQUENCE_DATA_SLOWER flag accommodates plugins requiring separate sequence data per rendering thread. Multiple SEQUENCE_SETDOWN calls correspond to cleanup on different threads.

*Tags: `aegp`, `mfr`, `sequence-data`, `threading`*

---

### Should I use multithreading or MFR in my After Effects plugin, or can both be used together?

Both multithreading (MT) and Multi-Frame Rendering (MFR) can coexist in plugins. While they may seem incompatible at first, MFR and MT don't necessarily compete for CPU resources because not all steps in the rendering pipeline of a single frame are multi-threadable. MFR excels at handling non-multi-threadable steps, while the multi-threadable parts are often CPU cache-friendly and don't compete with MFR threads. The Adobe After Effects engineering team has extensively tested these scenarios and optimized the mechanism to handle a wide range of rendering scenarios, including plugins that aren't MFR-enabled and those relying on GPU rendering.

*Tags: `mfr`, `render-loop`, `sdk`, `threading`*

---

### Why does an After Effects effect plugin crash when saving a project after upgrading to the latest SDK with MFR support?

The crash on save is likely related to sequence data flattening. When upgrading to newer SDKs, if you set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag during global setup but don't implement the PF_Cmd_SEQUENCE_FLATTEN selector, After Effects will crash when trying to save the project. The SEQUENCE_FLATTEN command is sent when saving or duplicating sequences and requires you to flatten sequence data containing pointers or handles so it can be written to disk. Additionally, verify you're allocating new handles when modifying instance sequence data rather than changing data in the same handle. Basic troubleshooting includes cleaning build intermediates, rebuilding from scratch, testing sample SDK projects, and progressively commenting out plugin code to isolate the problem.

*Tags: `build`, `debugging`, `mfr`, `sdk`, `sequence-data`*

---

### Can you hide the second dropdown in PF_ADD_LAYER layer selector parameters?

No, the second dropdown (which allows selection between "Source", "Masks", or "Effects & Masks") cannot be hidden. That is the standard appearance of the layer selector in current versions of the After Effects SDK.

*Tags: `mfr`, `params`, `sdk`, `ui`*

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

### Why don't slave parameters update when the master parameter is animated and scrubbed through the timeline?

Since CC2015, the render thread is not allowed to modify the project in any way, which prevents parameter updates during rendering. Additionally, with MFR (Multiple Frame Rendering), multiple frames are rendered simultaneously, making it ambiguous what value a parameter should have. If the slave parameter only affects the display UI and doesn't influence the actual render, you can work around this by: (1) creating a custom UI that redraws on time changes using its draw event, or (2) creating an AEGP with an idle_hook that scans the current composition for your effect and modifies it as needed. However, if the slave parameter needs to affect the render or be read by expressions in other effects/layers, this design pattern is not supported by After Effects' workflow.

*Tags: `aegp`, `mfr`, `params`, `render-loop`, `ui`*

---

### How does the PF_FloatMatrix work and how do you use it with transform_world() to transform a PF_EffectWorld?

PF_FloatMatrix is a standard 3x3 transformation matrix used in graphics programming. It can perform translation, rotation, scaling, and skewing transformations all in one concatenated matrix. The matrix indices control different transformation properties: the first row typically contains x-translation and scaling/rotation components, while the third row contains y-translation and scaling/rotation components. When applying rotations using a matrix, you place sin and cos (and negative values) in specific positions following standard 2D/3D graphics conventions. Important note: After Effects uses row-based matrices, which is the opposite of the OpenGL standard, so you need to swap the axis accordingly. The transform_world() function applies the supplied matrix to transform the input image, and the result is transformed along with the layer. There is no "current transformation" state—the matrix is applied independently.

*Tags: `mfr`, `params`, `reference`, `transformation`*

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

### How do you animate shapes in an After Effects plugin as the user moves the timeline slider?

Use in_data->current_time to get the current frame's time step and in_data->time_step to get the number of steps per second. AE calls the plugin to render a frame with a given time stamp. When rendering a sequence, AE calls the plugin multiple times, each with a corresponding timestamp. You can then compute your shape function shape(t) for each requested frame time, allowing dynamic shapes with varying numbers of paths over time.

```cpp
double x = 100;
double y = 100;
double R = 50;
double phi1 = 0.;
double phi2 = 2*MF_PI;
cairo_t *cr = cairo_create(surface);
cairo_arc(cr, x, y, R, phi1, phi2);
// Use in_data->current_time to get frame time and call shape(t) for animation
```

*Tags: `aegp`, `mfr`, `params`, `render-loop`*

---

### How do you force re-rendering when masks are deleted or added in an AEGP plugin?

Set the PF_OutFlag2_DEPENDS_ON_UNREFERENCED_MASKS flag in your plugin's global setup. This tells After Effects that your effect depends on mask information even when those masks aren't directly referenced as parameters, ensuring the render refreshes when masks change.

*Tags: `aegp`, `mfr`, `params`, `render-loop`*

---

### What other techniques can help trigger re-renders when external layer properties change in AEGP plugins?

Use GuidMixInPtr to force in external factors, set 'non param vary' in global setup, or consider implementing SmartFX for more sophisticated dependency tracking. Note that GuidMixInPtr may require SmartFX implementation and additional tweaking.

*Tags: `aegp`, `mfr`, `params`, `smartfx`*

---

### Can I implement a mirror effect using the transform_world function?

Yes, transform_world handles all 2D transformations including rotating, scaling, and moving. If 2D transformations are sufficient for your mirror effect, there is no reason not to use transform_world rather than manually calculating transformations.

*Tags: `mfr`, `params`, `reference`*

---

### How can I efficiently access neighboring pixel values during CCU iteration without using sampleIntegral32 on every pixel?

You can use pointer math to access adjacent pixels directly from the input buffer pointer. If inP is a pointer to the current pixel, you can get the next pixel on the same line with `PF_Pixel *next = inP + 1;`. However, for better performance when needing multiple integer samples of other pixels, use iterate_generic instead of the regular iteration suites, as it allows much more optimization potential than acquiring and releasing the sample suite on every pixel.

```cpp
PF_Pixel *next = inP + 1; // gives you the next pixel on the same line
```

*Tags: `aegp`, `ccu`, `iteration`, `memory`, `mfr`, `performance`*

---

### What happens when a user interrupts rendering by scrubbing the playhead in After Effects?

When the user interrupts rendering by moving the playhead, After Effects does not wait for the render function to complete, and frame setdown is not called in-between. The plugin should call PF_ABORT to check if After Effects wants to quit the current frame's render. If the plugin decides to honor the abort request, it should return PF_Interrupt_CANCEL as the error code from the main entry point.

*Tags: `aegp`, `interruption`, `mfr`, `render-loop`*

---

### What happens if a plugin doesn't check for interrupt with PF_ABORT during rendering?

After Effects does not forcibly kill plugins in mid-call. Instead, AE tells the plugin it wants to abort and waits for the render call to return. The call should return either PF_ErrNONE or PF_Interrupt_CANCEL. If PF_ErrNONE is returned, AE might cache that result. If PF_Interrupt_CANCEL is returned, AE will discard the output buffer content as it has been reported as invalid.

*Tags: `aegp`, `caching`, `mfr`, `render-loop`*

---

### Why is motion blur incorrect when using PF_TransformWorld with fast rotation?

When using PF_TransformWorld with only two matrices (previous and current), motion blur artifacts can occur during fast rotation, causing particles to appear size-limited. The solution is to use multiple intermediate matrices for in-between values rather than just the start and end positions. Additionally, ensure that the PF_Rect *dest_rect is expanded to encompass the full area of all rotated samples to capture the complete motion blur region.

```cpp
PF_Err transform_world (
  PF_InData *in_data,
  PF_Quality quality,
  PF_ModeFlags m_flags,
  PF_Field field,
  const PF_EffectWorld *src_world,
  const PF_CompositeMode *comp_mode,
  const PF_MaskWorld *mask_world0,
  const PF_FloatMatrix *matrices,
  A_long num_matrices,
  Boolean src2dst_matrix,
  const PF_Rect *dest_rect,
  PF_EffectWorld *dst_world);
```

*Tags: `mfr`, `motion-blur`, `output-rect`, `rendering`, `transform`*

---

### How can I access suite functions from within a pixel iteration function?

Use the refcon (reference context) argument of the iterate() function to pass a struct containing pointers to the suites and any other data you need. Define a struct with your suite references and other variables, pass a pointer to an instance of this struct as the refcon argument to iterate(), and then cast it back to your struct type inside the pixel function to access the suites.

```cpp
typedef struct {
  PF_in_data *in_data;
  AEGP_Suites &suites;
  bool someVariable;
  float someOtherVariable;
} myIterationData;

// In calling function
myIterationData data;
data.suites = &suites;
data.someVariable = FALSE;
suites.Iterate16Suite1()->iterate(
  arg,
  arg,
  arg,
  (void*)&data
);

// In pixel function
myIterationData &data = *(myIterationData*)refcon;
if(data.someVariable) {
  data.suites.whateverSuite()->
}
```

*Tags: `aegp`, `mfr`, `reference`, `render-loop`*

---

### When should I use PreRender and SmartRender in After Effects plugins, and what are their benefits?

PreRender (PF_Cmd_SMART_PRE_RENDER) and SmartRender (PF_Cmd_SMART_RENDER) are supported only in After Effects and are mandatory if you wish to support 32bpc (32-bit per channel) rendering. They also offer rendering pipeline improvements. However, if your plugin needs to work in Premiere Pro as well, you must support the old rendering pipeline with FrameSetup() and regular Render() calls instead.

*Tags: `ae`, `mfr`, `reference`, `render-loop`, `smart-render`*

---

### How can I optimize subpixel sampling performance in After Effects plugins?

To optimize subpixel sampling, acquire the sampling suite once before your rendering loop using AEFX_AcquireSuite, get a direct pointer to it, and release it afterwards. Avoid acquiring and releasing the suite for each sample operation. Alternatively, define the AEGP_SuitesHandler before the loop and pass a pointer to it to the sampling function, though acquiring the suite directly is likely faster since it reduces function call overhead.

*Tags: `aegp`, `mfr`, `performance`, `render-loop`, `sampling`*

---

### What is the difference between PF_SUBPIXEL_SAMPLE and subpixel_sample in After Effects plugins?

PF_SUBPIXEL_SAMPLE is a macro that wraps the in_data->utils->subpixel_sample() function. Both provide the same sampling functionality. The macro approach can be used even though some documentation may mark it as unsupported. The key performance difference is not in which function you use, but in how you manage suite acquisition—acquire the sampling suite once before your loop rather than repeatedly inside it.

*Tags: `aegp`, `mfr`, `params`, `sampling`*

---

### How can I iterate through 16-bit and 32-bit per channel pixels directly without using the iterate suites?

You can iterate 16bpc and 32bpc pixels directly by casting the buffer data to the appropriate pixel type. For 16-bit pixels, use: `PF_Pixel16 pix = (PF_Pixel16)buffer->data + xCount + yCount * (buffer->rowbytes / sizeof(Pixel16));`. Convert similarly for 32-bit pixels. The rest of the process is the same as in the CCU sample. Replace all PF_Pixel8 references with PF_Pixel16 or PF_Pixel32 as needed, and update constants like PF_MAX_CHAN8 to PF_MAX_CHAN16 (or 1.0f for 32-bit). For 16-bit support, set PF_OutFlag_DEEP_COLOR_AWARE in out_data->out_flags. For 32-bit support, use SmartFX (see the smartyPants sample).

```cpp
PF_Pixel16 pix = (PF_Pixel16)buffer->data + xCount + yCount * (buffer->rowbytes / sizeof(Pixel16));
```

*Tags: `memory`, `mfr`, `output-rect`, `render-loop`*

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

### How should I port a plug-in from CS4 to CS6 when PF_Context's cgrafptr and PF_GET_CGRAF_DATA macro are removed?

The drawing mechanism changed between CS4 and CS5. Direct drawing into a context is no longer supported. Instead, you must use the DrawBot suite to draw. The quickest conversion approach is to wrap the DrawBot commands under your old drawing command names. This change also makes the code cross-platform compatible.

*Tags: `cross-platform`, `mfr`, `porting`, `ui`, `windows`*

---

### How can I determine the bit depth and pixel format of input/output images in an After Effects plugin?

The easiest way to get the bit depth is using `extra->input->bitdepth` in the smart_render call, which returns 8, 16, or 32. Alternatively, use the PF_WorldSuite2 to call `PF_GetPixelFormat()` on the world. After Effects always uses ARGB pixel format. The pixel format enum returns values like PF_PixelFormat_ARGB128 or PF_PixelFormat_ARGB.

```cpp
PF_WorldSuite2 *wsP = NULL;
ERR(suites.Pica()->AcquireSuite(kPFWorldSuite, kPFWorldSuiteVersion2, (const void**)&wsP));
ERR(wsP->PF_GetPixelFormat(inputP, pixelFormat));
ERR(suites.Pica()->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2));
```

*Tags: `mfr`, `params`, `reference`, `sdk`, `smart-render`*

---

### How can I implement a custom pixel iteration function similar to PF_Iterate8Suite1::iterate?

To implement a custom iteration function, start by examining the 'shifter' sample which implements one of the iteration functions. For direct pixel data access without the iteration suite, consult the 'CCU' sample, particularly its render function. You can also use the iterate_generic() function to obtain threading services without additional functionality. For optimal performance, thread rows rather than individual pixels to reduce CPU overhead per pixel.

*Tags: `mfr`, `pixel-iteration`, `reference`, `render-loop`, `threading`*

---

### What sample plugins demonstrate pixel iteration and direct pixel data access in After Effects?

The 'shifter' and 'CCU' samples are recommended references. The 'shifter' sample implements iteration functions, while the 'CCU' sample demonstrates how to access pixel data directly without using the iteration suite, particularly in its render function implementation.

*Tags: `mfr`, `open-source`, `reference`, `render-loop`*

---

### How can you re-apply a mask after modifying the alpha channel in an After Effects plugin?

There are two main approaches: (1) Use PF_MaskWorldWithPath on each mask and composite them in the correct transfer mode to recreate the layer's mask matte. Note that PF_MaskWorldWithPath does not support mask expansion, so you must render expanded masks manually. Read feather, opacity, and other mask values using AEGP_GetNewMaskStream or AEGP_GetNewMaskOpacity, then access numeric values with AEGP_GetNewStreamValue. (2) A simpler alternative: checkout the layer's original pixels (without masks applied) using a hidden layer parameter and PF_CHECKOUT_PARAM (or checkout_layer for smartFX), then compare the alpha difference between the original and masked input pixels to derive the mask matte. The second method is easier but less precise than fully recreating masks.

*Tags: `aegp`, `layer-checkout`, `mfr`, `params`, `smartfx`*

---

### How do you access pixel data from multiple layers in an After Effects effect plugin?

To access pixel data from multiple layers in an effect plugin, you need to create a layer parameter (e.g., param number 3) that allows the user to select any layer from the composition. Then use PF_CHECKOUT_PARAM to checkout that parameter. The pixel data can be accessed from the resulting paramDef structure at paramDef->u.ld.data (or similar location in the structure). This approach allows an effect on one layer to access and process pixels from any other layer in the composition.

```cpp
paramDef->u.ld.data
```

*Tags: `aegp`, `layer-checkout`, `mfr`, `params`*

---

### How do you properly use the transform_world() function to scale and rotate a PF_LayerDef?

To use transform_world() effectively: (1) Pass NULL for mask_world0 to use the whole frame. (2) You can pass one or two matrices—two matrices enable motion blur between transformations. Use syntax (&matrix1, &matrix2) for two matrices or &matrix1 for one, and specify the matrix count. (3) Note that AE's 3x3 matrices have swapped x and y compared to standard math notation—what should be matrix[1][2] goes in matrix[2][1]. See the CCU sample for identity and transformation matrix functions. (4) Always pass a full-frame rectangle initially; transform_world() does not accept NULL for the destination rectangle. The offset should be embedded in the matrix itself, not assumed from the rectangle's top-left corner.

*Tags: `matrix`, `mfr`, `params`, `reference`, `transform`*

---

### Where can I find sample code for creating and manipulating transformation matrices in After Effects plugins?

The CCU sample included in the After Effects SDK contains helper functions for creating identity matrices and concatenating them with scale and rotation transformations. This sample demonstrates proper matrix manipulation for use with transform_world() and similar functions.

*Tags: `matrix`, `mfr`, `reference`, `sample`, `sdk`*

---

### How do you safely perform one-time lazy global initialization under MFR?

Under MFR, AE can enter `PF_Cmd_SMART_PRE_RENDER` / `PF_Cmd_SMART_RENDER` from several worker threads at once. A boolean-guarded lazy initializer races: two threads can both pass the `if (!g_initialized)` check before either sets it, so both run the init body and write the same global state concurrently -- corruption or an intermittent crash that only appears with MFR enabled and multiple render threads.

The symptom is non-deterministic: it never reproduces single-threaded, and may take many renders to surface.

The fix is **double-checked locking** -- a lock-free fast path once initialized, a mutex only during the contended startup window:

```cpp
static std::mutex g_init_mutex;
static bool       g_initialized = false;

static void EnsureInitialized() {
    if (g_initialized) return;                       // fast path: no lock after warm-up
    std::lock_guard<std::mutex> lk(g_init_mutex);
    if (g_initialized) return;                       // re-check inside the lock
    // ... populate global state exactly once ...
    g_initialized = true;
}
```

Any global state that is lazily initialized and then touched from render threads must be synchronized this way. Always test with MFR turned on (Preferences > Memory & Performance > Enable Multi-Frame Rendering) to surface these races. And declare your render safety accurately -- if the plugin has shared mutable state, prefer the instance-safe or unsafe render-safety level over fully-safe.

*Tags: `mfr`, `threading`, `initialization`, `race-condition`, `mutex`*

---
