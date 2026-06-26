# Premiere Pro

> 105 Q&As · source: AE plugin dev community Discord

### When is PF_Cmd_GLOBAL_SETUP called - during AE loading or when applying the effect?

In After Effects, GlobalSetup is called the first time the user applies the effect to a layer, not during the AE loading screen (during loading, AE only scans PiPLs). In Premiere Pro, GlobalSetup is called when the app is loading. You can show dialogs during GlobalSetup in AE since it happens after the app is fully loaded.

*Tags: `global-setup`, `initialization`, `plugin-lifecycle`, `premiere`*

---

### How do you set per-host out_flags when supporting both AE and Premiere?

PiPL doesn't support per-host flags. Instead, set different flags in GlobalSetup by checking in_data->appl_id. For example, to use PF_OutFlag_NON_PARAM_VARY only in AE: check if(in_data->appl_id != 'PrMr') before setting the flag. Note: PiPLs are no longer used by Premiere Pro, so you can write PiPL for AE only. Premiere uses PluginDataEntryFunction / PF_REGISTER_EFFECT instead.

```cpp
if (in_data->appl_id != 'PrMr') {
    out_data->out_flags = AE_OUT_FLAGS;
} else {
    out_data->out_flags = PR_OUT_FLAGS;
}
```

*Tags: `global-setup`, `out-flags`, `per-host`, `pipl`, `premiere`*

---

### How do you get the AE/Premiere version number programmatically?

in_data->version major/minor are not reliably updated between AE versions. Options: (1) Use ExtendScript via AEGP_ExecuteScript with 'app.version' - but may fail with 'cannot run script while modal dialog waiting'. (2) Use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP to get the AE install folder path, then parse the version from the folder name or binary metadata. (3) For plugins with Premiere GPU entry points, PPixSuite's appinfo provides version data (format like '24.3', add 2000 for year).

*Tags: `aegp`, `compatibility`, `extendscript`, `premiere`, `version-detection`*

---

### What is PF_REGISTER_EFFECT and will it replace PiPL?

AE's internal plugins now use two entry points: EffectMainExtra and PluginDataEntryFunction, defined via PF_REGISTER_EFFECT / PF_Register_effect_ext2. This appears in SDK samples like SDK_Invert_ProcAmp. While it seems like it could eventually replace PiPL, as of now PiPL is still required for AE - attempting to build without a PiPL/rsrc still results in AE not finding the effect. Premiere Pro no longer requires PiPL and uses this newer system. No official announcement about PiPL deprecation has been made.

*Tags: `future`, `pf-register-effect`, `pipl`, `plugin-entry-point`, `premiere`*

---

### Is checking out the current layer with prior effects applied possible in Premiere?

In AE, you can use checkout_layer_pixels in SmartRender to get the layer with prior effects applied. However, this does not work in Premiere. In Premiere, there is no straightforward way to check out the current layer with prior effects applied - PF_CHECKOUT_PARAM always returns the pre-effects layer. A potential hacky workaround might involve calling Premiere's JS scripting API to render, but this is not a clean solution.

*Tags: `checkout-layer`, `limitations`, `premiere`, `prior-effects`, `smart-render`*

---

### How can I trigger a plugin refresh in Premiere when CEP updates hidden parameters, since Premiere doesn't treat script interactions as user actions like AE does?

There is no known workaround for this. Premiere does not consider script interactions as user actions, so PF_UserChangedParam won't be triggered from CEP. A practical workaround is to use external dependencies watching for file changes (though it's not very reactive) or add a button parameter to force a manual refresh.

*Tags: `cep`, `parameter-update`, `premiere`, `script-interaction`, `workaround`*

---

### Why can I only select the first two items in a PF_ADD_POPUPX dropdown with 3 elements in Premiere, and the third snaps back to 2?

This can be caused by opening a Premiere project with the transition already applied (stale cached state). Applying the transition plugin fresh resolves the issue and the dropdowns work normally. Also ensure that each popup choice string has the pipe separator '|' at the end of each line (except the last), and that you have a proper AEFX_CLR_STRUCT(def) before the popup definition.

```cpp
AEFX_CLR_STRUCT(def);

PF_ADD_POPUPX("Color", 3, 2, 
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Tags: `cache-bug`, `dropdown`, `parameter`, `popup`, `premiere`, `transition`*

---

### How do I copy pixel data row by row between Premiere buffers and a 3D API (Vulkan/OpenGL), and why does memcpy crash when copying from the 3D API back to Premiere?

For copying from Premiere to the 3D API, iterate row by row using the row bytes from the Premiere world definition. The crash when copying back likely relates to incorrect buffer mapping or row byte calculations. In Premiere, row bytes can be negative, and image buffers are 16-byte aligned with possible padding at the end of each line. For Vulkan specifically, check the actual VkSubresourceLayout via vkGetImageSubresourceLayout to ensure correct copy offsets. An access violation usually means a calculation is wrong or one of the buffers is not mapped correctly.

```cpp
// Premiere to 3D API (works):
Void PrPtr = (Char)PRData + y * PrRowByte;
Void ApiPtr = (Char)ApiData + y * APIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);

// APIRowByte = width * 4 (channels) * sizeOfPixel (1 for 8-bit, 4 for 32-bit)
// PrRowByte comes from Premiere world def, often negative
```

*Tags: `memcpy`, `negative-row-bytes`, `opengl`, `pixel-buffer`, `premiere`, `row-bytes`, `vulkan`*

---

### How can I use multithreading for pixel iteration in Premiere, since the AE Iterate8Suite doesn't work correctly?

The AE Iterate8Suite may not work correctly in Premiere and can cause rendering issues (e.g., sequence thumbnail updates but program window doesn't). For 8-bit BGRA, Premiere has its own dedicated iteration suite. For 32-bit float, you need to write your own iteration function. Use std::parallel or equivalent for multithreading. The SDK Noise example demonstrates BGRA and VUYA colorspace handling. The iterate suite should work in 8-bit ARGB (Suitev2 since 2022), but for BGRA you need the special Premiere suite.

*Tags: `bgra`, `colorspace`, `iterate-suite`, `multithreading`, `pixel-iteration`, `premiere`*

---

### How should I handle color spaces in Premiere plugins, and can I avoid dealing with multiple color modes?

You can use basic render in 8/16-bit without handling all color spaces -- it works but is not optimized, and you won't get the 32-bit icon next to the plugin. You can choose to support only BGRA and skip VUYA. In GlobalSetup, you select which colorspaces to support. However, 32-bit float is required for color grading workflows to access values outside the 0-1 range. The SDKNoise example in the AE SDK demonstrates both BGRA and VUYA colorspace handling for Premiere.

*Tags: `32-bit`, `bgra`, `color-grading`, `colorspace`, `global-setup`, `premiere`, `vuya`*

---

### Does Premiere support SmartFX for requesting only a portion of the input buffer?

Premiere does not support SmartFX. You cannot officially request only a specific region of the buffer like you can in AE. As a workaround, you can manually iterate through the data using row bytes to delimit the part you need. Be careful with negative row byte values in Premiere.

*Tags: `buffer`, `premiere`, `region-of-interest`, `row-bytes`, `smartfx`, `transition`*

---

### Is there an equivalent to PF_ABORT() in Premiere for cancelling long renders?

PF_ABORT(in_data) does not work in Premiere -- it never sets err to PF_Interrupt_CANCEL even when scrubbing the timeline with frames taking over half a second to render. There may be a realtime flag that hints whether the effect should be treated as realtime or not, but no confirmed workaround for render cancellation in Premiere was identified.

*Tags: `pf-abort`, `premiere`, `realtime`, `render-cancel`, `scrubbing`*

---

### Why does Premiere give a NULL input_world in the Render call when compiled with fast optimizations (-Ofast), but works without optimizations?

This is a known issue that occurs in Premiere but not AE when using -Ofast compiler optimizations. A workaround is to add __attribute__((optnone)) before the render function to disable optimization for that specific function while keeping the rest of the plugin optimized. Another suggestion is to check if the input world is null and return PF_Err_OUT_OF_MEMORY (not bad param), which may cause Premiere to retry the render. Also verify your colorspace setup in GlobalSetup and consider trying BGRA instead of ARGB.

```cpp
__attribute__ ((optnone))
static PF_Err Render(...) {
    // render code here
}
```

*Tags: `compiler`, `null-input-world`, `ofast`, `optimization`, `premiere`, `render`, `workaround`*

---

### Why do keyframed parameters on adjustment layers in Premiere always return the value at the first keyframe regardless of current time?

This is caused by the PF_OutFlag_NON_PARAM_VARY flag. When this flag is set, keyframes on adjustment layers in Premiere always return the first keyframe value during checkout param, regardless of current time. This affects sliders, angles, checkboxes, and 2D points. The bug occurs in Premiere v23 and v24. Removing the NON_PARAM_VARY flag fixes the issue. In AE, this flag can be replaced using MIX_GUID during the pre_render thread. Note that apart from this bug, the flag does nothing useful in Premiere.

*Tags: `adjustment-layer`, `bug`, `checkout-param`, `keyframes`, `non-param-vary`, `premiere`*

---

### What causes the 'Not able to acquire AEFX Suite' error in Premiere?

This error can occur during playback in Premiere when code attempts to acquire an AE-specific suite that is not available in the Premiere host. Check if you have a shared function that conditionally calls AEFX suites based on host detection (e.g., if app_id != 'PrMr' then call AEFX suite). The host detection condition might be returning the wrong result, causing Premiere to try acquiring AE-only suites. It could also be a bug with a Premiere suite itself.

*Tags: `aefx-suite`, `error`, `host-detection`, `premiere`, `suite-acquisition`*

---

### What is UXP for Premiere Pro and how does it relate to CEP panel migration?

UXP (Unified Extensibility Platform) for Premiere Pro entered public beta in December 2024. It is the successor to CEP for building panels and extensions. Initially it is a web-based development environment, but a hybrid UXP+C++ approach is planned. For C++ computational tasks without the hybrid mode, you would need a service with IPC. The Premiere UXP beta is currently limited in functionality but expected to expand. Plugin developers with existing CEP panels should begin evaluating migration. Resources include the Adobe Creative Cloud Developer forums and the Hyperbrew blog (hyperbrew.co/blog/premiere-pro-uxp-beta). Bolt UXP (hyperbrew.co/resources/bolt-uxp/) is a tool to help with UXP development.

*Tags: `beta`, `cep`, `hybrid-cpp`, `migration`, `panel`, `premiere`, `uxp`*

---

### How do I clear Premiere's plugin cache to fix rendering or display glitches?

Hold Shift or Alt when opening Premiere to clean the plugin cache. This has been known to solve problems where rendering appears incorrect or display issues occur.

*Tags: `cache`, `plugin-cache`, `premiere`, `troubleshooting`*

---

### How do you prevent an After Effects plugin from appearing in Premiere Pro?

Set the PF_OutFlag_I_AM_OBSOLETE flag in out_data->out_flags when the application ID is 'PrMr' (Premiere Pro). This is done by checking if(in_data->appl_id == 'PrMr') and then setting out_data->out_flags = PF_OutFlag_I_AM_OBSOLETE;

```cpp
if(in_data->appl_id == 'PrMr')
{
    out_data->out_flags = PF_OutFlag_I_AM_OBSOLETE;
}
```

*Tags: `params`, `pipl`, `premiere`*

---

### How do you toggle a UI element's visibility on and off in an After Effects plugin?

To toggle UI element visibility, use the AEGP ParamUI thread approach instead of directly modifying ui_flags. Get the effect and stream references using AEGP_GetNewEffectForEffect and AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag with the AEGP_DynStreamFlag_HIDDEN flag. The issue with the direct ui_flags approach is that the bitwise NOT operator (!PF_PUI_INVISIBLE) doesn't properly clear the flag—you need to use the AEGP suite methods instead. Note that Premiere has a different method for accomplishing this.

```cpp
static PF_Err
changeParamVisibility(PF_InData            *in_data,
                      PF_OutData            *out_data,
                      PF_ParamDef          *paramsDef,
                      PF_ParamIndex        paramIndex,
                      PF_Boolean           paramVisibleB)
{
    PF_Err                err                    = PF_Err_NONE,
    err2                = PF_Err_NONE;
    global_dataP        globP                = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler        suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr')
    {
        AEGP_EffectRefH            meH                = NULL;
        AEGP_StreamRefH      currStreamH        = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex,     &currStreamH));
        if (meH && currStreamH)
        {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
        if (meH){
            ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
        }
        if ( currStreamH ){
            ERR2(suites.StreamSuite2()->AEGP_DisposeStream( currStreamH ));
        }
    }
    return err;
}
```

*Tags: `aegp`, `params`, `premiere`, `ui`*

---

### Why do third-party GPU debuggers crash or fail when used with After Effects?

Third-party GPU debuggers are designed for games with permanent instances, making them incompatible with plugins running inside host applications like After Effects or Premiere. Even modern API debuggers struggle with this architecture. RenderDoc, for example, tends to capture the host application's DirectX instead of the plugin's OpenGL, and gDEBugger crashes after tracing.

*Tags: `debugging`, `gpu`, `opengl`, `premiere`, `vulkan`*

---

### Why does a plugin act weird on Media Encoder and render without the effect applied when queued from After Effects?

The issue is likely related to static global variables in the plugin or elements defined in globalData or during global initialization thread. These can cause problems across different Media Encoder versions.

*Tags: `debugging`, `memory`, `premiere`, `threading`*

---

### Does Media Encoder load AEGP plugins to render After Effects projects?

No, AEGP plugins are not supported by Media Encoder (AME). If your plugin needs an AEGP plugin to render, you will need to get your AEGP as a standalone/command line tool to work around this limitation.

*Tags: `aegp`, `deployment`, `premiere`*

---

### Is it possible to include multiple PiPLs in the same file?

Yes, it is possible to include multiple plug-ins (both AEGPs and effects) in the same file using multiple PiPLs, but it is not recommended. If there are PiPLs for both AEGPs and effects in the same file, the AEGPs must come first. No other hosts, not even Premiere Pro, support multiple PiPLs pointing to multiple effects within the same .dll or code fragment. The recommendation is to use one PiPL and one plug-in per code fragment.

*Tags: `aegp`, `build`, `pipl`, `premiere`*

---

### When is PF_Cmd_GLOBAL_SETUP called in After Effects and Premiere?

In After Effects, GLOBAL_SETUP is called the first time the user applies the effect plugin to a layer. In Premiere, it is called when the application is loading.

*Tags: `aegp`, `pipl`, `premiere`, `ui`*

---

### How do you check out a frame of the current layer after prior effects are applied instead of before them?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects. However, this approach may not work in Premiere Pro. An alternative approach using Cmd_RENDER may be needed for cross-application compatibility.

*Tags: `layer-checkout`, `premiere`, `render-loop`, `smartfx`*

---

### What is the official way to determine pixel format in Premiere using the AE SDK?

The official way to determine pixel format in Premiere using the AE SDK is to call the PF_PixelFormatSuite1, as seen in the SDK_Noise sample in the AE SDK since AE CS5. Using PF_WorldFlag_RESERVED1 was an unofficial/internal workaround by Adobe developers that is no longer recommended.

*Tags: `params`, `pipl`, `premiere`*

---

### How can you access the Adobe application version from an After Effects plugin?

You can use ExtendScript with 'app.version' via the aegp execute script. Alternatively, if your plugin supports both After Effects and Premiere Pro with GPU features, you can access the version through the Premiere Pro PICA Suite AppInfo, which is called before global setup. The version data is typically formatted like '24.3', and you need to add 2000 to convert it. For versions with additional components like '24.3.x', some parsing is required.

*Tags: `aegp`, `cross-platform`, `premiere`, `scripting`*

---

### Can global_outflags be set differently for specific host applications in PIPL?

Yes, you can set flags per host in global setup by checking the host application ID. Use a conditional check on in_data->appl_id to determine which host is running, then set different out_flags accordingly. For example, you can check if the application is not Premiere Pro (appl_id != 'PrMr') and apply different flags for After Effects versus other hosts.

```cpp
if (in_data->appl_id != 'PrMr') {
  out_data->out_flags = PR_OUT_FLAGS;
}
```

*Tags: `cross-platform`, `params`, `pipl`, `premiere`*

---

### Is it possible in PiPL to set global_outflags specific to a host app, such as removing a flag for Premiere while keeping it in After Effects?

Yes, it is possible to set global_outflags specific to host apps in PiPL files. You can use conditional logic or host-specific PiPL configurations to remove flags like no_params_vary for Premiere while maintaining them in After Effects.

*Tags: `params`, `pipl`, `premiere`*

---

### Can a single After Effects plugin file contain multiple PiPL entries with different entry points?

Yes, a plugin can have multiple PiPL entries allowing different entry points and different plugins in one file. However, this functionality is not consistently supported throughout After Effects and is not supported at all in Premiere Pro, so it has not gained much traction in practice.

*Tags: `pipl`, `plugin-architecture`, `premiere`*

---

### What are the advantages of writing host and plugin format-independent code across multiple applications?

Writing format-independent code for multiple hosts (AE, Premiere Pro, OFX, Nuke, Frei0r, etc.) frees up significant resources by allowing you to write the processing code once and only occasionally update the wrapper projects, rather than maintaining separate implementations for each host.

*Tags: `code-architecture`, `cross-platform`, `deployment`, `premiere`*

---

### Is there a reliable way to get the current After Effects or Premiere version number (e.g., 2025)?

The in_data->version major and minor fields do not appear to be reliably updated between versions, returning the same values on both 2024 and 2025. This approach may work on Mac AE but needs verification on Premiere and Windows.

*Tags: `debugging`, `deployment`, `macos`, `premiere`, `windows`*

---

### How can you hide parameters in Premiere within UpdateParamsUI?

To hide parameters in Premiere, use PF_UpdateParamUI with the PF_PUI_INVISIBLE flag set on the param's ui_flags. It's recommended to also set PF_PUI_DISABLED to avoid backend conflicts. The key is to ensure no error code (err) is set before calling PF_UpdateParamUI, as the function only executes when err is 0. Use bitwise OR to set flags (paramsDef[paramIndex].ui_flags |= PF_PUI_INVISIBLE) and bitwise AND with negation to unset them (paramsDef[paramIndex].ui_flags &= ~PF_PUI_INVISIBLE).

```cpp
if (!err && in_data->appl_id == kAppID_Premiere) {
    paramsDef[paramIndex].ui_flags |= PF_PUI_INVISIBLE;
    paramsDef[paramIndex].ui_flags |= PF_PUI_DISABLED;
    ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref,
        paramIndex,
        &paramsDef[paramIndex]));
}
```

*Tags: `params`, `premiere`, `ui`*

---

### Does the Mediacore plugin loading issue affect Premiere Pro 2026?

Yes, plugins in the Mediacore folder work in Premiere Pro 2023, 2024, and 2025, but do not show up or load in Premiere Pro 2026, indicating a similar compatibility issue across Adobe applications.

*Tags: `deployment`, `macos`, `premiere`*

---

### How can you prevent an After Effects plugin from being detected and displayed in Premiere's effects list?

Return an error in global setup, since Premiere runs global setup for all plugins during application startup. If the plugin returns an error during this initialization phase, it should be hidden from the effects list.

*Tags: `debugging`, `pipl`, `premiere`*

---

### Does PF_OutFlag_I_AM_OBSOLETE flag work to hide a plugin in Premiere Pro while keeping it visible in After Effects?

The flag will hide the plugin in both AE and Premiere Pro, so it won't achieve the desired result of hiding only in PPro unless you conditionally set it after checking the host name. You can make it work by checking the host application first and only applying the flag when running in Premiere Pro.

*Tags: `cross-platform`, `deployment`, `pipl`, `premiere`*

---

### How can you trigger parameter updates in Premiere from CEP script interactions when PF_UserChangedParam doesn't work?

When working with AE/Premiere plugins updated by CEP backend data, using a hidden parameter updated via script that calls PF_UserChangedParam works well in After Effects, but Premiere does not recognize script interactions as user actions. Alternative approaches include monitoring external file dependencies for reactivity (though this can be slow), or implementing a manual button to force refresh as a workaround.

*Tags: `cep`, `cross-platform`, `params`, `premiere`, `scripting`*

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

### How do you handle different color spaces when developing plugins for both After Effects and Premiere?

In After Effects, plugins typically work in ARGB colorspace. Premiere uses BGRA_8u with a special suite for 8-bit processing. For 32-bit float, you need to write your own conversion function. You can use the Iterate8Suite1 (or SuiteV2 since 2022) for ARGB, but Premiere requires a special BGRA suite. You don't have to support all color spaces—you can optimize for BGRA only and skip YUVA if needed. Basic rendering in 8/16 bit works without optimization, though you won't get the 32-bit icon next to the plugin. Check the SDK noise example for reference implementations of BGRA and YUVA colorspace handling.

*Tags: `argb`, `bgra`, `color-space`, `optimization`, `premiere`, `render-loop`*

---

### What is the recommended approach for multithreading in Premiere plugins when the iterate suite doesn't work?

If the Iterate8Suite doesn't work reliably in Premiere (particularly with certain colorspace configurations), you should implement your own multithreading code instead of relying on the suite. Standard approaches include using std::parallel on Windows and equivalent threading libraries for Mac (like Grand Central Dispatch or pthreads). The iterate suite should theoretically work for 8-bit ARGB, but if you encounter issues, custom multithreading implementations are a stable fallback.

*Tags: `bgra`, `macos`, `multithreading`, `performance`, `premiere`, `threading`, `windows`*

---

### Is there a reference implementation for handling BGRA and YUVA colorspaces in Premiere plugins?

Yes, the SDK noise example in the After Effects SDK demonstrates proper handling of both BGRA and YUVA colorspaces for Premiere. This example shows how to use the special BGRA suite for 8-bit processing and serves as a reference for implementing colorspace support in plugins targeting both AE and Premiere.

*Tags: `bgra`, `open-source`, `premiere`, `reference`, `sdk-example`, `yuva`*

---

### How do you troubleshoot colorspace and iteration suite issues in Premiere?

When working with Premiere's colorspace and iteration suite, start by clarifying which specific colorspace was chosen, as this is often the root cause of issues. The problem may stem from colorspace selection or configuration within the iteration workflow.

*Tags: `color`, `debugging`, `premiere`*

---

### How can you limit the output buffer to a specific region when Premiere doesn't support smartFX for transitions?

Since Premiere Pro does not support smartFX like After Effects does, you cannot use the smartFX output_rect mechanism to specify a smaller region of interest. When building transitions for Premiere, the plugin receives frames at the full sequence resolution even if the source footage is smaller. You need to work with the full buffer dimensions provided by Premiere's API rather than requesting a subset.

*Tags: `output-rect`, `premiere`, `smartfx`, `transition`*

---

### How do you debug a Premiere plugin from Xcode and load debug symbols?

Ensure that you have 'Generate Symbols' set to 'Yes' in your Xcode scheme settings. If symbols are set to 'No', breakpoints will not work and Xcode will not be able to load the necessary debugging information for your Premiere plugin.

*Tags: `build`, `debugging`, `premiere`, `xcode`*

---

### Does PF_ABORT() work in Premiere Pro the same way it does in After Effects?

No, PF_ABORT(in_data) does not abort rendering in Premiere Pro like it does in After Effects. Even when frames take over half a second to render and PF_ABORT() is called, Premiere does not set err to PF_Interrupt_CANCEL as expected. In Premiere, you appear to be required to completely finish rendering every frame that is requested, without the ability to abort mid-render like in After Effects.

*Tags: `debugging`, `premiere`, `render-loop`*

---

### Why does Premiere pass NULL for input_world in Render calls when compiled with fast optimizations, while After Effects doesn't have this issue?

When a plugin is compiled with fast optimizations, Premiere may pass input_world = NULL in the Render call, whereas the same code without optimizations receives the data correctly. This issue does not occur in After Effects. The root cause appears to be related to compiler optimization levels affecting how Premiere handles input data passing to plugins.

*Tags: `debugging`, `premiere`, `render-loop`, `smartfx`*

---

### How should you handle null inoutworld parameters in Premiere Pro plugins?

If you set a condition to check if inoutworld is null and return an error, Premiere Pro will redo the render. This is a valid strategy for handling null world pointers in render functions.

```cpp
if (inoutworld == null) return err;
```

*Tags: `debugging`, `mfr`, `premiere`, `render-loop`*

---

### Why does a plugin on a Premiere adjustment layer only receive the first keyframe value regardless of current time when checking out parameters?

This is a known issue in Premiere where plugins applied to adjustment layers with multiple keyframes on parameters receive only the value at the first keyframe, regardless of the current playhead time during parameter checkout. The issue affects all parameter types tested including sliders, angles, checkboxes, and 2D points. This behavior does not occur in After Effects, suggesting it is specific to Premiere's adjustment layer implementation.

*Tags: `debugging`, `layer-checkout`, `params`, `premiere`*

---

### What causes a plugin to not work properly in Premiere when it works in After Effects?

The issue can stem from the no_params_vary flag. In After Effects, this can be replaced using MIX_GUI during the pre_render thread. However, this flag has no effect in Premiere, so alternative approaches are needed for cross-application compatibility.

*Tags: `aegp`, `cross-platform`, `params`, `premiere`, `render-loop`*

---

### Why does a plugin fail with 'Not able to acquire AEFX Suite' error when playing in Premiere?

The issue may occur during render/playback if the plugin calls AEFX Suite in a shared thread or has incorrect host application detection logic. Check if there is conditional code that calls AEFX Suite only when the host is NOT Premiere (e.g., `if (app_id != 'PrMr') => call AEFX_suite`), but the host detection returns an incorrect value in Premiere. This can cause the plugin to attempt AEFX Suite calls in Premiere where they are not available. Premiere can also have unexpected behavior with certain suites, such as colorspace-related ones.

*Tags: `aegp`, `debugging`, `premiere`, `threading`*

---

### How long will the CEP engine remain available in Premiere Pro before the transition to UXP is required?

This question was asked but not answered in the conversation. Erin F. shared information about UXP for Premiere Pro entering public beta and encouraged developers with CEP panels to communicate their migration needs to the Premiere Pro team, but did not provide a specific timeline for CEP deprecation.

*Tags: `cep`, `migration`, `premiere`, `uxp`*

---

### What is the status of UXP support for Premiere Pro and where can I learn more about it?

UXP for Premiere Pro is now in public beta. Developers with existing CEP panels should review the announcement and communicate their migration requirements to the Premiere Pro team. More information is available at the Creative Cloud Developer Forums: https://forums.creativeclouddeveloper.com/t/uxp-now-available-in-premiere-pro-beta/8795

*Tags: `cep`, `deployment`, `migration`, `premiere`, `uxp`*

---

### Where can I find information about Premiere UXP Beta?

Alex Bizeau from Maxon published a blog post with information on Premiere UXP Beta. The blog post was shared as a resource for developers interested in learning more about the Premiere UXP Beta.

*Tags: `premiere`, `reference`, `resource`, `uxp`*

---

### What is the best documentation to get started with UXP for Premiere Pro?

According to the discussion, there is a comprehensive article that contains all the links for UXP documentation, forums, and other resources for Premiere Pro development. UXP in Premiere Pro is currently limited but will be expanded soon.

*Tags: `premiere`, `reference`, `ui`, `uxp`*

---

### Can C++ be integrated as a backend for UXP plugins in Premiere Pro, or is IPC required for computational tasks?

There is a new hybrid UXP approach that combines HTML and C++ backends, though the exact implementation details were not fully explored in the conversation. This hybrid solution is currently in beta and represents a promising development for allowing computational tasks without requiring a separate service with IPC.

*Tags: `build`, `premiere`, `uxp`*

---

### What is Bolt UXP and how does it relate to Premiere Pro plugin development?

Bolt UXP is a resource mentioned in the discussion as relevant to Premiere Pro UXP development. Justin shared this as a reference for developers looking to build UXP-based plugins.

*Tags: `open-source`, `premiere`, `tool`, `uxp`*

---

### How do you toggle the visibility of a UI parameter in After Effects?

To toggle parameter visibility, use the DynamicStreamSuite2 in the ParamUI thread rather than directly manipulating ui_flags. The issue with toggling ui_flags is that the bitwise NOT operator (!) doesn't work as expected for flag manipulation. Instead, get the effect and stream references, then use AEGP_SetDynamicStreamFlag with the AEGP_DynStreamFlag_HIDDEN flag. Set the last parameter to !paramVisibleB where paramVisibleB is true to show and false to hide the parameter. Note that Premiere Pro requires a different approach.

```cpp
static PF_Err
changeParamVisibility(PF_InData            *in_data,
                      PF_OutData            *out_data,
                      PF_ParamDef          *paramsDef,
                      PF_ParamIndex        paramIndex,
                      PF_Boolean           paramVisibleB)
{
    PF_Err                err                    = PF_Err_NONE,
    err2                = PF_Err_NONE;
    global_dataP        globP                = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler        suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr')
    {
        AEGP_EffectRefH            meH                = NULL;
        AEGP_StreamRefH      currStreamH        = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex,     &currStreamH));
        if (meH && currStreamH)
        {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
        if (meH){
            ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
        }
        if ( currStreamH ){
            ERR2(suites.StreamSuite2()->AEGP_DisposeStream( currStreamH ));
        }
    }
    return err;
}
```

*Tags: `aegp`, `params`, `premiere`, `ui`*

---

### Does Media Encoder load AEGP plugins?

No, Media Encoder does not load AEGP plugins. If your workflow relies on AEGP plugins for rendering, you cannot use Media Encoder as an alternative.

*Tags: `aegp`, `deployment`, `premiere`*

---

### Is Media Encoder compatible with AEGP plugins for rendering After Effects projects?

AEGP plugins are not supported by Adobe Media Encoder (AME). If your plugin requires AEGP functionality to render, you will need to convert the AEGP plugin into a standalone or command-line tool to use with Media Encoder.

*Tags: `aegp`, `deployment`, `premiere`, `tool`*

---

### Is it possible to include multiple PiPLs in a single plugin file, and what are the recommendations?

Yes, it is technically possible to include multiple PiPLs (both AEGPs and effects) in the same file, but it is not recommended. If you do use multiple PiPLs in the same file, AEGPs must come first. However, no other hosts (not even Premiere Pro) support multiple PiPLs pointing to multiple effects within the same .dll or code fragment. The recommendation from the SDK is to use one PiPL and one plugin per code fragment, especially to avoid shipping new builds of all plugins when updating just one.

*Tags: `aegp`, `deployment`, `pipl`, `premiere`*

---

### When is PF_Cmd_GLOBAL_SETUP called in After Effects versus Premiere Pro?

In After Effects, PF_Cmd_GLOBAL_SETUP is called the first time the user applies the effect plugin to a layer. In Premiere Pro, it is called when the application is loading. The timing differs between the two host applications.

*Tags: `aegp`, `pipl`, `premiere`, `reference`*

---

### How do you check out a frame of the current layer after prior effects are applied in After Effects?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects have been applied. This allows you to access the layer state after upstream effects like Exposure have been rendered, but before the current effect processes it. However, this approach may not work in Premiere Pro, requiring alternative solutions for cross-application compatibility.

*Tags: `aegp`, `layer-checkout`, `premiere`, `render-loop`, `smartfx`*

---

### How can you access layer pixels in Premiere Pro during parameter changes if checkout_layer_pixels is not available in Cmd_RENDER?

checkout_layer_pixels is available in PF_SmartRenderCallbacks for After Effects, but for Premiere Pro compatibility, you need to access layer pixels during the user changed parameter callback instead of the render command. The approach involves handling pixel checkout in the parameter change event rather than the standard render path.

*Tags: `aegp`, `cross-platform`, `layer-checkout`, `params`, `premiere`*

---

### How can you detect the After Effects version from within a plugin?

You can use ExtendScript with 'app.version' via the AEGP execute script. Alternatively, if your plugin supports both After Effects and Premiere Pro with GPU acceleration, you have access to a Premiere Pro PICA Suite AppInfo that provides the version. The version data is in format like 24.3, and you need to add 2000 to get the actual version number (e.g., 24.3 becomes 2024.3). Some versions may include additional components like 24.3.x which require parsing.

*Tags: `aegp`, `gpu`, `premiere`, `scripting`, `version-detection`*

---

### How do you debug 'failed to load' errors in After Effects and 'filter offline' errors in Premiere Pro?

When debugging plugin loading failures, check whether the error occurs before global_setup is called. If it happens before global_setup, it may indicate a library dependency issue loaded by AE/Premiere in a newer version that conflicts with your plugin. Test on both Windows and macOS to determine if the issue is platform-specific, as the same plugin may work on one platform but fail on another.

*Tags: `cross-platform`, `debugging`, `deployment`, `macos`, `premiere`, `windows`*

---

### Can you set global_outflags in a pipl file to be host-application specific?

The user asked whether pipl files support setting global_outflags with host-app-specific conditions. They wanted to disable the no_params_vary flag for Premiere due to a bug while keeping it enabled in After Effects. No definitive answer was provided in the conversation, so this question remains unresolved.

*Tags: `aegp`, `params`, `pipl`, `premiere`*

---

### How can you set different global_outflags for different host applications in a PIPL plugin?

You can set flags per host in global setup by checking the appl_id in in_data. For example, you can conditionally set out_flags based on whether the host is Premiere or After Effects: if (in_data->appl_id != 'PrMr') { out_data->out_flags = PR_OUT_FLAGS; }. This allows you to remove flags like no_params_vary for Premiere while using them in After Effects.

```cpp
if (in_data->appl_id != 'PrMr') {
  out_data->out_flags = PR_OUT_FLAGS;
}
```

*Tags: `aegp`, `cross-platform`, `pipl`, `premiere`*

---

### Is it possible in PiPL to set global_outflags specific to a host application like Premiere?

Yes, you can set global_outflags with host-specific conditions in PiPL files. One approach is to use GUID-mix with time values to conditionally apply flags like no_params_vary for different host applications, allowing you to remove a flag for Premiere while keeping it enabled in After Effects.

*Tags: `aegp`, `params`, `pipl`, `premiere`*

---

### What are the advantages of writing host and plugin format-independent code for multiple platforms?

Writing host and plugin format-independent code frees up significant resources by allowing the processing code to be written once and only occasionally requiring wrapper project updates. This approach has proven effective across multiple platforms including After Effects, Premiere Pro, OFX, Nuke, and Frei0r. For example, when porting the GMIC plugin suite (containing ~2000 plugins) to OFX, all plugins fit in a single file, whereas for AE/Premiere Pro, 2000 tiny individual loader plugins had to be created. The OFX SDK has also benefited from this approach by eliminating detours and inconveniences present in the AE SDK, such as PiPL requirements.

*Tags: `aegp`, `cross-platform`, `ofx`, `plugin-architecture`, `premiere`*

---

### How can you reliably get the current After Effects or Premiere version number (e.g., 2025)?

James Whiffin noted that in_data->version major and minor fields are not being reliably updated between versions like 2024 and 2025, returning the same values. Jonah reported that accessing the version appears to work on macOS After Effects, though behavior on Premiere and Windows still needs verification.

*Tags: `aegp`, `cross-platform`, `debugging`, `macos`, `premiere`, `windows`*

---

### How do you hide parameters in Premiere Pro using PF_UpdateParamUI?

To hide parameters in Premiere Pro, check if the application is Premiere using `in_data->appl_id == kAppID_Premiere`, then set the `PF_PUI_INVISIBLE` flag on the parameter's `ui_flags`. It's also recommended to set `PF_PUI_DISABLED` to avoid conflicts. Use bitwise OR to set flags (`paramsDef[paramIndex].ui_flags |= PF_PUI_INVISIBLE`) and bitwise AND with negation to unset them (`paramsDef[paramIndex].ui_flags &= ~PF_PUI_INVISIBLE`). Call `PF_UpdateParamUI` with the modified parameter definition. Ensure that no error code is set before calling this function, as it will only execute if `err` is 0.

```cpp
if (!err && in_data->appl_id == kAppID_Premiere)
{
    paramsDef[paramIndex].ui_flags |= PF_PUI_INVISIBLE;
    paramsDef[paramIndex].ui_flags |= PF_PUI_DISABLED;
    ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref,
        paramIndex,
        &paramsDef[paramIndex]));
}
```

*Tags: `params`, `premiere`, `ui`*

---

### Is there an open-source example of hide/show functionality for After Effects plugins?

Alex Bizeau from Maxon mentioned the 36pix wrapper they created years ago as a working reference implementation for hide/show functionality in AE plugins. They also noted there is a special trick required for Premiere Pro but that hide/show is feasible to implement. The codebase is accessible to authorized developers.

*Tags: `open-source`, `premiere`, `reference`, `ui`*

---

### What is the most logical way to prevent an After Effects plugin from being detected in Premiere Pro?

One approach is to return an error in the global setup function, since Premiere Pro runs global setup for all plugins during application startup. This may cause the plugin to be hidden from the effects list, though this behavior was noted as not always working reliably.

*Tags: `aegp`, `deployment`, `pipl`, `premiere`*

---

### How can you prevent the 'need to reinstall plugin' message when After Effects updates without hiding the plugin from the UI?

James Whiffin suggests installing plugins into mediacore to avoid yearly reinstall requirements when AE versions up. However, Tobias Fleischer notes that using PF_OutFlag_I_AM_OBSOLETE will hide the plugin in AE itself. To use this flag only for Premiere Pro while keeping it visible in After Effects, you should add a host name check before applying the flag.

*Tags: `cross-platform`, `deployment`, `pipl`, `premiere`*

---

### What happened to Transcriptive and how did Premiere's native transcription feature impact third-party plugins?

Transcriptive by Digital Anarchy was a popular plugin for years but Adobe introduced native transcription features in Premiere, making third-party solutions less necessary. Transcriptive's web services are ending in May 2026. Reference: https://digitalanarchy.com/blog/video-editing-plugins/transcriptive-end-of-life-web-services-will-be-ending-in-may-2026/

*Tags: `deployment`, `premiere`, `reference`*

---

### How can you trigger parameter updates in Premiere Pro from a CEP script when PF_userChangedParam doesn't work?

In After Effects, using a hidden parameter updated by a script calling PF_userChangedParam works well for triggering updates. However, Premiere Pro does not consider script interactions as user actions, so this approach is not reactive. Alternative solutions include: monitoring external file dependencies for changes (though this is less reactive), or setting up a button to force manual refresh of the plugin parameters.

*Tags: `cep`, `params`, `premiere`, `scripting`*

---

### Why were dropdown menus in a transition plugin malfunctioning during debugging?

The issue occurred when opening a Premiere project that already had the transition plugin applied. Opening a fresh instance of the plugin and applying it newly resolved the dropdown malfunction. The problem was related to the state of the plugin when loaded from an existing project versus a fresh application.

*Tags: `debugging`, `params`, `premiere`, `ui`*

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

### How do you handle colorspace differences between After Effects and Premiere Pro plugins?

After Effects uses ARGB colorspace while Premiere Pro uses BGRA_8u. For 8-bit BGRA, Premiere has a special suite available in the AE SDK (see the sdknoise example which demonstrates BGRA and YUVA colorspace support). For 32-bit float, you need to write your own conversion function. You can choose to support only specific colorspaces in global setup rather than all modes Premiere supports. Basic rendering in 8/16-bit works without optimization, though you won't get the 32-bit icon indicator.

*Tags: `argb`, `bgra`, `colorspace`, `premiere`, `render-loop`*

---

### Can you use the Iterate8Suite in Premiere Pro plugins, and what are the alternatives for multithreading?

The Iterate8Suite has issues in Premiere Pro and doesn't work reliably with multithreading. The suite should work in 8-bit ARGB colorspace (SuiteV2 since 2022), but for BGRA you must use the special BGRA suite. If you need multithreading, implement your own using standard libraries like std::parallel for cross-platform compatibility.

*Tags: `aegp`, `multithreading`, `premiere`, `threading`*

---

### Where can you find example code for handling Premiere Pro BGRA and YUVA colorspaces?

The sdknoise example in the After Effects SDK demonstrates how to properly handle BGRA and YUVA colorspaces for Premiere Pro plugins. This example shows the correct approach for colorspace handling without relying on problematic iterate suites.

*Tags: `bgra`, `colorspace`, `open-source`, `premiere`, `reference`, `yuva`*

---

### How can you specify a custom output region in Premiere transitions when smartFX is not available?

Premiere does not support smartFX, which means you cannot use the smart render feature to request only a specific region of the buffer. When making transitions, Premiere will provide frames at the sequence resolution regardless of the actual footage size. This is a limitation of Premiere's plugin architecture compared to After Effects, where smartFX allows you to define output rectangles and optimize rendering to only the needed buffer region.

*Tags: `output-rect`, `premiere`, `render-loop`, `smartfx`*

---

### How can you iterate through pixel data when working with rowbytes in After Effects or Premiere?

You can iterate through pixel data yourself using rowbytes. Be careful with negative values in Premiere, as they can occur and need special handling.

*Tags: `caching`, `memory`, `mfr`, `premiere`*

---

### What changes are coming to the Premiere SDK regarding capture and recording tools?

According to the SDK roadmap, Premiere will be removing tools for capture recording functionality in the future, signaling the deprecation of tape-based workflows.

*Tags: `deployment`, `premiere`, `sdk`*

---

### How do you debug a Premiere plugin from Xcode when symbols aren't loading and breakpoints don't work?

Ensure that symbol generation is enabled in your build settings. Set 'Generate Symbols' to 'Yes' in your Xcode scheme. If this setting is accidentally set to 'No', symbols won't be generated and debuggers won't be able to load them, preventing breakpoints from working properly.

*Tags: `build`, `debugging`, `premiere`, `xcode`*

---

### Does PF_ABORT() work the same way in Premiere as it does in After Effects?

Unlike in After Effects where PF_ABORT(in_data) can interrupt rendering and set err to PF_Interrupt_CANCEL, this mechanism does not work reliably in Premiere. Even when frames take significant time to render (over half a second), calling PF_ABORT() does not trigger an abort. In Premiere, you may need to finish rendering every frame that is requested rather than relying on abort functionality to interrupt the render process.

*Tags: `aegp`, `cross-platform`, `premiere`, `render-loop`*

---

### Why does Premiere pass NULL for input_world in Render when compiled with fast optimizations, but not in After Effects?

A developer reported that when compiling a plugin with fast optimizations enabled, Premiere passes input_world = NULL in the Render call, whereas without optimizations the data is present. The same code does not exhibit this problem in After Effects. This suggests the issue may be related to optimization levels affecting how Premiere handles or passes buffer data to plugins during rendering.

*Tags: `build`, `debugging`, `premiere`, `render-loop`, `smartfx`*

---

### How should ARGB_8u be defined in globalSetup for proper colorspace handling?

When using ARGB_8u colorspace, it needs to be properly defined in the globalSetup function. The user inquired about the correct definition method and suggested checking colorspace settings and adding null checks for inoutworld (if inoutworld is null return err) to see if Premiere will redo the render.

*Tags: `debugging`, `params`, `premiere`, `render-loop`*

---

### What is the issue with the no_params_vary flag and how does it differ between After Effects and Premiere?

The no_params_vary flag was identified as the origin of a problem in plugin behavior. In After Effects, this flag's behavior can be replaced using MIX_GUI during the pre_render thread. However, in Premiere, the flag does nothing, indicating different parameter variation handling between the two applications.

*Tags: `mfr`, `params`, `premiere`, `render-loop`*

---

### Why does my plugin report 'Not able to acquire AEFX Suite' when playing in Premiere?

This error typically occurs during playback/render in Premiere and may be caused by calling AEFX Suite functions in a shared thread or in code paths that execute in Premiere context. Check if you have conditional logic like `if (app_id != 'PrMr') => call AEFX_suite` where the app_id detection may be returning an incorrect value in Premiere, causing AEFX Suite calls to execute when they shouldn't. Additionally, Premiere can have quirks with certain suites (similar to known colorspace suite issues), so verify your host application detection logic and ensure AEFX Suite calls are not made from Premiere contexts.

*Tags: `aegp`, `debugging`, `premiere`, `threading`*

---

### How long will the CEP engine remain available in Premiere Pro before it's deprecated?

The conversation mentions that UXP for Premiere Pro is now in public beta and encourages CEP panel developers to migrate, but no specific timeline for CEP deprecation was provided in the response.

*Tags: `cep`, `cross-platform`, `deployment`, `premiere`, `uxp`*

---

### What is the current status of UXP support for Premiere Pro?

UXP for Premiere Pro is now available in public beta. Developers with existing CEP panels are encouraged to migrate to UXP and provide feedback to the Premiere Pro team about their migration needs. More information is available at: https://forums.creativeclouddeveloper.com/t/uxp-now-available-in-premiere-pro-beta/8795

*Tags: `deployment`, `premiere`, `resource`, `ui`, `uxp`*

---

### What is the best documentation to start learning UXP for Premiere Pro?

UXP in Premiere Pro is currently very limited but will change soon. There is a comprehensive article that contains all the relevant links to documentation, forums, and other resources to get started with UXP development for Premiere Pro.

*Tags: `documentation`, `premiere`, `reference`, `ui`, `uxp`*

---

### Can you integrate a C++ backend with UXP for Premiere Pro, or do you need to use a separate service with IPC for computational tasks?

There is a new hybrid UXP approach that combines HTML and C++ functionality, though details on how it works are still emerging. This hybrid approach is currently in beta and promises to allow C++ backend integration rather than requiring a separate IPC service for computational tasks.

*Tags: `c++`, `premiere`, `ui`, `uxp`*

---

### What is Bolt UXP?

Bolt UXP is a framework or resource for UXP development in Premiere Pro. It was shared as a relevant tool for developers working with UXP plugins.

*Tags: `premiere`, `reference`, `tool`, `ui`, `uxp`*

---

### Can you change the name of an effect parameter after initial setup?

Yes, you can change a parameter's name by getting its streamRef and changing the stream's name. However, this change will not persist between sessions—when you load a project, the default names will be used again. You can work around this by renaming parameters again during the sequence_resetup event. Note that this approach has been reported to cause issues in Premiere Pro.

*Tags: `params`, `pipl`, `premiere`*

---

### If SmartFX is implemented in a 32-bit After Effects plugin, do I still need to implement the Render() event?

If SmartFX is implemented, After Effects does not send RENDER or FRAME_SETUP calls anymore; instead it sends Smart Render and Pre-render calls. However, if you need Premiere Pro compatibility, you should keep both mechanisms implemented since Premiere Pro does not use SmartFX and still calls Render and Frame Setup events.

*Tags: `aegp`, `premiere`, `render-loop`, `smartfx`*

---

### How can I restrict an After Effects plugin to specific app versions while keeping it in a single installation folder?

Installing a plugin in the Adobe/Common/Plug-ins/7.0/MediaCore folder makes it available in all versions of After Effects and Premiere Pro. Unfortunately, there is no way to restrict it to specific app versions or applications from the MediaCore folder. To restrict a plugin to only specific versions (e.g., After Effects CC 2014-2019), you must install it in each application version's individual plug-ins folder instead of using the shared MediaCore location.

*Tags: `after effects`, `deployment`, `installation`, `plugin`, `premiere`*

---

### Can I disable an After Effects plugin when it loads in Premiere Pro by checking the application ID?

Previously, it was possible to disable a plugin in Premiere Pro by checking in_data->appl_id for 'PrMr' and returning an error code (-1 or similar) at the beginning of the entrypoint function. However, this method no longer works with Premiere Pro CC2018 and later versions. The recommended approach is to install the plugin in version-specific folders rather than relying on application ID checking.

*Tags: `aegp`, `after effects`, `deployment`, `plugin`, `premiere`*

---

### How can you adjust the level of motion blur when using transform_world in a plugin?

According to shachar carmi, transform_world spreads 16 instances across provided matrices to create motion blur. The level of motion blur can be adjusted in three ways: (1) by controlling the length of the blur (shutter angle), (2) by changing the number of samples in the blur, or (3) by adjusting the mix between the motion-blurred result and non-motion-blurred render. Providing more matrices gives a more refined look with better rotations and curved paths, though it doesn't increase the total number of samples. With 2 matrices you get 16 linear positions between them; with 3 matrices you get 8 samples from mat1 to mat2 and another 8 from mat2 to mat3.

*Tags: `motion-blur`, `params`, `premiere`, `smartfx`, `transform_world`*

---

### Can an After Effects plugin be used as-is in Premiere Pro?

Yes, an AE plugin can be used in Premiere Pro if it meets two criteria: (1) it supports the old 'render' and 'frame setup' calls, not only SmartFX 'smart render' and 'pre-render' calls (since Premiere doesn't support SmartFX), though a plugin can support both in parallel; (2) it doesn't use any AEGP suites, which Premiere doesn't support. Additionally, Premiere Pro requires support for BGRA pixel format in addition to AE's ARGB format.

*Tags: `aegp`, `cross-platform`, `premiere`, `smartfx`*

---

### What pixel formats does Premiere Pro require for plugin compatibility?

Premiere Pro requires support for BGRA pixel format in addition to After Effects' ARGB format for cross-application plugin compatibility.

*Tags: `cross-platform`, `premiere`*

---

### How can I set default values for PF_Param_POINT parameters based on sequence resolution in After Effects plugins?

Point parameters in After Effects use percentage-based coordinates for their defaults, so a default value of {50, 50} will always represent the center of the composition or sequence, regardless of resolution. This means you don't need to dynamically set defaults based on sequence information during PF_Cmd_USER_CHANGED_PARAM or other callbacks—the percentage-based system handles this automatically.

*Tags: `ae transition extensions`, `params`, `pf_paramdef`, `premiere`*

---

### My plugin produces completely black output in Premiere Pro when border_expansion is non-zero. What's happening?

Premiere Pro does **not** support `PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS` or negative output rect coordinates. This is an AE-only feature.

When `SmartPreRender` expands `result_rect` into negative coordinates (e.g., `(-65,-65,3905,2225)`), Premiere silently refuses to call `SmartRender` — it returns black/empty output without any error. The debug log will show PreRender being called repeatedly but SmartRender never firing.

**Fix**: Guard border expansion with a host check in `SmartPreRender`:

```cpp
PF_Cmd_SMART_PRE_RENDER:
    auto expansion = compute_expansion();
    if (in_data->appl_id == 'PrMr')
        expansion = 0;   // Premiere doesn't support border expansion

    if (expansion > 0) {
        extra->output->result_rect.left   -= expansion;
        extra->output->result_rect.right  += expansion;
        extra->output->result_rect.top    -= expansion;
        extra->output->result_rect.bottom += expansion;
        extra->output->flags |= PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS;
    }
```

Premiere users see the effect stop at frame edges, but the core algorithm still runs correctly within the frame. For border expansion effects (edge glow, blur with extend), this is an unavoidable Premiere limitation.

**Key Premiere vs AE SmartFX feature differences:**

| Feature | After Effects | Premiere Pro |
|---|:---:|:---:|
| Border expansion / negative rects | Yes | **No** |
| `PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS` | Yes | **No** |
| `output_origin_x/y` | Yes | **No** (always 0,0) |
| VUYA_4444_32f pixel format | No (ARGB) | Yes |
| 8/16/32-bit depths | Yes | 32-bit float only |

Detect Premiere via `in_data->appl_id == 'PrMr'` in any command handler.

*Tags: `premiere`, `border-expansion`, `smart-pre-render`, `black-output`, `result-rect`, `smartfx`*

---
