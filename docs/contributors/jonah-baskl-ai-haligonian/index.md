# Jonah (Baskl.ai/Haligonian)

**15 contributions** to AE SDK community knowledge.

Top topics: `debugging`, `pipl`, `xcode`, `premiere`, `mac`, `arb-data`, `mfr`, `performance`, `macos`, `optimization`

---

## What is PF_REGISTER_EFFECT and will it replace PiPL?

AE's internal plugins now use two entry points: EffectMainExtra and PluginDataEntryFunction, defined via PF_REGISTER_EFFECT / PF_Register_effect_ext2. This appears in SDK samples like SDK_Invert_ProcAmp. While it seems like it could eventually replace PiPL, as of now PiPL is still required for AE - attempting to build without a PiPL/rsrc still results in AE not finding the effect. Premiere Pro no longer requires PiPL and uses this newer system. No official announcement about PiPL deprecation has been made.

*Source: adobe-plugin-devs · 2025-06-05 · Tags: `pipl`, `pf-register-effect`, `plugin-entry-point`, `future`, `premiere` · [View in Q&A](../qa/pipl/)*

---

## How do you debug AE plugins built with Rust on macOS?

Use VSCode with the CodeLLDB extension. Create a launch.json that launches AE directly and specifies 'sourceLanguages': ['rust']. The sourceLanguages line is important - without it, launching AE from LLDB can cause crashes. Use a preLaunchTask to build the plugin before launching.

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build",
      "sourceLanguages": ["rust"]
    }
  ]
}
```

*Source: adobe-plugin-devs · 2025-04-18 · Tags: `rust`, `debugging`, `lldb`, `vscode`, `macos` · [View in Q&A](../qa/rust/)*

---

## Can AE plugins target AVX2 instruction set for better performance?

AE has required AVX2 since version 2023, and Steam surveys show 96%+ CPU support for AVX2. It's reasonable to target x86-64-v3 (AVX2) instead of v2 (SSSE3), which could double performance for SIMD-heavy processing. ISPC is a good option for automatic multi-ISA compilation (compiles multiple versions and auto-switches at runtime). Alternatively, use runtime detection with if(HasAVX()) dispatch, though this adds complexity.

*Source: adobe-plugin-devs · 2025-03-19 · Tags: `avx2`, `simd`, `performance`, `ispc`, `instruction-set`, `optimization` · [View in Q&A](../qa/avx2/)*

---

## Why do rowbytes sometimes differ from width * bytes_per_pixel?

Rowbytes can be larger than expected because AE uses it as an optimization for layer cropping. When a layer is cropped (e.g., dragged partially off the comp), AE updates width/height and moves the data pointer to the new top-left pixel, but leaves rowbytes unchanged. This avoids memory reallocation. After a cache purge, rowbytes may return to the expected value. Always use rowbytes (not width * pixel_size) when iterating rows.

*Source: adobe-plugin-devs · 2024-12-27 · Tags: `rowbytes`, `memory-layout`, `cropping`, `pixel-access`, `buffer-stride` · [View in Q&A](../qa/rowbytes/)*

---

## How do you store a struct in global_data?

Create a pointer to the struct in your global data type, then in GlobalSetup allocate it with new: globaldata->structPointer = new StructName(). Use new/malloc (not smart pointers) since AE does special things with the global_data pointer. Optionally delete it in GlobalSetdown, but it's not strictly necessary since GlobalSetdown only gets called when AE closes and the OS will clean up.

*Source: adobe-plugin-devs · 2024-09-03 · Tags: `global-data`, `memory-management`, `global-setup`, `global-setdown` · [View in Q&A](../qa/global-data/)*

---

## Is checking out the current layer with prior effects applied possible in Premiere?

In AE, you can use checkout_layer_pixels in SmartRender to get the layer with prior effects applied. However, this does not work in Premiere. In Premiere, there is no straightforward way to check out the current layer with prior effects applied - PF_CHECKOUT_PARAM always returns the pre-effects layer. A potential hacky workaround might involve calling Premiere's JS scripting API to render, but this is not a clean solution.

*Source: adobe-plugin-devs · 2024-06-19 · Tags: `premiere`, `checkout-layer`, `prior-effects`, `smart-render`, `limitations` · [View in Q&A](../qa/premiere/)*

---

## Can you have multiple effect plugins in one .aex file?

The SDK documentation says it's possible using multiple PiPLs, with AEGPs placed first. However, it's not recommended because: (1) No other hosts including Premiere Pro support multiple PiPLs. (2) If you need to update one plugin, you'd have to ship a new build of all plugins. In practice, adding another PiPL in Xcode with a different name/matchname may only show one plugin. Adobe recommends one PiPL per code fragment.

*Source: adobe-plugin-devs · 2024-05-28 · Tags: `pipl`, `multiple-plugins`, `aex`, `architecture` · [View in Q&A](../qa/pipl/)*

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

## What is the best practice when a plugin runs out of GPU memory during rendering?

Returning PF_Err_OUT_OF_MEMORY may cause AE to show a black frame without warning the user. Options: (1) Use a C++ text library to render 'Out of GPU memory' text directly onto the frame. (2) Set a static global boolean (protected by mutex) in the render thread, then display the warning in another thread like PF_UpdateParamUI. Avoid using out_data warning messages during renders as they would fail render queue operations and prevent MFR from retrying with fewer threads.

*Source: adobe-plugin-devs · 2023-11-30 · Tags: `gpu`, `out-of-memory`, `error-handling`, `mfr`, `render-queue` · [View in Q&A](../qa/gpu/)*

---

## What causes slow AE loading when debugging from Xcode 15?

After upgrading to macOS 14/Xcode 15, AE loading becomes ~3x slower due to 'dlsym cannot find symbol xSDKExport' messages for MediaCore bundles. This is Xcode trying to load symbols for all loaded bundles. Workarounds: (1) Run Xcode 14 binary directly from Terminal. (2) Open AE standalone first, then attach the debugger after AE is fully loaded. (3) Check if 'load symbols' option is accidentally selected in your IDE settings.

*Source: adobe-plugin-devs · 2023-11-29 · Tags: `xcode`, `debugging`, `performance`, `macos`, `symbol-loading` · [View in Q&A](../qa/xcode/)*

---

## How do you run an older version of Xcode on a newer macOS?

Download older Xcode versions from https://developer.apple.com/download/all/. Run the binary directly from Terminal: '/Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode'. The .app bundle won't open normally on newer macOS, but running the binary directly bypasses this restriction. For quick access, add the /Contents/MacOS folder to Finder sidebar.

*Source: adobe-plugin-devs · 2023-11-28 · Tags: `xcode`, `mac`, `debugging`, `development-environment` · [View in Q&A](../qa/xcode/)*

---

## What causes uninitialized PF_Err variables and subtle bugs?

A common C++ syntax error: 'PF_Err err, err2 = PF_Err_NONE;' only initializes err2, leaving err undefined (often ends up as 1). The correct form is 'PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;'. This can cause if(!err) checks to fail unexpectedly and lead to hard-to-track bugs like AEGP_GetNewEffectStreamByIndex returning 0x0.

```cpp
// WRONG - err is uninitialized:
PF_Err err, err2 = PF_Err_NONE;

// CORRECT - both initialized:
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Source: adobe-plugin-devs · 2023-10-05 · Tags: `debugging`, `pf-err`, `initialization`, `c-plus-plus`, `common-mistake` · [View in Q&A](../qa/debugging/)*

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

## How do you implement a text editor in the Effect Control Window of an AE plugin?

Instead of trying to handle PF_Event_KEYDOWNs directly (which has limitations like missing Tab key and no param index), use a button that runs a script via AEGP_ExecuteScript(). The script string can display a text box dialog. Pass data to it by find+replacing in the script string before execution. Store the text in an arb data parameter. The script can return data back to AE.

*Source: adobe-plugin-devs · 2023-07-31 · Tags: `custom-ui`, `text-input`, `aegp-execute-script`, `arb-data`, `button-param` · [View in Q&A](../qa/custom-ui/)*

---
