# Q&A: pipl

**113 entries** tagged with `pipl`.

---

## How do you build AE plugins for Apple Silicon (ARM64)?

You can build on an M1 Mac (even a cheap Mac Mini works as a build machine). It's not strictly required to build on M1, but it helps for testing. The key thing many people miss is that you must add ARM64 to your PiPL resource file - just building as Universal Binary in Xcode is not enough. Without the ARM64 entry in the PiPL, AE will show the 'not yet compatible' warning.

*Contributors: [**gabgren**](../contributors/gabgren/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/), [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2023-08-18 · Tags: `apple-silicon`, `arm64`, `pipl`, `xcode`, `mac`, `universal-binary`*

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

*Contributors: [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2022-12-07 · Tags: `param-visibility`, `aegp`, `dynamic-stream`, `custom-ui`, `pipl`*

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

*Contributors: [**gabgren**](../contributors/gabgren/), [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: adobe-plugin-devs · 2024-03-27 · Tags: `pipl`, `out-flags`, `build-system`, `macros`*

---

## Can you have multiple effect plugins in one .aex file?

The SDK documentation says it's possible using multiple PiPLs, with AEGPs placed first. However, it's not recommended because: (1) No other hosts including Premiere Pro support multiple PiPLs. (2) If you need to update one plugin, you'd have to ship a new build of all plugins. In practice, adding another PiPL in Xcode with a different name/matchname may only show one plugin. Adobe recommends one PiPL per code fragment.

*Contributors: [**tlafo**](../contributors/tlafo/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2024-05-28 · Tags: `pipl`, `multiple-plugins`, `aex`, `architecture`*

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

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2025-06-04 · Tags: `pipl`, `out-flags`, `premiere`, `per-host`, `global-setup`*

---

## What is PF_REGISTER_EFFECT and will it replace PiPL?

AE's internal plugins now use two entry points: EffectMainExtra and PluginDataEntryFunction, defined via PF_REGISTER_EFFECT / PF_Register_effect_ext2. This appears in SDK samples like SDK_Invert_ProcAmp. While it seems like it could eventually replace PiPL, as of now PiPL is still required for AE - attempting to build without a PiPL/rsrc still results in AE not finding the effect. Premiere Pro no longer requires PiPL and uses this newer system. No official announcement about PiPL deprecation has been made.

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**tlafo**](../contributors/tlafo/), [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2025-06-05 · Tags: `pipl`, `pf-register-effect`, `plugin-entry-point`, `future`, `premiere`*

---

## How do I build an After Effects plugin for Apple Silicon (M1/ARM64)?

You need to do two things: (1) Set the Xcode build architecture to include arm64 (ensure the architecture setting targets ARM), and (2) Add 'CodeMacARM64 {"EffectMain"}' to your PiPL resource file. Without the PiPL entry, AE won't recognize the plugin as ARM-native even if the binary is compiled for arm64.

*Contributors: [**Nate**](../contributors/nate/) · Source: adobe-plugin-devs · 2022-04-28 · Tags: `apple-silicon`, `m1`, `arm64`, `pipl`, `xcode`, `mac`, `build-configuration`*

---

## How does AE_Effect_Version work in the PiPL resource file?

AE_Effect_Version encodes version information using bit shifting. The formula is: (MAJOR_VERSION << 19) + (MINOR_VERSION << 15) + (BUG_VERSION << 11) + BUILD_VERSION. Note that MAJOR_VERSION may not be larger than 8 due to bit width constraints. The full PF_VERSION macro in AE_Effect.h provides more detail with additional fields for stage and build. If you get a PiPL version mismatch error, output the version you have in code for global setup and compare it to the one in the PiPL using a hex editor.

```cpp
#define PF_VERSION(vers, subvers, bugvers, stage, build)    \
    (PFVersionInfo)(    \
         ((((A_u_long)PF_Vers_VERS_HIGH(vers)) & PF_Vers_VERS_HIGH_BITS) << PF_Vers_VERS_HIGH_SHIFT) |   \
        ((((A_u_long)(vers)) & PF_Vers_VERS_BITS) << PF_Vers_VERS_SHIFT) |    \
        ((((A_u_long)(subvers)) & PF_Vers_SUBVERS_BITS)<<PF_Vers_SUBVERS_SHIFT) |\
        ((((A_u_long)(bugvers)) & PF_Vers_BUGFIX_BITS) << PF_Vers_BUGFIX_SHIFT) |\
        ((((A_u_long)(stage)) & PF_Vers_STAGE_BITS) << PF_Vers_STAGE_SHIFT) |    \
        ((((A_u_long)(build)) & PF_Vers_BUILD_BITS) << PF_Vers_BUILD_SHIFT)    \
    )
```

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2026-02-23 · Tags: `pipl`, `version`, `ae-effect-version`, `pf-version`, `bit-shifting`*

---

## How do you set up separate debug and release plugin names in Visual Studio for AE plugins?

Use preprocessor conditionals in the PiPL .r file for both the Name and AE_Effect_Match_Name fields. Add _DEBUG to the PiPL custom build command line. You'll likely need different output filenames too. Watch out for rebuilds cleaning the previous output folder -- add a post-build copy step that copies the generated file from the local folder to the MediaCore folder.

```cpp
Name {
    #ifdef _DEBUG
    "plugin_DEBUG"
    #else
    "plugin"
    #endif
},
AE_Effect_Match_Name {
    #ifdef _DEBUG
    "ADBE plugin_DEBUG"
    #else
    "ADBE plugin"
    #endif
},
```

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/), [**Blace Plugins**](../contributors/blace-plugins/) · Source: aescripts discord · 2026-02-25 · Tags: `debug`, `release`, `pipl`, `visual-studio`, `build-configuration`, `match-name`*

---

## Is there a Rust alternative for AE/Premiere plugin development?

Yes, there is a Rust bindings project at https://github.com/virtualritz/after-effects/ with documentation at https://docs.rs/after-effects/ and https://docs.rs/premiere/. It was originally created by Moritz Moeller and significantly refactored by Adrian Eddy (author of Gyroflow). The Gyroflow AE/Premiere plugin uses these bindings. The examples show significantly less boilerplate compared to C/C++ versions, are mostly 100% safe Rust, and include a PiPL helper crate. The crates also build on Linux for development purposes (though you need Windows/macOS to run the plugin).

*Contributors: [**Lloyd Alvarez**](../contributors/lloyd-alvarez/) · Source: aescripts discord · 2024-10-02 · Tags: `rust`, `bindings`, `gyroflow`, `alternative-language`, `pipl`*

---

## What are the parts of the AE PiPL version system that are legacy and how old is some of the AE SDK code?

Some parts of the AE codebase that developers still struggle with, like PiPL data, date back to 1990 -- over 35 years old. Adobe has been reluctant to modernize these legacy systems or listen to developer suggestions. The sequence data handling hasn't changed for 15+ years either.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2026-02-04 · Tags: `pipl`, `legacy`, `sdk-history`, `adobe`*

---

## How do you prevent an After Effects plugin from appearing in Premiere Pro?

Set the PF_OutFlag_I_AM_OBSOLETE flag in out_data->out_flags when the application ID is 'PrMr' (Premiere Pro). This is done by checking if(in_data->appl_id == 'PrMr') and then setting out_data->out_flags = PF_OutFlag_I_AM_OBSOLETE;

```cpp
if(in_data->appl_id == 'PrMr')
{
    out_data->out_flags = PF_OutFlag_I_AM_OBSOLETE;
}
```

*Tags: `premiere`, `pipl`, `params`*

---

## What is required to support M1 Macs when including external libraries like OpenCV in an After Effects plugin?

When building for M1 with external libraries, ensure the library is compiled with appropriate Mac support options. If using dynamic libraries (dylib), you need to embed them in the plugin file and specify rpath in Xcode. Static libraries (.a files) should be linked directly. The Missing Entry Point error typically occurs when the plugin is applied, not during loading, and may indicate a PiPL configuration issue rather than a linking problem. It's recommended to test on both M1 and x86 Mac architectures to isolate platform-specific issues, as the problem may be related to project configuration rather than the external library itself.

*Tags: `apple-silicon`, `macos`, `build`, `pipl`, `deployment`*

---

## At what point does the Missing Entry Point error occur when loading a plugin?

The Missing Entry Point error occurs when the effect is applied, not during After Effects loading. This timing distinction helps differentiate between entry point/PiPL issues versus runtime initialization problems.

*Tags: `pipl`, `debugging`*

---

## How can I create a parameter slider with a percentage display and negative range (-100% to 100%) instead of the default 0% to 100%?

Use PF_ADD_FLOAT_SLIDERX instead of PF_ADD_FIXED, specifying the full range including negative values. The slider_min, slider_max, value_min, and value_max parameters should all support negative values. The PF_ValueDisplayFlag_PERCENT flag will display the values as percentages regardless of the range.

```cpp
PF_ADD_FLOAT_SLIDERX("Length", -100, 100, -100, 100, 100, PF_Precision_TENTHS, PF_ValueDisplayFlag_PERCENT, NULL, PARAM_LENGTH_ID);
```

*Tags: `params`, `ui`, `pipl`*

---

## How can you debug plugin crashes when changing the macOS deployment target doesn't resolve the issue?

Add logging at function entry and exit points to determine where the crash occurs (global setup, param setup, etc). You can use conditional logging by checking for a log file's presence in global setup, so users only enable logging when needed. For production plugins, consider profiling performance impact of logging and use a library like spdlog with file rotation to manage log file sizes.

*Tags: `macos`, `debugging`, `pipl`*

---

## How should logging be implemented in plugins without slowing down render performance?

For real-time or fast-rendering plugins, logging can impact performance. Implement conditional logging by checking for a marker file (e.g., 'pixel_sorter_log.txt') in global setup, enabling logging only when the file exists. This allows users to request logging without it affecting normal operation. Use spdlog library which includes file rotation capabilities to manage log file sizes and count.

*Tags: `debugging`, `performance`, `pipl`*

---

## Why might a plugin fail to load even though it builds successfully?

The plugin's PiPL resource needs to include ARM64 architecture support. If ARM64 is not added to the PiPL, the plugin won't load properly even if the build succeeds.

*Tags: `pipl`, `apple-silicon`, `build`*

---

## How can you enable or toggle a slider parameter to be collapsed in After Effects?

Slider parameters cannot be collapsed individually. The PF_ParamFlag_COLLAPSE_TWIRLY flag only works with parameter groups (topics), not individual slider parameters. To collapse parameters, you need to use PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG on a group/topic, not on individual sliders.

```cpp
def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;
```

*Tags: `params`, `ui`, `pipl`*

---

## Why does changing the match name cause the CUI and parameters to become invisible in the ECW?

This was caused by an underlying C++ syntax error where a PF_Err variable was declared but not initialized, causing AEGP_GetNewEffectStreamByIndex to return 0x0. The real issue was that error checking with if(!err) statements was failing because the uninitialized error variable had a garbage value (usually 1) instead of PF_Err_NONE (0). The solution is to properly initialize all error variables when declaring multiple variables of the same type.

```cpp
// Wrong - err is undefined
PF_Err err, err2 = PF_Err_NONE;

// Correct - both variables initialized
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `pipl`, `aegp`, `debugging`, `params`*

---

## How can you ensure that PiPL flags are automatically updated when compiling without manual definition?

Define the flags in a header file using #define (e.g., FX_OUT_FLAGS) and reference that macro in the PiPL file. This way the PiPL auto-updates when compiling. However, this technique has limitations with higher bits/newest flags as it can cause overflow in Adobe's pipl compiler, in which case you must revert to manually defining flags as bits.

```cpp
#define FX_OUT_FLAGS (  PF_OutFlag_FORCE_RERENDER + \
                         PF_OutFlag_USE_OUTPUT_EXTENT + \
                         PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                         PF_OutFlag_DEEP_COLOR_AWARE + \
                         PF_OutFlag_CUSTOM_UI + \
                         PF_OutFlag_I_DO_DIALOG )

AE_Effect_Global_OutFlags {
    FX_OUT_FLAGS
}
```

*Tags: `pipl`, `build`, `params`*

---

## In what scenarios would the nullptr checks for extraP and its members fail?

The checks for extraP, extraP->cb, and extraP->cb->GuidMixInPtr can fail in older versions of After Effects (before PF_AE130_PLUG_IN_VERSION) where PreRenderExtra or the callback structure are not available, or when the GuidMixInPtr function pointer is not implemented or initialized. The code also checks for a sentinel value (0xabababababababab) to detect uninitialized function pointers. These safeguards ensure compatibility across different AE versions and configurations where these features may not be present.

```cpp
if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION && extraP && extraP->cb && extraP->cb->GuidMixInPtr && (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab)
```

*Tags: `pipl`, `params`, `cross-platform`, `debugging`*

---

## Is it possible to include multiple PiPLs in the same file?

Yes, it is possible to include multiple plug-ins (both AEGPs and effects) in the same file using multiple PiPLs, but it is not recommended. If there are PiPLs for both AEGPs and effects in the same file, the AEGPs must come first. No other hosts, not even Premiere Pro, support multiple PiPLs pointing to multiple effects within the same .dll or code fragment. The recommendation is to use one PiPL and one plug-in per code fragment.

*Tags: `pipl`, `aegp`, `premiere`, `build`*

---

## When is PF_Cmd_GLOBAL_SETUP called in After Effects and Premiere?

In After Effects, GLOBAL_SETUP is called the first time the user applies the effect plugin to a layer. In Premiere, it is called when the application is loading.

*Tags: `aegp`, `pipl`, `premiere`, `ui`*

---

## Is it possible to show a dialog during Global Setup?

It should be possible to show any dialog during Global Setup, though there is some uncertainty about whether dialogs can be displayed during the initial AE loading screen when plugins are being scanned.

*Tags: `ui`, `aegp`, `pipl`*

---

## Can I directly modify the output PF_LayerDef pointer or do I need to use PF_NewWorld?

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

*Tags: `mfr`, `memory`, `output-rect`, `pipl`*

---

## Has anyone successfully compiled an After Effects plugin using cmake/ninja?

Yes, there are working examples available. The mobile-bungalow/after_effects_cmake repository provides a cmake setup based on vulkanator's work. Additionally, the virtualritz/after-effects project offers a cross-platform build system with its own PiPL compiler written in Rust as an alternative to using pipltool.exe.

*Tags: `build`, `cmake`, `pipl`, `cross-platform`*

---

## How do you compile .rc files on macOS for After Effects plugins?

On macOS, you can use the native macOS tool called Rez to compile .rc files. The virtualritz/after-effects project demonstrates this approach as part of its build system. The Rust PiPL compiler from that same project can also handle resource compilation as part of a cross-platform build setup.

*Tags: `build`, `macos`, `pipl`, `deployment`*

---

## What is the official way to determine pixel format in Premiere using the AE SDK?

The official way to determine pixel format in Premiere using the AE SDK is to call the PF_PixelFormatSuite1, as seen in the SDK_Noise sample in the AE SDK since AE CS5. Using PF_WorldFlag_RESERVED1 was an unofficial/internal workaround by Adobe developers that is no longer recommended.

*Tags: `premiere`, `pipl`, `params`*

---

## How do you debug a plugin that fails to load in After Effects and appears offline in Premiere?

Check if the error occurs before global_setup is called. If it happens before that point, it may be due to a library dependency that AE loaded in a new version that your plugin uses. Verify whether the issue occurs on both Windows and macOS or only on specific platforms.

*Tags: `debugging`, `deployment`, `macos`, `windows`, `pipl`*

---

## Can global_outflags be set differently for specific host applications in PIPL?

Yes, you can set flags per host in global setup by checking the host application ID. Use a conditional check on in_data->appl_id to determine which host is running, then set different out_flags accordingly. For example, you can check if the application is not Premiere Pro (appl_id != 'PrMr') and apply different flags for After Effects versus other hosts.

```cpp
if (in_data->appl_id != 'PrMr') {
  out_data->out_flags = PR_OUT_FLAGS;
}
```

*Tags: `pipl`, `premiere`, `cross-platform`, `params`*

---

## Is it possible in PiPL to set global_outflags specific to a host app, such as removing a flag for Premiere while keeping it in After Effects?

Yes, it is possible to set global_outflags specific to host apps in PiPL files. You can use conditional logic or host-specific PiPL configurations to remove flags like no_params_vary for Premiere while maintaining them in After Effects.

*Tags: `pipl`, `premiere`, `params`*

---

## Is PiPL deprecated and are there alternative ways to describe a plugin entrypoint?

PiPL is considered somewhat deprecated, and there are alternative approaches to describing plugin entrypoints, though the conversation does not specify the exact alternatives mentioned.

*Tags: `pipl`, `deployment`*

---

## What is the first version of After Effects that supports the new plugin format without PIPL?

Based on the conversation, support started with After Effects 2018, though the documentation still references it as a future release at the time of discussion.

*Tags: `pipl`, `deployment`, `build`*

---

## What are the new entry points in After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. Currently, PIPL is still used by After Effects but not by Premiere Pro anymore, so you only need to write PIPL for AE. The new entry points potentially allow for removing PIPL/rsrc resources entirely, though the exact mechanism needs further investigation in the samples.

*Tags: `pipl`, `aegp`, `ae`, `plugin-architecture`*

---

## How do you specify which function is EffectMain when using the new PF_REGISTER_EFFECT entry points?

The exact mechanism for specifying the EffectMain function with the new entry points is not clearly documented. It should be checked in the official After Effects plugin samples, though the investigation into this approach is ongoing.

*Tags: `pipl`, `aegp`, `plugin-architecture`*

---

## Is a PiPL resource still needed to register After Effects plugins, or can plugins be registered with just the URL info function?

PiPL is still needed even with the PF_Register_effect_ext2 function added in version 23. While you can overwrite some PiPL values in code, the resource file cannot be completely eliminated. After Effects still cannot find effects without a PiPL/rsrc file, though Adobe may update this in the future.

*Tags: `pipl`, `build`, `deployment`, `plugin-registration`*

---

## Can a single After Effects plugin file contain multiple PiPL entries with different entry points?

Yes, a plugin can have multiple PiPL entries allowing different entry points and different plugins in one file. However, this functionality is not consistently supported throughout After Effects and is not supported at all in Premiere Pro, so it has not gained much traction in practice.

*Tags: `pipl`, `premiere`, `plugin-architecture`*

---

## How do you create a render-only version of a plugin that doesn't require additional licenses for render farms?

One approach is to make all parameters invisible while still allowing the plugin to read them, creating a read-only version of the plugin that can process renders without displaying the GUI interface.

*Tags: `deployment`, `ui`, `render-loop`, `pipl`*

---

## How can you communicate between an AEGP plugin and an FX plugin to push strings into a queue?

You will need a C interface to push the string into the queue between the two binaries. The AEGP will be one plugin and the FX plugin will be another separate binary.

*Tags: `aegp`, `pipl`, `cross-platform`*

---

## How can you prevent an After Effects plugin from being detected and displayed in Premiere's effects list?

Return an error in global setup, since Premiere runs global setup for all plugins during application startup. If the plugin returns an error during this initialization phase, it should be hidden from the effects list.

*Tags: `premiere`, `pipl`, `debugging`*

---

## Does PF_OutFlag_I_AM_OBSOLETE flag work to hide a plugin in Premiere Pro while keeping it visible in After Effects?

The flag will hide the plugin in both AE and Premiere Pro, so it won't achieve the desired result of hiding only in PPro unless you conditionally set it after checking the host name. You can make it work by checking the host application first and only applying the flag when running in Premiere Pro.

*Tags: `pipl`, `premiere`, `cross-platform`, `deployment`*

---

## How do you build an After Effects plugin for Apple Silicon (M1) native support?

To build for M1 native (non-Rosetta), you need to: 1) Set the appropriate build configuration for Apple Silicon architecture, and 2) Add CodeMacARM64 {"EffectMain"} entry in the PiPL resource file. Without the PiPL entry, the plugin won't be recognized as native ARM64 compatible even if the binary is built correctly.

```cpp
CodeMacARM64 {"EffectMain"}
```

*Tags: `macos`, `apple-silicon`, `build`, `pipl`*

---

## How do you properly include external libraries like OpenCV in an After Effects plugin targeting Apple Silicon (M1)?

When including external libraries in AE plugins for M1, ensure you're linking against static libraries (.a files) rather than dynamic libraries. If using dynamic libraries, you may need to specify rpath in Xcode and embed the dylib in the plugin file. Verify that your PiPL file contains the correct entry point definition. If experiencing 'Missing Entry Point' errors when applying the effect, test on x86 Mac to isolate whether the issue is architecture-specific or a general PiPL configuration problem. Common causes include outdated project settings when migrating older projects to M1 support.

*Tags: `macos`, `apple-silicon`, `build`, `pipl`, `cross-platform`*

---

## How do you ensure ARM64 architecture support is properly configured in an After Effects plugin build?

You need to add ARM64 to your PiPL (Property List) file. Without this entry, the plugin may not build or load correctly for ARM64 architecture even if the C++ code and Xcode settings are correct.

*Tags: `pipl`, `macos`, `apple-silicon`, `build`*

---

## How can you collapse a slider parameter in After Effects plugins?

To collapse a slider parameter, you need to set the global flag PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG and then set the param flag with 'def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;'. However, this approach may not work for individual sliders in the same way it works for topic groups.

```cpp
def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## What is the correct C++ syntax for initializing multiple PF_Err variables in After Effects plugins?

When declaring multiple PF_Err variables, you must initialize each one explicitly. Incorrect: `PF_Err err, err2 = PF_Err_NONE;` (only err2 is initialized, err is undefined). Correct: `PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;` (both variables initialized). This matters because uninitialized error codes will not evaluate to 0 in conditional statements like `if(!err)`, causing logic failures.

```cpp
// Incorrect - err is undefined
PF_Err err, err2 = PF_Err_NONE;

// Correct - both initialized
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `debugging`, `aegp`, `pipl`*

---

## How can you create a custom banner in the After Effects effects menu like FxFactory plugins do?

This appears to be a FxFactory-specific feature. The discussion suggests several investigation approaches: comparing AE installation folders before and after FxFactory installation to identify changes, examining the .aex plugin file and its PiPL resources for unknown SDK functions, or contacting FxFactory developers directly. It's unclear whether the banner is drawn through standard AEGP suite functions or through custom modifications to the plugin manager.

*Tags: `pipl`, `ui`, `reverse-engineering`, `aegp`, `debugging`*

---

## Is there a way for an effect to programmatically reset itself using a command ID instead of manually setting all parameters to defaults via streams?

James Whiffin asked about programmatically resetting an effect using a command ID rather than manually resetting each parameter through streams. This question was not answered in the conversation.

*Tags: `params`, `aegp`, `pipl`*

---

## Is there a way for an effect to programmatically reset itself instead of manually setting all parameters to their defaults?

James Whiffin asked whether effects can use a command ID or similar mechanism to reset themselves programmatically rather than manually setting each parameter to default values via streams. This question was asked but no answer was provided in the conversation.

*Tags: `params`, `aegp`, `pipl`*

---

## What is an easy way to calculate and verify the correct bitmask values for PiPL outflags and outflags2 fields?

Tobias Fleischer created a quick online lookup/cheat sheet tool in JavaScript that allows developers to easily find and verify the correct bitmask values for PiPL outflags and outflags2 fields without needing to run the host application and receive warnings. The tool is available at: https://www.reduxfx.com/piplflags/

*Tags: `pipl`, `tool`, `reference`, `debugging`, `build`*

---

## What is a best practice for managing PiPL outflags in plugin source code?

Define outflags as preprocessor macros in your plugin header file, then reference these macros in your PiPL resource. This way the flags are centrally defined and automatically updated during compilation, eliminating the need to manually touch the PiPL values. Example: define FX_OUT_FLAGS with all necessary flag constants (like PF_OutFlag_FORCE_RERENDER, PF_OutFlag_DEEP_COLOR_AWARE, etc.) and use them in AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 sections.

```cpp
#define FX_OUT_FLAGS (  PF_OutFlag_FORCE_RERENDER + \
                        PF_OutFlag_USE_OUTPUT_EXTENT + \
                        PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                        PF_OutFlag_DEEP_COLOR_AWARE + \
                        PF_OutFlag_CUSTOM_UI + \
                        PF_OutFlag_I_DO_DIALOG )

AE_Effect_Global_OutFlags {
    FX_OUT_FLAGS
},
AE_Effect_Global_OutFlags_2 {
    FX_OUT_FLAGS2
}
```

*Tags: `pipl`, `build`, `params`, `best-practice`*

---

## How should output flags be defined in an After Effects plugin to ensure they auto-update during compilation?

Define output flags using a macro in your plugin header file, then reference that macro in both the PiPL AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 entries. This ensures the PiPL automatically updates when you recompile. However, be aware that using higher bits/newest flags with this approach can cause overflow in Adobe's PiPL compiler, so for newer flags you may need to revert to manually defining flags as individual bits.

```cpp
#define FX_OUT_FLAGS (  PF_OutFlag_FORCE_RERENDER + \
                         PF_OutFlag_USE_OUTPUT_EXTENT + \
                         PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                         PF_OutFlag_DEEP_COLOR_AWARE + \
                         PF_OutFlag_CUSTOM_UI + \
                         PF_OutFlag_I_DO_DIALOG )

// In PiPL:
AE_Effect_Global_OutFlags {
    FX_OUT_FLAGS
},
AE_Effect_Global_OutFlags_2 {
    FX_OUT_FLAGS2
}
```

*Tags: `pipl`, `params`, `build`, `reference`*

---

## What is a tool for calculating PiPL outflags and outflags2 bitmasks?

Tobias Fleischer (reduxFX) created an online lookup/cheat sheet for calculating and evaluating bitmasks for the outflags and outflags2 fields in PiPL files. Available at https://reduxfx.com/aeoutflags.htm

*Tags: `pipl`, `tool`, `reference`, `deployment`*

---

## Is it possible to include multiple PiPLs in a single plugin file, and what are the recommendations?

Yes, it is technically possible to include multiple PiPLs (both AEGPs and effects) in the same file, but it is not recommended. If you do use multiple PiPLs in the same file, AEGPs must come first. However, no other hosts (not even Premiere Pro) support multiple PiPLs pointing to multiple effects within the same .dll or code fragment. The recommendation from the SDK is to use one PiPL and one plugin per code fragment, especially to avoid shipping new builds of all plugins when updating just one.

*Tags: `pipl`, `aegp`, `deployment`, `premiere`*

---

## Why does adding another PiPL resource with a different name and matchname only show one plugin in After Effects?

When multiple PiPL resources are added to the same Xcode project, After Effects may only recognize one of them. This is typically due to incorrect PiPL configuration, resource ID conflicts, or the build process not properly including all PiPL resources in the final plugin binary. Ensure each PiPL has a unique resource ID, verify the matchname and name fields are correctly differentiated, and check that all PiPL resources are included in the target's Copy Bundle Resources build phase.

*Tags: `pipl`, `build`, `xcode`, `plugin-structure`*

---

## When is PF_Cmd_GLOBAL_SETUP called in After Effects versus Premiere Pro?

In After Effects, PF_Cmd_GLOBAL_SETUP is called the first time the user applies the effect plugin to a layer. In Premiere Pro, it is called when the application is loading. The timing differs between the two host applications.

*Tags: `aegp`, `pipl`, `premiere`, `reference`*

---

## Can you display dialogs during the Global Setup phase of a plugin?

According to discussions in the community, it should be possible to show dialogs during Global Setup. However, there is some uncertainty about whether Global Setup is called during the AE loading screen when scanning plugins, or only when the effect is first applied. The capability exists, but the exact execution context may affect dialog behavior.

*Tags: `ui`, `pipl`, `aegp`, `debugging`*

---

## What is the official Adobe documentation for pixel sampling and layer manipulation in After Effects plugins?

Adobe provides comprehensive documentation at https://ae-plugins.docsforadobe.dev/effect-details/tips-tricks.html covering sampling pixels at X,Y coordinates and related plugin development techniques.

*Tags: `reference`, `aegp`, `pipl`, `debugging`*

---

## Is there a CMake-based build system available for After Effects plugins?

Yes, there is a CMake/Ninja setup available at https://github.com/mobile-bungalow/after_effects_cmake which is based on vulkanator's cmake setup. This uses pipltool.exe for PiPL compilation.

*Tags: `cmake`, `build`, `pipl`, `open-source`, `reference`*

---

## What is an alternative cross-platform build system for After Effects plugins with a custom PiPL compiler?

The virtualritz/after-effects project at https://github.com/virtualritz/after-effects provides its own cross-platform build system with a custom PiPL compiler written in Rust, offering an alternative to traditional build approaches.

*Tags: `build`, `pipl`, `cross-platform`, `open-source`, `reference`, `rust`*

---

## How can you compile .rc resource files on macOS for After Effects plugins?

On macOS, use the native macOS tool Rez to compile .rc resource files. This approach can be integrated into build systems like CMake to handle resource compilation as part of the plugin build process.

*Tags: `build`, `macos`, `pipl`, `resource-compilation`*

---

## Can you set global_outflags in a pipl file to be host-application specific?

The user asked whether pipl files support setting global_outflags with host-app-specific conditions. They wanted to disable the no_params_vary flag for Premiere due to a bug while keeping it enabled in After Effects. No definitive answer was provided in the conversation, so this question remains unresolved.

*Tags: `pipl`, `premiere`, `aegp`, `params`*

---

## How can you set different global_outflags for different host applications in a PIPL plugin?

You can set flags per host in global setup by checking the appl_id in in_data. For example, you can conditionally set out_flags based on whether the host is Premiere or After Effects: if (in_data->appl_id != 'PrMr') { out_data->out_flags = PR_OUT_FLAGS; }. This allows you to remove flags like no_params_vary for Premiere while using them in After Effects.

```cpp
if (in_data->appl_id != 'PrMr') {
  out_data->out_flags = PR_OUT_FLAGS;
}
```

*Tags: `pipl`, `premiere`, `aegp`, `cross-platform`*

---

## Is it possible in PiPL to set global_outflags specific to a host application like Premiere?

Yes, you can set global_outflags with host-specific conditions in PiPL files. One approach is to use GUID-mix with time values to conditionally apply flags like no_params_vary for different host applications, allowing you to remove a flag for Premiere while keeping it enabled in After Effects.

*Tags: `pipl`, `premiere`, `params`, `aegp`*

---

## Are there alternatives to PiPL for describing plugin entrypoints in After Effects plugins?

Yes, PiPL is considered somewhat deprecated. There are alternative ways to describe plugin entrypoints, though the conversation does not specify the exact modern replacement method. Developers should investigate newer plugin descriptor formats beyond the traditional PiPL approach.

*Tags: `pipl`, `aegp`, `deprecated`, `reference`*

---

## What are the new entry points for After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. PIPL is no longer used by Premiere Pro, so developers only need to write PIPL for After Effects. Some investigation suggests it may be possible to remove PIPL entirely by using these new entry points, though the exact mechanism is not yet fully documented.

*Tags: `pipl`, `aegp`, `EffectMain`, `entry-point`, `reference`*

---

## Can multiple effects be registered within a single DLL for After Effects plugins?

Yes, according to Alex Bizeau from maxon, it is possible to call multiple register effect functions in one DLL. This approach could be used to create a bootstrapper similar to what was done for OFX, allowing a single DLL to load multiple effects and reduce code duplication across OFX, After Effects, and AVX plugins.

*Tags: `pipl`, `aegp`, `plugin-architecture`, `deployment`, `build`*

---

## When was the URL-based plugin registration method added to After Effects?

The URL-based plugin registration version was added two versions ago in After Effects 23, according to tlafo. This allows plugins to be registered without needing traditional PIPL resources.

*Tags: `pipl`, `aegp`, `deployment`, `reference`*

---

## Is PiPL still required for After Effects plugins, or can plugins be registered without it?

PiPL is still required for After Effects plugins. While the function PF_Register_effect_ext2 was added two versions ago (in AE 23) to allow registration with URL info, this does not eliminate the need for PiPL/rsrc files. Plugins without PiPL cannot be found by After Effects. The new entry point functionality has advantages, but PiPL is still mandatory, though some of its values can be overwritten in code. However, OFX (which is partly based on the AE SDK) successfully eliminated PiPL entirely, making it more flexible for multi-plugin deployment.

*Tags: `pipl`, `aegp`, `registration`, `plugin-architecture`*

---

## What flag should be set in global setup to handle premultiplied alpha correctly?

Set the flag PF_outflags2_REVEALS_ZERO_ALPHA in global setup to properly handle cases where the plugin reveals zero alpha, which is related to the premultiplied alpha of the input layer.

*Tags: `pipl`, `params`, `alpha`, `mfr`*

---

## How do you create collapsible parameter groups in After Effects plugins?

To create collapsible parameter groups, set the global PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG flag. Then for each individual parameter, set PF_ParamFlag_START_COLLAPSED if that parameter should be collapsed by default, or omit the flag if it should not be collapsed.

*Tags: `params`, `ui`, `pipl`*

---

## What is the most logical way to prevent an After Effects plugin from being detected in Premiere Pro?

One approach is to return an error in the global setup function, since Premiere Pro runs global setup for all plugins during application startup. This may cause the plugin to be hidden from the effects list, though this behavior was noted as not always working reliably.

*Tags: `premiere`, `deployment`, `pipl`, `aegp`*

---

## How can plugin developers avoid requiring customers to reinstall plugins when After Effects versions are updated?

Use the PF_OutFlag_I_AM_OBSOLETE flag when setting plugin output flags. This flag, when added to the plugin configuration, signals to After Effects that the plugin can work across version updates without requiring reinstallation. Plugin developers should suggest customers install plugins into the mediacore directory to leverage this compatibility feature and avoid the yearly reinstall cycle that occurs with AE version upgrades.

*Tags: `pipl`, `deployment`, `cross-platform`*

---

## How can you prevent the 'need to reinstall plugin' message when After Effects updates without hiding the plugin from the UI?

James Whiffin suggests installing plugins into mediacore to avoid yearly reinstall requirements when AE versions up. However, Tobias Fleischer notes that using PF_OutFlag_I_AM_OBSOLETE will hide the plugin in AE itself. To use this flag only for Premiere Pro while keeping it visible in After Effects, you should add a host name check before applying the flag.

*Tags: `pipl`, `deployment`, `premiere`, `cross-platform`*

---

## Why are plugins not loading in After Effects 2026?

There is a known issue on Mac where plugins located in the Mediacore folder are not recognized by AE2026 and must be placed in After Effects' main plugin folder instead. This appears to be one of several compatibility issues affecting plugin loading in AE2026.

*Tags: `macos`, `deployment`, `pipl`, `debugging`*

---

## How do you build an After Effects plugin for Apple Silicon M1 Macs?

To build for M1, you need to: (1) Set the appropriate build configuration for Mac ARM64 architecture, and (2) Add CodeMacARM64 {"EffectMain"} to your PiPL (Plugin Property List) file to ensure the plugin is properly registered for native ARM64 execution rather than Rosetta emulation.

```cpp
CodeMacARM64 {"EffectMain"}
```

*Tags: `apple-silicon`, `macos`, `build`, `pipl`*

---

## How should popup menu items be formatted in After Effects plugin parameter definitions?

When using PF_ADD_POPUPX or add_popup to define popup menu parameters, the separator (pipe character |) must be placed at the end of each line for all choices except the last one. For example: "choice1|" "choice2|" "Choice3". Ensure the parameter definition is properly structured with AEFX_CLR_STRUCT before the popup definition and verify there are no missing parameter definitions between the popup and any subsequent function calls.

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_POPUPX("Color", 3, 2,
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## How do you properly update UI parameter values using PF_Cmd_UPDATE_PARAMS_UI in After Effects plugins?

During UPDATE_PARAMS_UI, you should only change parameter appearance (hidden, disabled, etc.) and not values. To change values, set proper defaults during PARAM_SETUP instead. When updating parameters, do not modify values and flags directly on the original params array. Instead, make a copy of the param struct, modify the copy, and pass it back to PF_UpdateParamUI. Also ensure you set out_data->out_flags |= PF_OutFlag_REFRESH_UI before returning. Refer to the Supervisor SDK sample project for correct implementation details.

```cpp
static PF_Err UpdateUI(PF_InData* in_data, PF_OutData* out_data, PF_ParamDef* params[]) {
  PF_Err err = PF_Err_NONE;
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  // Make a copy, modify the copy, then use PF_UpdateParamUI
  ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref, SKELETON_COLOR, params[SKELETON_COLOR]));
  out_data->out_flags |= PF_OutFlag_REFRESH_UI;
  return err;
}
```

*Tags: `params`, `ui`, `pipl`, `reference`*

---

## Is it possible to add or remove effect parameters after the PF_Cmd_PARAM_SETUP event?

No, you cannot add or remove parameters at any time other than the initial setup. The number and count of parameters are fixed. However, you can hide and show parameters dynamically based on user input. An alternative workaround used by plugins like Plexus is to add additional effects to the layer that contain sets of controls, and have the main effect read their parameters during rendering.

*Tags: `params`, `pipl`, `effect_api`*

---

## Can you change the name of an effect parameter after initial setup?

Yes, you can change a parameter's name by getting its streamRef and changing the stream's name. However, this change will not persist between sessions—when you load a project, the default names will be used again. You can work around this by renaming parameters again during the sequence_resetup event. Note that this approach has been reported to cause issues in Premiere Pro.

*Tags: `params`, `pipl`, `premiere`*

---

## How do I make a 3D plugin display the 3D icon and trigger rendering automatically when the camera moves in After Effects?

To enable 3D camera support in your After Effects plugin, you need to set the flags PF_OutFlag2_I_USE_3D_CAMERA and PF_OutFlag2_I_USE_3D_LIGHTS, and use AEGP_GetEffectCameraMatrix to access the camera matrix. However, you must also change the corresponding flags in the PiPL file. Additionally, on Windows, you need to clean and rebuild your project, as the PiPL only regenerates after a clean build.

*Tags: `3d`, `aegp`, `pipl`, `build`, `camera`*

---

## How do you get the original foreground layer pixels when implementing a custom blend mode effect on an adjustment layer?

To get the foreground layer's original pixels, use the layer parameter checkout mechanism. First, set up a layer parameter with "self" as the default. Then use PF_CHECKOUT_PARAM() to checkout the layer at the desired time, and access the fetched pixels at checkout.u.ld. Alternatively, you can use AEGP_GetLayerSourceItem() to get the layer's source item, AEGP_NewFromItem() to create render options, AEGP_RenderAndCheckoutFrame() to get a receipt, and AEGP_GetReceiptWorld() to fetch the pixels. The layer parameter checkout method fetches pre-masks and pre-effects pixels, which is what you need for blend modes.

```cpp
PF_ParamDef checkout;
ERR(PF_CHECKOUT_PARAM( in_data,
index_of_layer_param,
in_data->current_time,
in_data->time_step,
in_data->time_scale,
&checkout));
// Fetched pixels are at checkout.u.ld
```

*Tags: `layer-checkout`, `aegp`, `blend-mode`, `pipl`, `params`*

---

## What is the difference between the old checkout mechanism and checkout_layer_pixels() when fetching layer pixels?

The old checkout mechanism (PF_CHECKOUT_PARAM) fetches the original layer pixels pre-masks and pre-effects, making it suitable for implementing custom blend modes. The newer checkout_layer_pixels() function fetches pixels post-masks and post-effects. For blend mode implementations where you need the original foreground pixels, use the old PF_CHECKOUT_PARAM mechanism instead.

*Tags: `layer-checkout`, `pipl`, `params`, `blend-mode`*

---

## How can I define AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags2 as constants instead of magic numbers in PiPL.r?

Use an enum in a C++ header file with bitwise OR operations to combine the individual flag constants (e.g., PF_OutFlag_PIX_INDEPENDENT | PF_OutFlag_SEND_UPDATE_PARAMS_UI | etc.), then include that header in your resource file. On Visual Studio, you can hover over the enum value to see the calculated result, then copy it to a #define for cross-platform compatibility. On Xcode, use #define directly with the calculated hex values. Note that #define macros cannot be used directly in PiPL.r syntax—you must use the final numeric values.

```cpp
enum {
  MAJOR_VERSION = 1,
  MINOR_VERSION = 6,
  BUG_VERSION = 0,
  STAGE_VERSION = 2,
  BUILD_VERSION = 0,
  CALCULATED_RESOURCE_VERSION = MAJOR_VERSION * 524288 + MINOR_VERSION * 32768 + BUG_VERSION * 2048 + STAGE_VERSION * 512 + BUILD_VERSION
};

#define OUTFLAGS (PF_OutFlag_PIX_INDEPENDENT | PF_OutFlag_SEND_UPDATE_PARAMS_UI | PF_OutFlag_USE_OUTPUT_EXTENT | PF_OutFlag_DEEP_COLOR_AWARE)
#define OUTFLAGS2 (PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG | PF_OutFlag2_FLOAT_COLOR_AWARE | PF_OutFlag2_SUPPORTS_SMART_RENDER | PF_OutFlag2_DOESNT_NEED_EMPTY_PIXELS)
```

*Tags: `pipl`, `build`, `cross-platform`, `windows`, `macos`*

---

## How do I draw custom UI in the title area of a parameter to position it as far left as possible?

When defining the parameter, set PF_PUI_TOPIC instead of PF_PUI_CONTROL. You'll need to respond to PF_EA_PARAM_TITLE instead of PF_EA_CONTROL during the event call. If you need UI in both the title area and control area, you can define both for the same parameter, but note that the two areas won't join into one continuous segment.

*Tags: `ui`, `pipl`, `params`*

---

## Where did the 'About' button go for plugins in After Effects 2020?

The 'About' button was deliberately hidden in After Effects 2020. It is still accessible by right-clicking the effect name and choosing 'About' from the context menu. As an alternative, developers can use PF_SetOptionsButtonName() to implement an options button, which will be displayed next to the 'Reset' button.

*Tags: `ui`, `macos`, `pipl`, `debugging`*

---

## What API function should be used to add a custom options button to an After Effects plugin?

The PF_SetOptionsButtonName() function should be used to implement an options button in After Effects plugins. This button will be displayed next to the 'Reset' button and provides a visible alternative to the hidden 'About' button in After Effects 2020 and later.

```cpp
PF_SetOptionsButtonName()
```

*Tags: `ui`, `pipl`, `params`*

---

## How should PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS be used correctly when adding new parameters to legacy plugin versions?

The flag PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS is meant to provide a default value for old projects that don't have the new parameter. However, the PF_ADD_CHECKBOX macro copies the passed default over the passed value, which can cause unexpected behavior. The solution is to either copy the macro code into your own files and fix it there, or manually define the parameter without using the macro, defining it the old-fashioned way. Avoid modifying the AE headers directly as SDK updates would undo the changes.

```cpp
def.u.bd.value = FALSE;        // value for legacy projects which did not have this param
PF_ADD_CHECKBOX(
    STR(StrID_Checkbox_Param_Name),
    STR(StrID_Checkbox_Description),
    TRUE, // value for new applications, and when reset
    PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS,
    DOWNSAMPLE_DISK_ID);
```

*Tags: `params`, `pipl`, `sdk`, `reference`*

---

## How can I get the After Effects version string (e.g., 'After Effects CC 2017') in my plugin?

There is no direct SDK function to retrieve the version string. Instead, use PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH_W, wPath) to get the path to your plugin files, then decipher the AE version from the file path. Alternatively, look for the 'Presets' folder which is always in the same directory as AfterFX.exe. You can also check the Windows registry at HKEY_LOCAL_MACHINE\SOFTWARE\Adobe\After Effects\ or the macOS library at /Library/Preferences/com.Adobe.After Effects.paths.plist.

```cpp
A_UTF16Char wPath[AEFX_MAX_PATH];
PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH_W, wPath);
```

*Tags: `aegp`, `pipl`, `windows`, `macos`, `sdk`*

---

## How do you add static text labels to an After Effects effect UI?

You can use PF_ADD_TOPIC and PF_END_TOPIC macros without putting any params in between to create a text section. Alternatively, you can use PF_ADD_NULL, declare a minimal custom UI size, and not implement it.

*Tags: `ui`, `pipl`, `params`, `reference`*

---

## What does the PF_ prefix stand for in the After Effects SDK?

PF stands for 'Plug-in Filter', as opposed to AEGP which stands for 'After Effects General Plug-in'. The PF_ prefix is used throughout the SDK for parameters and classes related to filter plugins.

*Tags: `pipl`, `aegp`, `sdk`, `reference`*

---

## What functionality should be implemented in the Render function versus the 8-bit and 16-bit functions?

Your plugin's entry point function gets invoked with a RENDER or SMART_RENDER call. As long as your plugin responds to that call, you can implement whatever functionality you need. There is no strict separation required—you have flexibility in how you organize your plugin's main functionality across these functions.

*Tags: `render-loop`, `aegp`, `pipl`, `reference`*

---

## How can I group parameters into folders in an After Effects plugin?

Parameters can be grouped into folders (commonly referred to as "groups") by defining them between PF_ADD_TOPIC and PF_END_TOPIC macros. Topics can also be nested to create hierarchical parameter organization.

*Tags: `params`, `ui`, `pipl`, `sdk`*

---

## What is the function of the PiPL.r file in After Effects plugin development?

The PiPL.r (Plug In Property List) file was originally created to allow After Effects to get information about plug-ins without loading them, which was useful when computers had limited RAM (128MB). Nowadays it's mostly a legacy requirement. The values in the PiPL must correlate to the values set in the plugin's global setup call, or an error message will be sent during the plugin's launch indicating a mismatch. To set the correct values: put a breakpoint in the global setup function to see the numerical values applied to outflags and outflags2 variables, then copy these values to the PiPL file. Note that a clean rebuild is required for changes to take effect, as the pipl_tool only generates a new .rc file when one doesn't already exist.

*Tags: `pipl`, `plugin-properties`, `build`, `deployment`*

---

## How do you set AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 values in the PiPL.r file?

To set these flags correctly, place a breakpoint in the plugin's global setup function and observe the numerical values being applied to the outflags and outflags2 variables. Copy these exact numerical values to the corresponding fields in your PiPL.r file. The values must match what your plugin sets during initialization, or a mismatch error will occur at plugin launch. After making changes, perform a clean rebuild to ensure the pipl_tool regenerates the .rc file from your updated .r file.

*Tags: `pipl`, `params`, `build`, `debugging`*

---

## How can you keep UI and render threads in sync when automatically applying multiple effects in CC2015+?

When using Sequence Setup to automatically add effects to a layer, the UI and render copies of the project can become out of sync in CC2015+, causing verification failures and missing data effects. The solution is to use an AEGP (After Effects General Plug-in) with an idle hook instead of modifying the layer during Sequence Setup. The effect can set a flag when it's applied, and the AEGP's idle hook will then process the pending operation to add the additional effects. The AEGP can be bundled in the same binary as the effect plugins, but the AEGP's PiPL must be listed first in the bundle.

*Tags: `aegp`, `pipl`, `ui`, `smart-render`, `threading`, `plugin-architecture`*

---

## Is there a reference in the SDK documentation about bundling AEGP plugins with effect plugins?

According to the SDK guide, if you want to add an AEGP plug-in to the same binary as an effect plugin, the PiPL of the AEGP must be the first one in the resource list. This allows you to avoid creating a separate installer while still gaining access to AEGP features like idle hooks for proper effect synchronization.

*Tags: `aegp`, `pipl`, `sdk`, `plugin-architecture`, `reference`*

---

## How do you change the plugin name and menu in an After Effects SDK sample plugin?

The plug-in name and menu are set in the resource file (namePiPL.r). After you change it, you'll need to clean the build and re-build the project, otherwise the new settings will not apply in the compiled plug-in.

*Tags: `pipl`, `build`, `sdk`, `resource`*

---

## What does the PF_OutFlag2_SUPPORTS_QUERY_DYNAMIC_FLAGS error mean and how do you fix it?

The error 'effect cannot change non-dynamic flag bits during PF_Cmd_QUERY_DYNAMIC_FLAGS' occurs when there's a mismatch between the PiPL (Plug-In Property List) resource and the PF_Cmd_GLOBAL_SETUP implementation. The PF_OutFlag2_SUPPORTS_QUERY_DYNAMIC_FLAGS flag must be set consistently in both the PiPL and during PF_Cmd_GLOBAL_SETUP. On macOS, you can edit PiPL resources using Resorcerer with the PiPL template from the SDK, or use DeRez to create a .r file. The behaviors indicated during global setup must match those in the PiPL exactly. On Windows, ensure the .rc file is regenerated by cleaning and rebuilding the project.

*Tags: `pipl`, `macos`, `windows`, `build`, `debugging`*

---

## How do you hide a layer parameter in After Effects without breaking parameter evaluation?

To hide a layer parameter while maintaining evaluation, you must set the ui_flags during parameter setup using PF_PUI_NO_ECW_UI. If you hide the parameter later using AEGP_SetStreamFlag(), the parameter will stop evaluating. The correct approach is to hide it at initialization time rather than at runtime.

```cpp
def.ui_flags = PF_PUI_NO_ECW_UI;
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## What does the 'invalid filter 25::3' error mean when loading SDK examples in After Effects?

The 'invalid filter 25::3' error typically indicates a dependency issue with the compiled plugin. This often occurs when the Visual Studio compiler version or runtime libraries don't match what After Effects expects. For CS5 and above, plugins must be compiled as x64 (64-bit), not x86 (32-bit), as After Effects no longer loads x86 plugins. Use Dependency Walker to verify all dependencies are present and correct. Additionally, ensure you have the correct Visual C++ runtime libraries installed, such as MSVCR100D.DLL from the Microsoft Visual C++ Redistributable package.

*Tags: `sdk`, `build`, `windows`, `debugging`, `pipl`*

---

## How can I make an After Effects plugin compiled with CS6 SDK work with CS5 and 5.5?

You need to do more than just swap headers—you must also update the API version in the .r (PiPL) file. The recommended approach is to copy your code to a working CS5 project and recompile with the CS5 SDK. While Adobe states that suites are never removed or altered, rare deprecations have occurred (e.g., in CS6). If you need to support multiple versions, use preprocessor definitions to conditionally select the correct suite version for each target SDK version.

```cpp
#define SOME_SUITE someSuiteVer6  // for CS4
#define SOME_SUITE someSuiteVer7  // for CS5

// In code:
suites.SOME_SUITE()->someFunction("yo yo!");
```

*Tags: `sdk`, `cross-version`, `suites`, `pipl`, `build`*

---

## Why does After Effects report 'could not locate entrypoint' when loading a plugin?

The entrypoint function must be exported with the DllExport keyword on Windows. Without this declaration, After Effects cannot locate the EntryPointFunc symbol in the compiled plugin binary.

```cpp
DllExport PF_Err EntryPointFunc (
    PF_Cmd cmd,
    PF_InData *in_data,
    PF_OutData *out_data,
    PF_ParamDef *params[],
    PF_LayerDef *output,
    void *extra )
```

*Tags: `aegp`, `pipl`, `debugging`, `windows`, `build`*

---

## Why does the PiPL resource compiler fail with LNK1104 error when building an After Effects plugin in Visual Studio 2008?

The PiPL resource compiler and linker fail when the After Effects SDK is installed in a non-standard location. The SDK should be placed in the standard path (e.g., C:\Program Files\Adobe\After Effects CS6 SDK\) rather than in custom user directories. If you must use a different location, you can configure the custom build step to point to the tool in another location, but this requires additional configuration effort.

*Tags: `build`, `pipl`, `windows`, `visual-studio`, `sdk`*

---

## How can I get a Pixel Bender kernel's match name to use in an After Effects native plugin?

Create a new After Effects project with a composition containing a single solid layer. Apply your Pixel Bender plugin to the layer. Then run this ExtendScript to retrieve the match name: alert(app.project.item(1).layer(1).effect(1).matchName); This will return the match name (typically based on the id data in the Pixel Bender script) that you can use in your native plugin to make After Effects treat both as the same plugin for backward compatibility.

```cpp
alert(app.project.item(1).layer(1).effect(1).matchName);
```

*Tags: `scripting`, `params`, `reference`, `pipl`*

---

## What parameter type should be used to replace Pixel Bender percent sliders when porting to the After Effects SDK?

Use PF_ADD_FLOAT_SLIDERX instead of PF_ADD_PERCENT, since Pixel Bender implements percent sliders as float sliders. You can add the PF_ValueDisplayFlag_PERCENT flag to the PF_ADD_FLOAT_SLIDER call to display the float slider as a percentage in the UI without changing the underlying implementation.

```cpp
PF_ADD_FLOAT_SLIDERX("param_name", 0, 100, 0, 100, 1, PF_ValueDisplayFlag_PERCENT);
```

*Tags: `params`, `ui`, `pipl`*

---

## How do you create parameter groups in After Effects plugins to organize the UI?

Use the PF_ADD_TOPIC and PF_END_TOPIC macros to create parameter groups. First, define enum IDs for your topic and end topic in your header file. Then in your parameter setup, use PF_ADD_TOPIC with a group name and ID, add your parameters (checkboxes, points, etc.) between them, and close with PF_END_TOPIC using the corresponding end ID. This organizes related parameters into collapsible groups in the Effect Controls panel.

```cpp
// In .h file
enum {
  TOPIC_FEEDBACK_ID,
  SHOW_HISTOGRAM_DISK_ID,
  HIST_CORNER_ID,
  HIST_SIZE_ID,
  END_TOPIC_FEEDBACK_ID
};

// In .cpp file
PF_ADD_TOPIC("Histogram Overlay", TOPIC_FEEDBACK_ID);
AEFX_CLR_STRUCT(def);
def.flags = PF_ParamFlag_CANNOT_TIME_VARY;
PF_ADD_CHECKBOX(STR(StrID_ShowHist_Name), STR(StrID_ShowHist_Description), FALSE, 0, SHOW_HISTOGRAM_DISK_ID);
AEFX_CLR_STRUCT(def);
PF_ADD_POINT("Histogram location", HIST_LOCATION_X, HIST_LOCATION_Y, RESTRICT_BOUNDS, HIST_CORNER_ID);
AEFX_CLR_STRUCT(def);
PF_ADD_PERCENT("Histogram width", HIST_PCT_DFLT, HIST_SIZE_ID);
AEFX_CLR_STRUCT(def);
PF_END_TOPIC(END_TOPIC_FEEDBACK_ID);
```

*Tags: `params`, `ui`, `pipl`*

---

## How is the PiPL resource version number calculated in After Effects plugins?

The resource version is calculated using the PF_VERSION macro with bit shifting. The formula is: RESOURCE_VERSION = MAJOR_VERSION * 524288 + MINOR_VERSION * 32768 + BUG_VERSION * 2048 + STAGE_VERSION * 512 + BUILD_VERSION. In Visual Studio, you can hover over the enum value to see its calculated result and copy it to the resource file. On Windows, delete the .rc file before rebuilding to ensure a new resource version is generated when changes are made.

```cpp
out_data->my_version = PF_VERSION(MAJOR_VERSION, MINOR_VERSION, BUG_VERSION, STAGE_VERSION, BUILD_VERSION);
```

*Tags: `pipl`, `build`, `windows`*

---

## How can you create dynamic stroke parameters similar to the Paint effect in After Effects plugins?

Strokes are handled by the dynamicStreamSuite, which allows you to add, remove, and alter strokes and text layer parameters. However, custom parameters that behave like strokes cannot be created through the API. Two workarounds are: (1) Create your effect with hidden parameter groups (e.g., 50 groups, each with width, length, start, and end sliders) and unhide them as needed—this is simpler but limited by the number of compiled groups. (2) Create a dummy effect with the desired sliders and add new instances of it to the layer whenever more strokes are needed—this is more flexible and allows unlimited strokes but is significantly more complex to implement.

*Tags: `params`, `ui`, `sdk`, `aegp`, `pipl`*

---

## How can I hide an effect from the Effects menu while still allowing it to be added programmatically?

Set the PF_OutFlag_I_AM_OBSOLETE flag during global setup, and the effect will not appear in the effects list but can still be added using addEffect from code.

*Tags: `ui`, `pipl`, `aegp`*

---

## Why won't my After Effects plugin built with SDK 7.0 load on Mac CS4 when it works fine on Windows?

The issue is related to architecture support differences between SDK versions. AE7 was the last version to run on PPC processors; on Intel chips it ran emulated. CS3 was the first to run native on Intel chips and SDK 8.0 (CS3) was the first SDK to support plugins running native on Intel chipsets (and PPC). The CS3 SDK offers backwards compatibility to AE7, allowing plugins to work across AE7, CS3, and CS4 on both PC and Mac. In contrast, the CS4 SDK is not backwards compatible. To achieve cross-version compatibility on Mac, you should check the resource file declarations in the CS3 SDK samples, which contain separate entry point declarations for Intel and PPC architectures.

*Tags: `macos`, `windows`, `sdk`, `build`, `cross-platform`, `pipl`*

---

## How can I control layer copy-paste operations in an After Effects Effect plugin to restrict pasting within the same composition?

Use an AEGP (After Effects General Plugin) rather than an Effect plugin to intercept copy-paste operations. Register a command hook using AEGP_RegisterCommandHook to be notified whenever the user copies or pastes. This allows you to check the collection being pasted and remove or delete effects as needed. You can use AEGP_Command_ALL with AEGP_RegisterCommandHook to discover the command numbers for copy and paste operations. Attempting this from within an Effect plugin alone is not practical.

*Tags: `aegp`, `pipl`, `layer-checkout`, `scripting`*

---
