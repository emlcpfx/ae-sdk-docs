# tlafo

**34 contributions** to AE SDK community knowledge.

Top topics: `premiere`, `smart-render`, `pipl`, `xcode`, `mac`, `aegp`, `gpu`, `checkout-layer`, `memory-leak`, `debugging`

---

## How do you debug the 'plugin wasn't installed correctly' error on Windows?

Use dumpbin (included with Visual C++) to check plugin dependencies: 'dumpbin /dependents plugin.aex'. Common causes: (1) Missing Visual C++ Redistributable packages (download from https://aka.ms/vs/17/release/vc_redist.x64.exe). (2) Missing DLL dependencies. (3) Error in GlobalSetup. (4) Debug builds accidentally linking to debug CRT DLLs (msvcp140d.dll). Try setting a breakpoint on GlobalSetup in debug mode to see if the entry point is reached.

```cpp
dumpbin /dependents plugin.aex
```

*Source: adobe-plugin-devs · 2025-10-16 · Tags: `debugging`, `windows`, `dependencies`, `dumpbin`, `vcredist`, `installation` · [View in Q&A](../qa/debugging/)*

---

## What causes artifacts in 16/32-bit mode when a Track Matte is applied?

The root causes are typically: (1) Incorrect bit depth detection causing memory corruption - using wrong pixel sizes for memory operations. (2) Improper stride/rowbytes handling for padded buffers - when a track matte is applied, the layer gets cropped to a different size, changing the relationship between width and rowbytes. Fix bit depth detection to use rowbytes calculation and implement proper stride handling throughout your processing algorithm. The effect works in 8-bit because the rowbytes happen to align, but in 16/32-bit the padding differences cause corruption.

*Source: adobe-plugin-devs · 2025-10-15 · Tags: `track-matte`, `bit-depth`, `16-bit`, `32-bit`, `rowbytes`, `memory-corruption`, `artifacts` · [View in Q&A](../qa/track-matte/)*

---

## What is PF_REGISTER_EFFECT and will it replace PiPL?

AE's internal plugins now use two entry points: EffectMainExtra and PluginDataEntryFunction, defined via PF_REGISTER_EFFECT / PF_Register_effect_ext2. This appears in SDK samples like SDK_Invert_ProcAmp. While it seems like it could eventually replace PiPL, as of now PiPL is still required for AE - attempting to build without a PiPL/rsrc still results in AE not finding the effect. Premiere Pro no longer requires PiPL and uses this newer system. No official announcement about PiPL deprecation has been made.

*Source: adobe-plugin-devs · 2025-06-05 · Tags: `pipl`, `pf-register-effect`, `plugin-entry-point`, `future`, `premiere` · [View in Q&A](../qa/pipl/)*

---

## How does GLator (deprecated OpenGL sample) have memory leaks?

The GLator sample wraps GL rendering in try/catch. The suites.IterateFloatSuite1()->iterate call can throw PF_Interrupt_CANCEL in the download texture function, causing bufferH to never be deallocated. This leaks an entire output buffer of 32bpc pixels per frame from CPU, plus GPU FBO resources. Using scope guards (C++ RAII cleanup on scope exit) prevents this pattern - they handle cleanup regardless of how the scope is exited (exception, return, etc.).

*Source: adobe-plugin-devs · 2025-04-18 · Tags: `glator`, `opengl`, `memory-leak`, `exception-handling`, `scope-guard` · [View in Q&A](../qa/glator/)*

---

## Why might SmartRender never be called even though PreRender returns PF_Err_NONE?

Possible causes: (1) GuidMixInPtr in PreRender is telling AE 'no new render needed' if the mixed-in data hasn't changed. (2) Mismatched SDK versions - building with 2025.2 SDK source but 2023 SDK headers causes undefined behavior, especially in Release builds where optimizations may expose the mismatch. Always ensure your SDK headers and source files are from the same version.

*Source: adobe-plugin-devs · 2025-04-09 · Tags: `smart-render`, `pre-render`, `guid-mixin`, `sdk-version`, `release-build` · [View in Q&A](../qa/smart-render/)*

---

## How do you use dynamic dropdown lists (POPUP params) with changing options?

Modifying popup options at runtime requires updating the names pointer string during the UpdateParamsUI thread. One approach uses strncpy_s to modify param_union.pd.u.namesptr. However, this broke in AE CC2025.2 (empty dropdown, crash on click). An alternative approach uses AEGP stream suites to set the dropdown value programmatically via AEGP_SetStreamValue, though this causes undo history issues. This is done in UpdateParamsUI.

```cpp
PF_Err setFloatOrBoolOrDropdownParamViaAEGP(uint16_t i_param_to_change, float val,
    AEGP_PluginID my_id, AEGP_EffectRefH *effect_refHP, AEGP_SuiteHandler *suitesP)
{
    PF_Err err = PF_Err_NONE;
    AEGP_StreamValue streamvalue;
    streamvalue.val.one_d = val;
    AEGP_StreamRefH param_refH = NULL;
    ERR(suitesP->StreamSuite2()->AEGP_GetNewEffectStreamByIndex(my_id, *effect_refHP, i_param_to_change, &param_refH));
    streamvalue.streamH = param_refH;
    ERR(suitesP->StreamSuite2()->AEGP_SetStreamValue(my_id, param_refH, &streamvalue));
    ERR(suitesP->StreamSuite5()->AEGP_DisposeStream(param_refH));
    return err;
}
```

*Source: adobe-plugin-devs · 2025-04-03 · Tags: `popup`, `dropdown`, `dynamic-params`, `aegp-stream`, `update-params-ui` · [View in Q&A](../qa/popup/)*

---

## How do you read sequence data in SmartRender since AE CC2021/2022?

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

*Source: adobe-plugin-devs · 2025-04-02 · Tags: `sequence-data`, `smart-render`, `mfr`, `pf-get-const-sequence-data`, `threading` · [View in Q&A](../qa/sequence-data/)*

---

## How do you get the AE/Premiere version number programmatically?

in_data->version major/minor are not reliably updated between AE versions. Options: (1) Use ExtendScript via AEGP_ExecuteScript with 'app.version' - but may fail with 'cannot run script while modal dialog waiting'. (2) Use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP to get the AE install folder path, then parse the version from the folder name or binary metadata. (3) For plugins with Premiere GPU entry points, PPixSuite's appinfo provides version data (format like '24.3', add 2000 for year).

*Source: adobe-plugin-devs · 2025-03-25 · Tags: `version-detection`, `aegp`, `extendscript`, `premiere`, `compatibility` · [View in Q&A](../qa/version-detection/)*

---

## What causes the 'Not able to acquire AEFX Suite' error in Premiere?

This error can occur during playback in Premiere when code attempts to acquire an AE-specific suite that is not available in the Premiere host. Check if you have a shared function that conditionally calls AEFX suites based on host detection (e.g., if app_id != 'PrMr' then call AEFX suite). The host detection condition might be returning the wrong result, causing Premiere to try acquiring AE-only suites. It could also be a bug with a Premiere suite itself.

*Source: adobe-plugin-devs · 2024-09-18 · Tags: `premiere`, `aefx-suite`, `host-detection`, `suite-acquisition`, `error` · [View in Q&A](../qa/premiere/)*

---

## Is checking out the current layer with prior effects applied possible in Premiere?

In AE, you can use checkout_layer_pixels in SmartRender to get the layer with prior effects applied. However, this does not work in Premiere. In Premiere, there is no straightforward way to check out the current layer with prior effects applied - PF_CHECKOUT_PARAM always returns the pre-effects layer. A potential hacky workaround might involve calling Premiere's JS scripting API to render, but this is not a clean solution.

*Source: adobe-plugin-devs · 2024-06-19 · Tags: `premiere`, `checkout-layer`, `prior-effects`, `smart-render`, `limitations` · [View in Q&A](../qa/premiere/)*

---

## How do you properly copy pixel data from OpenCV cv::Mat to AE PF_LayerDef?

Never manually allocate layerDef->data - AE owns that memory. Copy pixel data respecting rowbytes alignment. Use the sampleIntegral32 pattern to correctly index pixels. AE uses ARGB format while OpenCV uses BGRA, so channel swizzling is needed. When creating a cv::Mat pointing to AE layer data, pass layerDef->rowbytes as the step parameter. For performance, use memcpy per row (if same pixel format) and the IterateSuite for multi-threaded processing.

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y) {
    return (PF_Pixel*)((char*)def.data +
        (y * def.rowbytes) +
        (x * sizeof(PF_Pixel)));
}

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

// Create cv::Mat from AE layer with proper stride:
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Source: adobe-plugin-devs · 2024-06-09 · Tags: `opencv`, `pixel-format`, `argb`, `bgra`, `rowbytes`, `cv-mat` · [View in Q&A](../qa/opencv/)*

---

## When is PF_Cmd_GLOBAL_SETUP called - during AE loading or when applying the effect?

In After Effects, GlobalSetup is called the first time the user applies the effect to a layer, not during the AE loading screen (during loading, AE only scans PiPLs). In Premiere Pro, GlobalSetup is called when the app is loading. You can show dialogs during GlobalSetup in AE since it happens after the app is fully loaded.

*Source: adobe-plugin-devs · 2024-05-30 · Tags: `global-setup`, `plugin-lifecycle`, `premiere`, `initialization` · [View in Q&A](../qa/global-setup/)*

---

## Can you have multiple effect plugins in one .aex file?

The SDK documentation says it's possible using multiple PiPLs, with AEGPs placed first. However, it's not recommended because: (1) No other hosts including Premiere Pro support multiple PiPLs. (2) If you need to update one plugin, you'd have to ship a new build of all plugins. In practice, adding another PiPL in Xcode with a different name/matchname may only show one plugin. Adobe recommends one PiPL per code fragment.

*Source: adobe-plugin-devs · 2024-05-28 · Tags: `pipl`, `multiple-plugins`, `aex`, `architecture` · [View in Q&A](../qa/pipl/)*

---

## What is plugplug.DLL and how can you use it from C++ plugins?

plugplug.DLL is the gateway to communicate with CEP (Common Extensibility Platform) from C++. It allows sending events between C++ plugins and CEP panels. You can reverse-engineer the DLL's exports using depends.exe and reference the Illustrator SDK sample as a guide (it's not 1:1 but close enough). A GitHub repo with a working implementation is available: https://github.com/Trentonom0r3/AE-SDK-CEP-UTILS

*Source: adobe-plugin-devs · 2024-05-02 · Tags: `plugplug`, `cep`, `inter-process-communication`, `dll` · [View in Q&A](../qa/plugplug/)*

---

## What is the Compute Cache API and how does it differ from sequence data and arb data?

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

*Source: adobe-plugin-devs · 2024-02-27 · Tags: `compute-cache`, `mfr`, `sequence-data`, `arb-data`, `caching`, `thread-safety` · [View in Q&A](../qa/compute-cache/)*

---

## Why does Media Encoder not render plugins correctly or fail to load AEGP plugins?

Media Encoder does not load AEGP plugins. If your effect plugin relies on a companion AEGP plugin for rendering, it will fail in Media Encoder. Issues may also be related to static global variables or elements defined in GlobalData/GlobalSetup. The AEGP functionality would need to be incorporated directly into the effect plugin, or you'd need a standalone/command-line tool alternative.

*Source: adobe-plugin-devs · 2024-02-27 · Tags: `media-encoder`, `aegp`, `render-issues`, `global-data` · [View in Q&A](../qa/media-encoder/)*

---

## How do you check out a frame with prior effects applied (not the raw layer)?

Use checkout_layer_pixels (from PF_SmartRenderCallbacks in SmartRender) instead of PF_CHECKOUT_PARAM. PF_CHECKOUT_PARAM always returns the layer before all effects were applied. checkout_layer_pixels returns the frame with all prior effects applied. You must first checkout the layer in PreRender, then checkout_layer_pixels in SmartRender.

*Source: adobe-plugin-devs · 2024-02-02 · Tags: `smart-render`, `checkout-layer`, `pre-render`, `effect-stack`, `frame-checkout` · [View in Q&A](../qa/smart-render/)*

---

## Why do keyframed parameters on adjustment layers in Premiere always return the value at the first keyframe regardless of current time?

This is caused by the PF_OutFlag_NON_PARAM_VARY flag. When this flag is set, keyframes on adjustment layers in Premiere always return the first keyframe value during checkout param, regardless of current time. This affects sliders, angles, checkboxes, and 2D points. The bug occurs in Premiere v23 and v24. Removing the NON_PARAM_VARY flag fixes the issue. In AE, this flag can be replaced using MIX_GUID during the pre_render thread. Note that apart from this bug, the flag does nothing useful in Premiere.

*Source: adobe-plugin-devs · 2024-02-01 · Tags: `premiere`, `keyframes`, `adjustment-layer`, `non-param-vary`, `checkout-param`, `bug` · [View in Q&A](../qa/premiere/)*

---

## What is the best practice when a plugin runs out of GPU memory during rendering?

Returning PF_Err_OUT_OF_MEMORY may cause AE to show a black frame without warning the user. Options: (1) Use a C++ text library to render 'Out of GPU memory' text directly onto the frame. (2) Set a static global boolean (protected by mutex) in the render thread, then display the warning in another thread like PF_UpdateParamUI. Avoid using out_data warning messages during renders as they would fail render queue operations and prevent MFR from retrying with fewer threads.

*Source: adobe-plugin-devs · 2023-11-30 · Tags: `gpu`, `out-of-memory`, `error-handling`, `mfr`, `render-queue` · [View in Q&A](../qa/gpu/)*

---

## What causes slow AE loading when debugging from Xcode 15?

After upgrading to macOS 14/Xcode 15, AE loading becomes ~3x slower due to 'dlsym cannot find symbol xSDKExport' messages for MediaCore bundles. This is Xcode trying to load symbols for all loaded bundles. Workarounds: (1) Run Xcode 14 binary directly from Terminal. (2) Open AE standalone first, then attach the debugger after AE is fully loaded. (3) Check if 'load symbols' option is accidentally selected in your IDE settings.

*Source: adobe-plugin-devs · 2023-11-29 · Tags: `xcode`, `debugging`, `performance`, `macos`, `symbol-loading` · [View in Q&A](../qa/xcode/)*

---

## Why can't macOS Instruments detect PF_EffectWorld memory leaks?

Instruments won't report PF_EffectWorld leaks because AE legitimately thinks you're holding important data in those worlds. AE allocates the worlds deep in its engine, not directly in your plugin code. While memory allocated with new/malloc within the plugin gets reported accurately (because Instruments has more context), AE-managed allocations are invisible to the leak detector. Creating C++ RAII wrappers around AE SDK objects for automatic memory management is recommended.

*Source: adobe-plugin-devs · 2023-11-22 · Tags: `memory-leak`, `instruments`, `mac`, `debugging`, `effect-world`, `raii` · [View in Q&A](../qa/memory-leak/)*

---

## How do you restrict two float sliders relative to each other (e.g., Start cannot be >= End)?

Modifying param values in UserChangedParam works but causes problems with keyframing: a keyframe with an acceptable value can later become unacceptable, locking the value and preventing keyframe deletion. A better approach is to use the min()/max() pattern: treat the parameters as generic 'endpoints' and in your render code use min(A,B) as Start and max(A,B) as End.

*Source: adobe-plugin-devs · 2023-10-02 · Tags: `param-constraints`, `sliders`, `user-changed-param`, `keyframing` · [View in Q&A](../qa/param-constraints/)*

---

## Why is checkout_layer_pixels returning data == NULL for a layer parameter in SmartRender?

In GPU mode (CUDA, Metal, OpenCL), the SDK documentation states that you must assume the data pointer is NULL in PF_EffectWorld. The pixel data lives on the GPU, not in CPU memory. This is expected behavior when GPU rendering is active.

*Source: adobe-plugin-devs · 2023-09-19 · Tags: `smart-render`, `gpu`, `checkout-layer`, `null-data`, `effect-world` · [View in Q&A](../qa/smart-render/)*

---

## What is the AEGP memory leak issue on Mac during export with Compute Cache?

When using AEGP_MemorySuite to create memHandles during ComputeCache threads, the memory may not be properly freed during export on Mac (works fine on Windows). The used memory grows past the RAM limit. Using new/delete instead of AEGP memHandles avoids the leak. The issue is that memHandles allocated in one thread and freed in another may not actually release memory during the render thread. The AEGP tools report the memory as freed, but virtual memory keeps growing.

*Source: adobe-plugin-devs · 2023-09-07 · Tags: `memory-leak`, `compute-cache`, `mac`, `export`, `aegp-memory-suite`, `threading` · [View in Q&A](../qa/memory-leak/)*

---

## Why does Premiere give a NULL input_world in the Render call when compiled with fast optimizations (-Ofast), but works without optimizations?

This is a known issue that occurs in Premiere but not AE when using -Ofast compiler optimizations. A workaround is to add __attribute__((optnone)) before the render function to disable optimization for that specific function while keeping the rest of the plugin optimized. Another suggestion is to check if the input world is null and return PF_Err_OUT_OF_MEMORY (not bad param), which may cause Premiere to retry the render. Also verify your colorspace setup in GlobalSetup and consider trying BGRA instead of ARGB.

```cpp
__attribute__ ((optnone))
static PF_Err Render(...) {
    // render code here
}
```

*Source: adobe-plugin-devs · 2023-09-07 · Tags: `premiere`, `null-input-world`, `optimization`, `ofast`, `compiler`, `render`, `workaround` · [View in Q&A](../qa/premiere/)*

---

## How do you build AE plugins for Apple Silicon (ARM64)?

You can build on an M1 Mac (even a cheap Mac Mini works as a build machine). It's not strictly required to build on M1, but it helps for testing. The key thing many people miss is that you must add ARM64 to your PiPL resource file - just building as Universal Binary in Xcode is not enough. Without the ARM64 entry in the PiPL, AE will show the 'not yet compatible' warning.

*Source: adobe-plugin-devs · 2023-08-18 · Tags: `apple-silicon`, `arm64`, `pipl`, `xcode`, `mac`, `universal-binary` · [View in Q&A](../qa/apple-silicon/)*

---

## How do you pass custom data structs to GPU kernels in AE plugins?

Define the struct in both the kernel code and your C++ header, then use void pointers to transfer the data from AE to the kernel. Pointers work with CUDA. For Metal, the approach may differ. For OpenCL it works with a few caveats.

*Source: adobe-plugin-devs · 2023-07-29 · Tags: `gpu`, `cuda`, `metal`, `opencl`, `kernel`, `custom-struct` · [View in Q&A](../qa/gpu/)*

---

## Does Premiere support SmartFX for requesting only a portion of the input buffer?

Premiere does not support SmartFX. You cannot officially request only a specific region of the buffer like you can in AE. As a workaround, you can manually iterate through the data using row bytes to delimit the part you need. Be careful with negative row byte values in Premiere.

*Source: adobe-plugin-devs · 2023-07-13 · Tags: `premiere`, `smartfx`, `buffer`, `region-of-interest`, `row-bytes`, `transition` · [View in Q&A](../qa/premiere/)*

---

## Are there CMake-based build systems for AE plugins?

Yes, there are several examples on GitHub: Vulkanator (https://github.com/Wunkolo/Vulkanator) uses CMake, and after_effects_cmake (https://github.com/mobile-bungalow/after_effects_cmake) is based on Vulkanator's setup. There's also a Rust-based project (https://github.com/virtualritz/after-effects) with its own PiPL compiler written in Rust. On Mac, you can use the Rez tool for .rsrc files and potentially use Ninja for faster incremental compilation.

*Source: adobe-plugin-devs · 2023-07-06 · Tags: `cmake`, `build-system`, `xcode`, `ninja`, `cross-platform` · [View in Q&A](../qa/cmake/)*

---

## How can I use multithreading for pixel iteration in Premiere, since the AE Iterate8Suite doesn't work correctly?

The AE Iterate8Suite may not work correctly in Premiere and can cause rendering issues (e.g., sequence thumbnail updates but program window doesn't). For 8-bit BGRA, Premiere has its own dedicated iteration suite. For 32-bit float, you need to write your own iteration function. Use std::parallel or equivalent for multithreading. The SDK Noise example demonstrates BGRA and VUYA colorspace handling. The iterate suite should work in 8-bit ARGB (Suitev2 since 2022), but for BGRA you need the special Premiere suite.

*Source: adobe-plugin-devs · 2023-03-22 · Tags: `premiere`, `multithreading`, `iterate-suite`, `bgra`, `pixel-iteration`, `colorspace` · [View in Q&A](../qa/premiere/)*

---

## How should I handle color spaces in Premiere plugins, and can I avoid dealing with multiple color modes?

You can use basic render in 8/16-bit without handling all color spaces -- it works but is not optimized, and you won't get the 32-bit icon next to the plugin. You can choose to support only BGRA and skip VUYA. In GlobalSetup, you select which colorspaces to support. However, 32-bit float is required for color grading workflows to access values outside the 0-1 range. The SDKNoise example in the AE SDK demonstrates both BGRA and VUYA colorspace handling for Premiere.

*Source: adobe-plugin-devs · 2023-03-22 · Tags: `premiere`, `colorspace`, `bgra`, `vuya`, `32-bit`, `global-setup`, `color-grading` · [View in Q&A](../qa/premiere/)*

---

## How do I clear Premiere's plugin cache to fix rendering or display glitches?

Hold Shift or Alt when opening Premiere to clean the plugin cache. This has been known to solve problems where rendering appears incorrect or display issues occur.

*Source: adobe-plugin-devs · 2023-03-22 · Tags: `premiere`, `cache`, `plugin-cache`, `troubleshooting` · [View in Q&A](../qa/premiere/)*

---

## Why can I only select the first two items in a PF_ADD_POPUPX dropdown with 3 elements in Premiere, and the third snaps back to 2?

This can be caused by opening a Premiere project with the transition already applied (stale cached state). Applying the transition plugin fresh resolves the issue and the dropdowns work normally. Also ensure that each popup choice string has the pipe separator '|' at the end of each line (except the last), and that you have a proper AEFX_CLR_STRUCT(def) before the popup definition.

```cpp
AEFX_CLR_STRUCT(def);

PF_ADD_POPUPX("Color", 3, 2, 
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Source: adobe-plugin-devs · 2022-12-17 · Tags: `premiere`, `popup`, `dropdown`, `parameter`, `transition`, `cache-bug` · [View in Q&A](../qa/premiere/)*

---

## How do you toggle parameter visibility (show/hide) in an AE plugin?

Use AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to toggle visibility. The approach using PF_PUI_INVISIBLE with ui_flags and PF_UpdateParamUI is unreliable. Instead, use the AEGP stream suites: get the effect ref via AEGP_GetNewEffectForEffect, get the stream via AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag. Note this only works in AE (check in_data->appl_id != 'PrMr'), not Premiere.

```cpp
static PF_Err
changeParamVisibility(PF_InData *in_data, PF_OutData *out_data,
                      PF_ParamDef *paramsDef, PF_ParamIndex paramIndex,
                      PF_Boolean paramVisibleB)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    global_dataP globP = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr') {
        AEGP_EffectRefH meH = NULL;
        AEGP_StreamRefH currStreamH = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex, &currStreamH));
        if (meH && currStreamH) {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
    }
    return err;
}
```

*Source: adobe-plugin-devs · 2022-12-07 · Tags: `param-visibility`, `aegp`, `dynamic-stream`, `custom-ui`, `pipl` · [View in Q&A](../qa/param-visibility/)*

---
