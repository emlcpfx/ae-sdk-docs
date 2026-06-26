# gabgren

**25 contributions** to AE SDK community knowledge.

Top topics: `xcode`, `debugging`, `premiere`, `mfr`, `pipl`, `thread-safety`, `apple-silicon`, `mac`, `build-system`, `arb-data`

---

## Where can I find resources and community support for developing OFX plugins for Flame on Linux?

There are several resources: (1) The official OpenFX Slack channel (now under ASWF - Academy Software Foundation) is the recommended place, though access may be restricted to specific email domains like linuxfoundation.org or aswf.io. (2) There is an unofficial OFX Discord server with a dedicated Flame channel, though it's not very active. (3) OpenFX also has a mailing list you can join. (4) You can request a free developer license from Autodesk for Flame testing. For DaVinci Resolve OFX debugging, Maxon has published documentation that may also be helpful.

*Source: adobe-plugin-devs · 2026-02-09 · Tags: `ofx`, `flame`, `linux`, `openfx`, `autodesk`, `resolve`, `community` · [View in Q&A](../qa/ofx/)*

---

## How do you set per-host out_flags when supporting both AE and Premiere?

PiPL doesn't support per-host flags. Instead, set different flags in GlobalSetup by checking in_data->appl_id. For example, to use PF_OutFlag_NON_PARAM_VARY only in AE: check if(in_data->appl_id != 'PrMr') before setting the flag. Note: PiPLs are no longer used by Premiere Pro, so you can write PiPL for AE only. Premiere uses PluginDataEntryFunction / PF_REGISTER_EFFECT instead.

```cpp
if (in_data->appl_id != 'PrMr') {
    out_data->out_flags = AE_OUT_FLAGS;
} else {
    out_data->out_flags = PR_OUT_FLAGS;
}
```

*Source: adobe-plugin-devs · 2025-06-04 · Tags: `pipl`, `out-flags`, `premiere`, `per-host`, `global-setup` · [View in Q&A](../qa/pipl/)*

---

## Why might SmartRender never be called even though PreRender returns PF_Err_NONE?

Possible causes: (1) GuidMixInPtr in PreRender is telling AE 'no new render needed' if the mixed-in data hasn't changed. (2) Mismatched SDK versions - building with 2025.2 SDK source but 2023 SDK headers causes undefined behavior, especially in Release builds where optimizations may expose the mismatch. Always ensure your SDK headers and source files are from the same version.

*Source: adobe-plugin-devs · 2025-04-09 · Tags: `smart-render`, `pre-render`, `guid-mixin`, `sdk-version`, `release-build` · [View in Q&A](../qa/smart-render/)*

---

## What caused plugins to fail to load in AE 25.2 on macOS?

The issue was related to code signature validation. Adobe built AE 25.2 with a newer Xcode that enforces stricter code signature checks. Multiple root causes were identified: (1) Unsigned plugins now fail to load on Apple Silicon. (2) Installers that don't properly clean up old files can break code signatures - if v1 had file X and v2 removes it, but the installer leaves it, the signature breaks. (3) Some plugins had 0-byte .plugin files after the AE upgrade. Solution: Ensure proper code signing, notarization, and clean installation (remove all old files before installing new version).

*Source: adobe-plugin-devs · 2025-04-08 · Tags: `code-signing`, `macos`, `ae-25-2`, `notarization`, `installer`, `apple-silicon` · [View in Q&A](../qa/code-signing/)*

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

## How do you share state across multiple effect plugins in the same AE session?

Options: (1) Use a companion AEGP plugin that communicates with all your effect plugins. (2) Use dlopen/symbol loading: define a global variable in one plugin and have other plugins dynamically load a function from it to get/set the shared state. (3) Use PlugPlug DLL for IPC communication. For state that should reset on next AE launch (not persist to prefs), the dlopen approach or AEGP companion are preferred.

*Source: adobe-plugin-devs · 2025-03-30 · Tags: `inter-plugin-communication`, `global-state`, `dlopen`, `aegp`, `shared-data` · [View in Q&A](../qa/inter-plugin-communication/)*

---

## Why might Adobe report high numbers of After Effects force-quits during launch?

A significant portion of force-quit reports during AE launch may actually be caused by plugin developers stopping AE from their debuggers (e.g., hitting 'Stop Debugging' in Visual Studio or Xcode while AE is attached). This registers as a force quit in Adobe's telemetry but is not an actual crash or bug.

*Source: adobe-plugin-devs · 2025-01-18 · Tags: `debugging`, `force-quit`, `crash-reports`, `telemetry`, `visual-studio` · [View in Q&A](../qa/debugging/)*

---

## What is UXP for Premiere Pro and how does it relate to CEP panel migration?

UXP (Unified Extensibility Platform) for Premiere Pro entered public beta in December 2024. It is the successor to CEP for building panels and extensions. Initially it is a web-based development environment, but a hybrid UXP+C++ approach is planned. For C++ computational tasks without the hybrid mode, you would need a service with IPC. The Premiere UXP beta is currently limited in functionality but expected to expand. Plugin developers with existing CEP panels should begin evaluating migration. Resources include the Adobe Creative Cloud Developer forums and the Hyperbrew blog (hyperbrew.co/blog/premiere-pro-uxp-beta). Bolt UXP (hyperbrew.co/resources/bolt-uxp/) is a tool to help with UXP development.

*Source: adobe-plugin-devs · 2024-12-06 · Tags: `premiere`, `uxp`, `cep`, `panel`, `migration`, `hybrid-cpp`, `beta` · [View in Q&A](../qa/premiere/)*

---

## How do you keep PiPL out_flags in sync with GlobalSetup flags?

Define the flags as macros in your plugin header file (e.g., #define FX_OUT_FLAGS (...)) and use those same macros in both your PiPL .r file and GlobalSetup code. The PiPL should include the header with all bit flags defined so it auto-updates when compiling. However, note that this approach may fail with higher bits/newest flags due to overflow in Adobe's PiPL compiler. An online tool for calculating flag bitmasks is available at https://reduxfx.com/aeoutflags.htm.

```cpp
#define FX_OUT_FLAGS (PF_OutFlag_FORCE_RERENDER + \
                       PF_OutFlag_USE_OUTPUT_EXTENT + \
                       PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                       PF_OutFlag_DEEP_COLOR_AWARE + \
                       PF_OutFlag_CUSTOM_UI + \
                       PF_OutFlag_I_DO_DIALOG)

// In PiPL:
AE_Effect_Global_OutFlags { FX_OUT_FLAGS },
AE_Effect_Global_OutFlags_2 { FX_OUT_FLAGS2 },
```

*Source: adobe-plugin-devs · 2024-03-27 · Tags: `pipl`, `out-flags`, `build-system`, `macros` · [View in Q&A](../qa/pipl/)*

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

## Why do keyframed parameters on adjustment layers in Premiere always return the value at the first keyframe regardless of current time?

This is caused by the PF_OutFlag_NON_PARAM_VARY flag. When this flag is set, keyframes on adjustment layers in Premiere always return the first keyframe value during checkout param, regardless of current time. This affects sliders, angles, checkboxes, and 2D points. The bug occurs in Premiere v23 and v24. Removing the NON_PARAM_VARY flag fixes the issue. In AE, this flag can be replaced using MIX_GUID during the pre_render thread. Note that apart from this bug, the flag does nothing useful in Premiere.

*Source: adobe-plugin-devs · 2024-02-01 · Tags: `premiere`, `keyframes`, `adjustment-layer`, `non-param-vary`, `checkout-param`, `bug` · [View in Q&A](../qa/premiere/)*

---

## What causes slow AE loading when debugging from Xcode 15?

After upgrading to macOS 14/Xcode 15, AE loading becomes ~3x slower due to 'dlsym cannot find symbol xSDKExport' messages for MediaCore bundles. This is Xcode trying to load symbols for all loaded bundles. Workarounds: (1) Run Xcode 14 binary directly from Terminal. (2) Open AE standalone first, then attach the debugger after AE is fully loaded. (3) Check if 'load symbols' option is accidentally selected in your IDE settings.

*Source: adobe-plugin-devs · 2023-11-29 · Tags: `xcode`, `debugging`, `performance`, `macos`, `symbol-loading` · [View in Q&A](../qa/xcode/)*

---

## How do you disable the 'previous crash warning' popup in AE?

Press CMD+F12 (or Ctrl+Shift+F12 on Windows) to open the debug console, then use the hamburger menu next to the console and change to 'Debug Database'. Search for 'ShowPreviousCrashWarning' and turn it off.

*Source: adobe-plugin-devs · 2023-11-23 · Tags: `debugging`, `crash-warning`, `ae-preferences`, `debug-database` · [View in Q&A](../qa/debugging/)*

---

## How do you restrict two float sliders relative to each other (e.g., Start cannot be >= End)?

Modifying param values in UserChangedParam works but causes problems with keyframing: a keyframe with an acceptable value can later become unacceptable, locking the value and preventing keyframe deletion. A better approach is to use the min()/max() pattern: treat the parameters as generic 'endpoints' and in your render code use min(A,B) as Start and max(A,B) as End.

*Source: adobe-plugin-devs · 2023-10-02 · Tags: `param-constraints`, `sliders`, `user-changed-param`, `keyframing` · [View in Q&A](../qa/param-constraints/)*

---

## How does PF_Arbitrary_NEW_FUNC work, and when is it called?

PF_Arbitrary_NEW_FUNC is never called in post-CS6 AE. The way arb data works now: you allocate a default arb handle during ParamSetup. For every new instance, AE sends PF_Arbitrary_COPY_FUNC with your default arb handle. Your arb data won't be zero length because it copies from the default you set up. Follow the ColorGrid SDK example which uses both NEW_FUNC and ParamSetup for this.

*Source: adobe-plugin-devs · 2023-08-22 · Tags: `arb-data`, `arbitrary-data`, `param-setup`, `new-func`, `copy-func` · [View in Q&A](../qa/arb-data/)*

---

## How do you build AE plugins for Apple Silicon (ARM64)?

You can build on an M1 Mac (even a cheap Mac Mini works as a build machine). It's not strictly required to build on M1, but it helps for testing. The key thing many people miss is that you must add ARM64 to your PiPL resource file - just building as Universal Binary in Xcode is not enough. Without the ARM64 entry in the PiPL, AE will show the 'not yet compatible' warning.

*Source: adobe-plugin-devs · 2023-08-18 · Tags: `apple-silicon`, `arm64`, `pipl`, `xcode`, `mac`, `universal-binary` · [View in Q&A](../qa/apple-silicon/)*

---

## What's a good approach for logging in AE plugins?

Use spdlog library which supports file rotation (configurable file size and number of files to keep). For performance-sensitive plugins, use conditional logging: check for the existence of a specific file (e.g., 'plugin_log.txt') during GlobalSetup, and only enable logging when that file is present. This lets users enable logging on demand for debugging.

*Source: adobe-plugin-devs · 2023-08-17 · Tags: `logging`, `spdlog`, `debugging`, `performance` · [View in Q&A](../qa/logging/)*

---

## What causes the 'Invalid Filter 25::3' error on macOS?

This error is typically caused by the Deployment Target being set too high in Xcode. For example, setting it to 12.3 will cause this error on older macOS versions. Try lowering it to support older versions. For Intel builds, supporting back to macOS 10.10 is common.

*Source: adobe-plugin-devs · 2023-08-16 · Tags: `mac`, `xcode`, `deployment-target`, `invalid-filter`, `compatibility` · [View in Q&A](../qa/mac/)*

---

## Are there CMake-based build systems for AE plugins?

Yes, there are several examples on GitHub: Vulkanator (https://github.com/Wunkolo/Vulkanator) uses CMake, and after_effects_cmake (https://github.com/mobile-bungalow/after_effects_cmake) is based on Vulkanator's setup. There's also a Rust-based project (https://github.com/virtualritz/after-effects) with its own PiPL compiler written in Rust. On Mac, you can use the Rez tool for .rsrc files and potentially use Ninja for faster incremental compilation.

*Source: adobe-plugin-devs · 2023-07-06 · Tags: `cmake`, `build-system`, `xcode`, `ninja`, `cross-platform` · [View in Q&A](../qa/cmake/)*

---

## Why does DrawBot draw colors at ~90% brightness compared to what you specify?

This can be caused by using DrawImage with opacity less than 1.0, which makes the drawing darker while alpha still appears as 1.0 (since AE's UI is behind with solid alpha). It could also be related to AE's UI brightness preferences or the project's color space settings, as AE's drawing suites may compensate for those things.

*Source: adobe-plugin-devs · 2023-06-24 · Tags: `drawbot`, `custom-ui`, `color-accuracy`, `brightness` · [View in Q&A](../qa/drawbot/)*

---

## How can I trigger a plugin refresh in Premiere when CEP updates hidden parameters, since Premiere doesn't treat script interactions as user actions like AE does?

There is no known workaround for this. Premiere does not consider script interactions as user actions, so PF_UserChangedParam won't be triggered from CEP. A practical workaround is to use external dependencies watching for file changes (though it's not very reactive) or add a button parameter to force a manual refresh.

*Source: adobe-plugin-devs · 2022-05-31 · Tags: `premiere`, `cep`, `script-interaction`, `parameter-update`, `workaround` · [View in Q&A](../qa/premiere/)*

---

## How much work is required to update a simple AE plugin for Multi-Frame Rendering (MFR)?

For simple plugins that don't use sequence data or caching, there is not much to do besides rebuilding. The complexity comes when you have GPU plugins, custom caching, or sequence data which require more significant refactoring for thread safety.

*Source: adobe-plugin-devs · 2022-02-27 · Tags: `mfr`, `multi-frame-rendering`, `plugin-update`, `thread-safety` · [View in Q&A](../qa/mfr/)*

---

## Where should After Effects SDK developers share code snippets and get help?

Stack Overflow with 'After Effects SDK' tags is recommended as a good system for searching and hosting code. The AE SDK is very niche so resources are scarce, but building up a Stack Overflow presence helps the community. The adobe-plugin-devs Discord server also serves as a community resource, though Discord's chat format has limitations for code sharing compared to a forum system.

*Source: adobe-plugin-devs · 2022-02-27 · Tags: `community`, `stack-overflow`, `resources`, `code-sharing` · [View in Q&A](../qa/community/)*

---
