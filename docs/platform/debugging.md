# Debugging

> 332 Q&As · source: AE plugin dev community Discord

### What's a good approach for logging in AE plugins?

Use spdlog library which supports file rotation (configurable file size and number of files to keep). For performance-sensitive plugins, use conditional logging: check for the existence of a specific file (e.g., 'plugin_log.txt') during GlobalSetup, and only enable logging when that file is present. This lets users enable logging on demand for debugging.

*Tags: `debugging`, `logging`, `performance`, `spdlog`*

---

### What causes uninitialized PF_Err variables and subtle bugs?

A common C++ syntax error: 'PF_Err err, err2 = PF_Err_NONE;' only initializes err2, leaving err undefined (often ends up as 1). The correct form is 'PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;'. This can cause if(!err) checks to fail unexpectedly and lead to hard-to-track bugs like AEGP_GetNewEffectStreamByIndex returning 0x0.

```cpp
// WRONG - err is uninitialized:
PF_Err err, err2 = PF_Err_NONE;

// CORRECT - both initialized:
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `c-plus-plus`, `common-mistake`, `debugging`, `initialization`, `pf-err`*

---

### Why can't macOS Instruments detect PF_EffectWorld memory leaks?

Instruments won't report PF_EffectWorld leaks because AE legitimately thinks you're holding important data in those worlds. AE allocates the worlds deep in its engine, not directly in your plugin code. While memory allocated with new/malloc within the plugin gets reported accurately (because Instruments has more context), AE-managed allocations are invisible to the leak detector. Creating C++ RAII wrappers around AE SDK objects for automatic memory management is recommended.

*Tags: `debugging`, `effect-world`, `instruments`, `mac`, `memory-leak`, `raii`*

---

### How do you disable the 'previous crash warning' popup in AE?

Press CMD+F12 (or Ctrl+Shift+F12 on Windows) to open the debug console, then use the hamburger menu next to the console and change to 'Debug Database'. Search for 'ShowPreviousCrashWarning' and turn it off.

*Tags: `ae-preferences`, `crash-warning`, `debug-database`, `debugging`*

---

### How do you run an older version of Xcode on a newer macOS?

Download older Xcode versions from https://developer.apple.com/download/all/. Run the binary directly from Terminal: '/Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode'. The .app bundle won't open normally on newer macOS, but running the binary directly bypasses this restriction. For quick access, add the /Contents/MacOS folder to Finder sidebar.

*Tags: `debugging`, `development-environment`, `mac`, `xcode`*

---

### What causes slow AE loading when debugging from Xcode 15?

After upgrading to macOS 14/Xcode 15, AE loading becomes ~3x slower due to 'dlsym cannot find symbol xSDKExport' messages for MediaCore bundles. This is Xcode trying to load symbols for all loaded bundles. Workarounds: (1) Run Xcode 14 binary directly from Terminal. (2) Open AE standalone first, then attach the debugger after AE is fully loaded. (3) Check if 'load symbols' option is accidentally selected in your IDE settings.

*Tags: `debugging`, `macos`, `performance`, `symbol-loading`, `xcode`*

---

### How do you use RenderDoc to debug GPU plugins in After Effects?

Use RenderDoc's in-application API with process injection. Steps: (1) Enable 'Enable process injection' in RenderDoc's Tools > Settings. (2) Compile your plugin using RenderDoc's API - in GlobalSetup, check for renderdoc.dll with GetModuleHandleA and get the API pointer. (3) Launch AE first, then inject RenderDoc via File > Inject into Process. (4) RenderDoc must be injected before applying your plugin. (5) Use rdoc_api in your render function to manually begin/end frame capture.

```cpp
// In PF_Cmd_GLOBAL_SETUP:
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll")) {
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
// Then use rdoc_api->StartFrameCapture() / EndFrameCapture() in render
```

*Tags: `debugging`, `gpu`, `opengl`, `process-injection`, `renderdoc`*

---

### How do you debug the 'plugin wasn't installed correctly' error on Windows?

Use dumpbin (included with Visual C++) to check plugin dependencies: 'dumpbin /dependents plugin.aex'. Common causes: (1) Missing Visual C++ Redistributable packages (download from https://aka.ms/vs/17/release/vc_redist.x64.exe). (2) Missing DLL dependencies. (3) Error in GlobalSetup. (4) Debug builds accidentally linking to debug CRT DLLs (msvcp140d.dll). Try setting a breakpoint on GlobalSetup in debug mode to see if the entry point is reached.

```cpp
dumpbin /dependents plugin.aex
```

*Tags: `debugging`, `dependencies`, `dumpbin`, `installation`, `vcredist`, `windows`*

---

### What causes AE error code 1397908844?

Error 1397908844 is a generic exception handler/catch-all in Adobe's suites manager, dating back to CS2/CS3. It commonly appears when double-freeing/releasing a suite pointer. It appears more often in newer AE versions due to MFR with global suite handles shared between threads. Known triggers include: problems with arb params, crashes after exporting RAM previews, and double-releasing suites. The underlying issue is usually in the plugin's suite management code.

*Tags: `debugging`, `double-free`, `error-code`, `mfr`, `suite-manager`*

---

### How do you debug AE plugins built with Rust on macOS?

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

*Tags: `debugging`, `lldb`, `macos`, `rust`, `vscode`*

---

### Why might Adobe report high numbers of After Effects force-quits during launch?

A significant portion of force-quit reports during AE launch may actually be caused by plugin developers stopping AE from their debuggers (e.g., hitting 'Stop Debugging' in Visual Studio or Xcode while AE is attached). This registers as a force quit in Adobe's telemetry but is not an actual crash or bug.

*Tags: `crash-reports`, `debugging`, `force-quit`, `telemetry`, `visual-studio`*

---

### How can I debug a plugin when it's rendered through AE Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible/separate AE instance for rendering. You can see these processes in the task manager. To debug, attach your debugger to the already running AE process spawned by AME. In Visual Studio, use Debug > Attach to Process. In Xcode, use Debug > Attach to Process by PID or Name.

*Tags: `ame`, `attach-debugger`, `debugging`, `media-encoder`, `render-queue`*

---

### Is it possible to hot-reload AE plugins during development without restarting AE?

Hot reload (called 'Edit and Continue' in Visual Studio) was common on 32-bit builds but became less reliable on 64-bit. Even with hot reloading, there are limitations: you can't change ParamSetup because AE calls it only once per session when the plugin is first loaded. Starting AE via the debugger (Visual Studio or Xcode) tends to launch faster than normal startup. Each time you rebuild C++ code, you generally need to restart AE, unlike ExtendScript or CEP where reloads are possible without full restart.

*Tags: `debugging`, `development-workflow`, `hot-reload`, `visual-studio`, `xcode`*

---

### After upgrading a plugin from CS5 SDK to the 2021 SDK with MFR support, AE crashes when saving the project. How to debug?

The crash is likely related to sequence data flattening. Check if PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set in global setup - if so, you must handle PF_Cmd_SEQUENCE_FLATTEN properly. When modifying sequence data, be careful about whether you're changing data in the same handle or allocating a new one. Debug steps: (1) Purge Xcode intermediates and clean rebuild. (2) Test SDK sample projects to verify they work. (3) Comment out parts of your plugin to isolate the issue. (4) Copy your code onto a fresh sample project.

*Tags: `crash`, `debugging`, `mfr`, `save`, `sdk-upgrade`, `sequence-flatten`*

---

### How do you debug/log variables from a C++ AE plugin on Windows?

Use OutputDebugString to send debug output that can be viewed with tools like DebugView or Visual Studio's Output window.

```cpp
std::string log1 = "variable: " + std::to_string(variable) + "\n"; OutputDebugString(log1.c_str());
```

*Tags: `cpp`, `debugging`, `logging`, `outputdebugstring`, `windows`*

---

### How do you view the current pixel bit depth in Premiere Pro?

Use the DogEars feature. You can look up the keyboard shortcut online, or edit the debug database to enable it. This feature is only available in Premiere Pro, not in After Effects.

*Tags: `bit-depth`, `debugging`, `dogears`, `premiere-pro`*

---

### What are the main challenges with GPU plugins and MFR compatibility?

GPU plugins have significant stability issues with MFR compared to CPU plugins. The issues are hard to track down because crashes occur only after extended rendering periods. OpenGL-based plugins experience particular difficulties, especially on macOS where platform-specific issues arise.

*Tags: `debugging`, `gpu`, `mfr`, `opengl`*

---

### If a resource fails to allocate, does it need to be disposed of?

No, if a resource fails to allocate, it does not need to be disposed of since it was never successfully created in the first place.

*Tags: `debugging`, `memory`*

---

### What happens if you try to dispose of a world that failed to allocate due to insufficient memory?

If allocation fails, do not attempt to dispose of the world. Attempting to dispose of a world you didn't successfully allocate will trigger an error warning stating you should not dispose of a world you didn't allocate. This error appears on top of the original out of memory warning, creating a confusing user experience.

*Tags: `aegp`, `debugging`, `memory`*

---

### Why is the plugin experiencing glitch behavior with MFR enabled when the Multithreaded flag is not set?

Different threads/CPUs are using different previous frames when MFR (Multi-Frame Rendering) is enabled, causing the glitch behavior. The issue resolves when MFR is turned off. The root cause appears to be that MFR is being invoked even though the Multithreaded flag is not explicitly set for the plugin.

*Tags: `debugging`, `mfr`, `threading`*

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

### At what point does the Missing Entry Point error occur when loading a plugin?

The Missing Entry Point error occurs when the effect is applied, not during After Effects loading. This timing distinction helps differentiate between entry point/PiPL issues versus runtime initialization problems.

*Tags: `debugging`, `pipl`*

---

### What causes an error when working with arbitrary parameters?

The error occurs if you don't give the arbitrary (arb) parameter a name.

*Tags: `arb-data`, `debugging`, `params`*

---

### How can you make After Effects re-render a frame multiple times until a calculation is complete?

You need to call render() several times in a loop until your calculation is done. This requires a flag that tells AE to re-render the frame again, and inside the render function you check with an if clause whether the render is actually finished or not.

*Tags: `debugging`, `render-loop`, `smart-render`*

---

### What causes an infinite progress bar at the end of a caching effect?

An infinite progress bar look can be triggered by an incoherent PF_Progress value. Give it impossible values like 200/100 and it should display this infinite progress behavior.

*Tags: `caching`, `debugging`, `ui`*

---

### Does changing image size or color depth affect the frame number where a cache-related crash occurs?

The suggestion is to test whether changing image size or color depth of the project changes the crash frame number, as this could indicate whether RAM access is being handled correctly with time-varying data, or if there's a conflict between sequence data writing and MFR.

*Tags: `caching`, `debugging`, `memory`, `mfr`, `sequence-data`*

---

### Is it illegal to checkout a parameter multiple times?

You cannot checkout parameters during sequence setup, resetup (when AE is loading the project), or sequence setdown. The world checkout can sometimes be empty in these contexts. Additionally, you must handle checkout errors and check in the parameter immediately after using it in loops.

*Tags: `aegp`, `debugging`, `layer-checkout`, `params`*

---

### What causes a plugin to fail loading during After Effects startup with a 'bad filter' error when the entry point cannot be found?

This type of error typically indicates a dependency issue rather than a driver or OS problem. The error occurs before the Vulkan instance initialization (as evidenced by no Vulkan errors in logs), suggesting the problem happens during plugin loading or initialization of global data. Common causes include missing or incompatible dependencies, incorrect plugin structure, or issues in the main entry point implementation.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What can cause main entry point errors in After Effects plugins?

Main entry point errors can happen when a dll is missing from the system. Additionally, the libraries/dlls need to have the right architecture (32-bit vs 64-bit) to match the plugin and After Effects installation.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### How can you verify that all required dependencies are present in a Windows plugin build?

Use Dependency Walker on Windows to check for missing shared libraries and ensure all external dependencies are properly resolved.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What causes VK_ERROR_LAYER_NOT_PRESENT and VK_ERROR_EXTENSION_NOT_PRESENT errors when creating Vulkan instances with embedded MoltenVK in a plugin?

These errors typically indicate that the MoltenVK ICD is not being located correctly by the Vulkan loader, or there are environment configuration issues. When MoltenVK is statically embedded rather than installed system-wide, the Vulkan loader may not find the necessary validation layers and extensions. This is particularly challenging in sandboxed or non-standard installations where the LunarG Vulkan SDK is not in the default /usr/local path. The solution involves ensuring the Vulkan loader can properly locate the MoltenVK ICD configuration.

*Tags: `debugging`, `deployment`, `macos`, `vulkan`*

---

### How do you set up MoltenVK compatibility with Vulkan 1.2 on macOS for After Effects plugins?

Get at least the October update to get a MoltenVK compatible with Vulkan 1.2. For debug builds, create a target using ICD in environment with dylib linking, add "VK_KHR_portability_subset" to instance extensions (debug only for now until MoltenVK uses it), and include debugging layers. For release builds, link to frameworks only as static libraries to work within Vulkan 1.2 limitations.

*Tags: `apple-silicon`, `build`, `debugging`, `macos`, `vulkan`*

---

### How to fix OpenGL loader initialization failure on M1 Mac with ImGui?

The issue is likely related to M1 chip compatibility. ImGui can run over Metal instead of OpenGL, which would require conversion from OpenGL matrices but would be the proper approach for Apple Silicon. The error may be similar to previous issues with GLSL version differences between Mac and Windows, but specific to ARM64 architecture on M1.

*Tags: `apple-silicon`, `debugging`, `macos`, `metal`, `opengl`*

---

### How can you obtain a drawing reference for rendering without user interaction events in After Effects plugins?

The drawing reference is typically only available through PF_Cmd_EVENT when user interaction occurs. To draw during rendering without user interaction, you need an alternative approach. One solution is to store the drawing reference when it becomes available (during any event where PF_GetDrawingReference is called), and then reuse that stored reference in your render loop. However, be cautious about thread safety and the validity of the reference across different render contexts. Another approach is to use AEGP suites to directly access composition or layer drawing capabilities if available, rather than relying on the Effect Custom UI Suite's event-driven drawing reference.

```cpp
// Store drawing ref when available from PF_Cmd_EVENT
PF_DrawingReference stored_drawing_ref = NULL;

// In PF_Cmd_EVENT handler:
if (event_extraP) {
  suite.EffectCustomUISuite1()->PF_GetDrawingReference(
    event_extraP->contextH, 
    &stored_drawing_ref
  );
}

// Later, attempt to use in render loop (verify validity first)
if (stored_drawing_ref) {
  // Use stored_drawing_ref
}
```

*Tags: `custom-ui`, `debugging`, `render-loop`, `ui`*

---

### How can you get a drawing reference from within the render loop without relying on user interaction events?

The drawing reference is typically obtained through PF_GetDrawingReference in the PF_Cmd_EVENT handler via the event_extraP context, which is only available when the user interacts with the effect. Getting a drawing reference from within the render loop without user interaction is not straightforward and requires alternative approaches, possibly involving undocumented APIs or workarounds that experienced developers like Shachar Raindel might know about.

```cpp
suites.EffectCustomUISuite1()->PF_GetDrawingReference(event_extraP->contextH, &drawing_ref);
```

*Tags: `aegp`, `debugging`, `render-loop`, `ui`*

---

### How do you print to the console when debugging an After Effects plugin in Visual Studio?

On Mac, use printf. On Windows, use the OutputDebugStringA macro. You can also use AEGP_WriteToOSConsole from the utility suite, though this may not work in Premiere. OutputDebugStringA is the recommended approach for Windows console output during debugging.

```cpp
OutputDebugStringA("debug message");
```

*Tags: `debugging`, `macos`, `windows`*

---

### What causes PF_Interrupt_CANCEL to occur in the middle of a render in After Effects?

PF_Interrupt_CANCEL can occur during rendering when certain conditions are met, such as when CAPS LOCK is off and the composition has not been previously rendered to the RAM cache. The issue appears to be related to some interaction with keyboard state or cache status, as the interrupt does not occur when CAPS LOCK is on or when the composition has been fully previewed in the RAM cache beforehand.

*Tags: `caching`, `debugging`, `memory`, `render-loop`*

---

### How can C++ check if a CEP panel is opened?

C++ can check if a CEP is opened using the TCP protocol or by using a hidden checkbox that the CEP activates via scripting.

*Tags: `cep`, `debugging`, `scripting`*

---

### Why do frames only invalidate after manually triggering a change like modifying the comp background color?

The issue appears to be that After Effects does not check whether frames need invalidating frequently enough. Even though the boolean value updates properly (confirmed with breakpoints) and frames should invalidate, they only do so after giving AE a 'kick' by changing the composition background color or changing a parameter and changing it back.

*Tags: `debugging`, `render-loop`, `smart-render`*

---

### Can a deployment target mismatch in Xcode cause an 'Invalid Filter' error on macOS?

Yes, setting a deployment target higher than the user's OS version will cause that error. The user had deployment target 12.3 in Xcode but the user was on Big Sur 11.17.9, which is lower than the deployment target.

*Tags: `debugging`, `deployment`, `macos`*

---

### How can you debug plugin crashes when changing the macOS deployment target doesn't resolve the issue?

Add logging at function entry and exit points to determine where the crash occurs (global setup, param setup, etc). You can use conditional logging by checking for a log file's presence in global setup, so users only enable logging when needed. For production plugins, consider profiling performance impact of logging and use a library like spdlog with file rotation to manage log file sizes.

*Tags: `debugging`, `macos`, `pipl`*

---

### How should logging be implemented in plugins without slowing down render performance?

For real-time or fast-rendering plugins, logging can impact performance. Implement conditional logging by checking for a marker file (e.g., 'pixel_sorter_log.txt') in global setup, enabling logging only when the file exists. This allows users to request logging without it affecting normal operation. Use spdlog library which includes file rotation capabilities to manage log file sizes and count.

*Tags: `debugging`, `performance`, `pipl`*

---

### What should be cleaned when experiencing build issues in Xcode?

Use Cmd+shift+k to clean the build folder, which cleans everything there is to clean. You can also reset After Effects preferences if needed.

*Tags: `build`, `debugging`, `macos`*

---

### What causes an 'entry point not found' error on Windows after a plugin is loaded?

Two main possibilities: a missing dependency (such as a Visual C++ redistributable package) or an error in global setup. The issue may be resolved by installing the Visual C++ redistributable package. In debug mode, you can set a breakpoint on global setup to verify if it's being called.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### Why is there a memory leak when using AEGP_memorysuite on Mac but not Windows during export?

The memory leak appears to be platform-specific (Mac only) and related to how AEGP_memorysuite handles memory operations. The issue is less severe with the mem_QUIET flag. Using standard C++ memory allocation (new/delete) instead of AEGP_memorysuite resolves the leak, and replacing memcpy with strncpy partially solves the problem (from mem pointer to buffer works, but not in the reverse direction). This suggests the issue may be related to how memcpy interacts with AEGP's memory management on macOS, particularly when memory is freed from a different thread than where it was allocated.

*Tags: `aegp`, `debugging`, `macos`, `memory`, `threading`*

---

### What is a recommended approach for tracking memory allocations and deallocations in C++ plugins?

Wrap memory operations in custom optionally-logging wrappers to get a definitive record of what happened. Additionally, have all C++ classes descend from a base class like ObjectBase that keeps track of live instances of each type, allowing you to report current counts for allocations versus frees.

*Tags: `caching`, `debugging`, `memory`*

---

### Is there a limitation with AEGP_memsuite when allocating and freeing memory during the render thread?

The user observed that memory allocated using aegp_memHandle during the compute cache thread is not being freed when the freemem function is called, even though instruments show no leaks. This suggests there may be specific limitations or requirements for using AEGP_memsuite during the render thread that differ from other threads.

*Tags: `aegp`, `compute-cache`, `debugging`, `memory`, `render-loop`, `threading`*

---

### Why is layer data NULL after checkout_layer_pixels when a layer is selected in the layer selection parameter?

This issue typically occurs when the layer parameter is not properly initialized or when the layer selection is invalid. Ensure that PF_ADD_LAYER uses appropriate default values and that the layer index being checked out actually contains valid data. The issue may also resolve itself if the layer parameter setup or plugin caching is refreshed. Verify that the layer being checked out is not disabled, hidden, or has zero dimensions, as these conditions can result in NULL data pointers.

```cpp
// Verify layer parameter setup
AEFX_CLR_STRUCT(def);
PF_ADD_LAYER(LAYER_STR, PF_LayerDefault_NONE, GPU_SKELETON_LAYER_Disk_ID);

// In SmartRender, add null checks before accessing data
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `debugging`, `layer-checkout`, `params`, `smart-render`*

---

### What is the best approach to restrict two float sliders so that Start cannot be equal to or greater than End?

Use the UserChangedParams callback to check which slider changed, then validate and update both parameter values accordingly. Avoid clamping min/max values dynamically during sliding, as this causes issues with keyframing where previously valid keyframes can become locked and unmutable. Instead, treat both parameters as unordered endpoints and use min() and max() of the two values in your code to determine Start and End.

```cpp
If sliderA has changed
    Check value of sliderA
    Check value of sliderB
    Apply value for both
If sliderB has changed
    Check value of sliderB
    Check value of sliderA
    Apply value for both
```

*Tags: `debugging`, `params`, `ui`*

---

### How can I debug a crash in After Effects on Windows that occurs during project loading when my plugin is installed, but not on Mac?

Use a debugger with breakpoints on EffectMain to trace when the crash occurs relative to plugin initialization. The crash appears to happen after GlobalSetup and ParamsSetup complete successfully (around 85% project load) with no plugin code executing at that point, suggesting a symbol clash or state corruption rather than direct plugin code failure. Try importing the project into a new empty project as a workaround, or preload the conflicting library (Cineware in this case) before opening the project.

*Tags: `crash`, `debugging`, `macos`, `windows`*

---

### Why might a plugin cause a third-party library like Cineware to crash during After Effects project loading even when the plugin code isn't executing at the crash point?

The plugin may be defining a symbol that clashes with the third-party library's loading process, or it may be causing some state corruption in After Effects that affects library initialization later. The fact that preloading Cineware before opening the project allows it to work suggests a symbol collision or initialization order issue rather than a direct code bug.

*Tags: `debugging`, `memory`, `windows`*

---

### Why does changing the match name cause the CUI and parameters to become invisible in the ECW?

This was caused by an underlying C++ syntax error where a PF_Err variable was declared but not initialized, causing AEGP_GetNewEffectStreamByIndex to return 0x0. The real issue was that error checking with if(!err) statements was failing because the uninitialized error variable had a garbage value (usually 1) instead of PF_Err_NONE (0). The solution is to properly initialize all error variables when declaring multiple variables of the same type.

```cpp
// Wrong - err is undefined
PF_Err err, err2 = PF_Err_NONE;

// Correct - both variables initialized
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `aegp`, `debugging`, `params`, `pipl`*

---

### Why doesn't macOS Instruments detect memory leaks from PF_EffectWorlds that are allocated but never disposed?

PF_EffectWorlds are likely allocated deep within the AE engine rather than directly within the plugin code, so memory leak detection tools like Instruments don't recognize them as true leaks since they're allocated at a different level in the AE memory management hierarchy.

*Tags: `aegp`, `debugging`, `macos`, `memory`*

---

### Why doesn't macOS Instruments detect memory leaks from undisposed PF_EffectWorlds even though After Effects runs out of memory?

PF_EffectWorlds allocated in After Effects plugins may not be detected by Instruments leak detection because After Effects itself may be managing the memory lifecycle through its own memory management systems, or the allocations may be attributed to After Effects' internal memory pools rather than the plugin's direct allocations. Instruments can detect direct memory leaks (e.g., data allocated in pre-render that isn't disposed), but large framework-managed allocations like PF_EffectWorlds may be tracked differently and not flagged as leaks even when they cause memory exhaustion.

*Tags: `debugging`, `macos`, `memory`, `plugin-leak-detection`*

---

### How do you disable the annoying popup that appeared in After Effects 2024?

Use CMD+F12 to open the console. Click the hamburger menu next to the console and change to Debug Database. Search for 'ShowPreviousCrashWarning' and turn it off. Alternatively, the setting can be modified in the prefs.txt file.

*Tags: `debugging`, `macos`, `ui`*

---

### What development strategy should be used to test plugin compatibility across macOS versions?

The best approach is to maintain separate development and testing machines: one dev Mac with the target macOS version and a separate test Mac updated to the latest version. This allows developers to confirm that products run on new OS versions and debug any version-specific bugs that only occur on the latest release. However, this is an expensive option.

*Tags: `build`, `debugging`, `deployment`, `macos`*

---

### How do you get Xcode 14 working with After Effects plugin development?

Run the Xcode binary directly from Terminal using the path /Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode (replacing .app with the actual version name). Download Xcode 14.3.1 from https://developer.apple.com/download/all/

*Tags: `build`, `debugging`, `macos`*

---

### What causes the 'dlsym cannot find symbol xSDKExport' error when launching After Effects from Xcode 15?

The error is related to MediaCore taking a long time to load all symbols. This output from Xcode started appearing after upgrading to Xcode 15. It may be related to symbol loading options being enabled, similar to a Visual Studio issue where selecting 'load symbols' option causes very long loading times.

```cpp
dlsym cannot find symbol xSDKExport in CFBundle 0x60000064ae60 </Applications/Adobe After Effects 2024/Adobe After Effects 2024.app/Contents/PlugIns/(MediaCore)/PlayerMediaCore.bundle> (bundle, loaded): dlsym(0xa68b4e61, xSDKExport): symbol not found
```

*Tags: `build`, `debugging`, `macos`*

---

### Is there a 'load symbols' option in the debugger that could be slowing down After Effects launch times?

Yes, there is a 'load symbols' option that can be selected in Visual Studio (and likely other debuggers) which causes very long loading times when After Effects is launched. This option can typically be disabled to speed up the launch process.

*Tags: `build`, `debugging`*

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

### Why might an AEGP not appear in AE and not hit any breakpoints in the entire project?

The questioner did not receive a definitive answer. The conversation moved to other topics without resolution.

*Tags: `aegp`, `debugging`, `macos`*

---

### Were other assignment operators like += working properly in GLSL?

The conversation indicates that /= was the problematic operator in GLSL, with the implication that other assignment operators like += were functioning correctly, though this wasn't explicitly confirmed in the replies.

*Tags: `debugging`, `gpu`, `opengl`*

---

### Why do third-party GPU debuggers crash or fail when used with After Effects?

Third-party GPU debuggers are designed for games with permanent instances, making them incompatible with plugins running inside host applications like After Effects or Premiere. Even modern API debuggers struggle with this architecture. RenderDoc, for example, tends to capture the host application's DirectX instead of the plugin's OpenGL, and gDEBugger crashes after tracing.

*Tags: `debugging`, `gpu`, `opengl`, `premiere`, `vulkan`*

---

### Why does a plugin act weird on Media Encoder and render without the effect applied when queued from After Effects?

The issue is likely related to static global variables in the plugin or elements defined in globalData or during global initialization thread. These can cause problems across different Media Encoder versions.

*Tags: `debugging`, `memory`, `premiere`, `threading`*

---

### How can you implement a key generation function that requires AEGP_HashSuite1 when the function signature only provides AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP parameters?

This question was asked but not answered in the conversation.

*Tags: `aegp`, `compute-cache`, `debugging`*

---

### How can I use RenderDoc to debug GPU code in After Effects without crashing?

Use RenderDoc's In-application API by injecting it into AE after it's already loaded. First, enable process injection in Tools > Settings > "Enable process injection". Launch AE fully, then inject RenderDoc via File > Inject into Process before applying your plugin. In your plugin code, compile with the RenderDoc API by loading renderdoc.dll in PF_Cmd_GLOBAL_SETUP and retrieving the API function pointer. Then use the rdoc_api pointer in your render function to manually begin and end frame captures.

```cpp
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll"))
{
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
```

*Tags: `debugging`, `gpu`, `windows`*

---

### In what scenarios would the nullptr checks for extraP and its members fail?

The checks for extraP, extraP->cb, and extraP->cb->GuidMixInPtr can fail in older versions of After Effects (before PF_AE130_PLUG_IN_VERSION) where PreRenderExtra or the callback structure are not available, or when the GuidMixInPtr function pointer is not implemented or initialized. The code also checks for a sentinel value (0xabababababababab) to detect uninitialized function pointers. These safeguards ensure compatibility across different AE versions and configurations where these features may not be present.

```cpp
if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION && extraP && extraP->cb && extraP->cb->GuidMixInPtr && (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab)
```

*Tags: `cross-platform`, `debugging`, `params`, `pipl`*

---

### What are the scenarios where nullptr checks on extraP and its members fail?

The conversation does not provide a specific answer to this question. The asker acknowledges the issue with 'that sucks' but no detailed explanation of failure scenarios is given.

*Tags: `debugging`, `memory`*

---

### Are there certain suites that shouldn't be called from Death Hook?

Yes, there are restrictions. The Utility Suite should not be called from Death Hook as it can cause unhandled exceptions when trying to use functions like ReportInfo.

*Tags: `aegp`, `debugging`, `threading`*

---

### Are there certain AEGP suites that should not be called from Death Hook, and does using Utility Suite to ReportInfo cause exceptions?

The user reported unhandled exceptions when attempting to call Utility Suite's ReportInfo function from Death Hook. This suggests that certain suites have restrictions on being called from Death Hook contexts, likely due to the cleanup phase not supporting all operations. The answer was not explicitly provided in the conversation, but the question indicates a known limitation.

*Tags: `aegp`, `death-hook`, `debugging`*

---

### What error occurs when trying to adjust effect parameters using AEGP during smartrender?

After Effects gives an error box saying 'error: effect attempting to modify a locked project' when trying to adjust the parameter of an effect using AEGP during a smartrender command.

*Tags: `aegp`, `debugging`, `params`, `smartrender`*

---

### If mask path APIs are available in the JavaScript API, why couldn't they be used in C++ plug-ins by running the script within the plug-in?

The conversation suggests this is theoretically possible - you can run scripts within a plug-in to access JavaScript API functionality. However, this approach has limitations compared to native C++ implementation, particularly regarding per-character path separation which is still being lobbied for in the C++ plugin API.

*Tags: `aegp`, `debugging`, `params`, `scripting`*

---

### What is causing custom UI arb parameters to glitch in After Effects 2025?

The issue is not directly related to arb parameters, but rather to drawing image buffers using the DrawBot suite. The problem occurs when the UI composites multiple images per plugin context. The culprit is likely NewImageFromBuffer, which on every call probably sets the pixels of all retained DRAWBOT_ImageRef objects within the same context, causing flashes or scrambled/duplicated UI. The issue can be avoided if the UI either doesn't composite an image at all or only composites a single image per plugin context.

*Tags: `arb-data`, `debugging`, `ui`*

---

### Is it expected to receive 5000+ calls to arb dispose when shutting down After Effects?

Yes, this is expected behavior. After Effects allocators generally do not dispose of memory until either available memory is saturated, threads are destroyed, or the application exits. You can test this behavior by artificially setting the available memory to a low value like 2GB, which will trigger more destruction calls during runtime.

*Tags: `arb-data`, `debugging`, `memory`*

---

### How can you get the bit depth of a PF_LayerDef without using the world suite?

You can check the world_flags field of the PF_LayerDef. If PF_WorldFlag_DEEP is set, it's 16-bit. If PF_WorldFlag_RESERVED1 is set, it's 32-bit. Otherwise, it defaults to 8-bit.

```cpp
int get_bitdepth(const PF_LayerDef& layer) {
    if (layer.world_flags & PF_WorldFlag_DEEP) {
        return 16;
    } if (layer.world_flags & PF_WorldFlag_RESERVED1) {
        return 32;
    } else {
        return 8;
    }
}
```

*Tags: `debugging`, `params`*

---

### Is it safe to rely on PF_WorldFlag_RESERVED1 to detect 32-bit depth in production plugins?

Yes, it appears safe to trust this method in actual plugins. According to the original code author, this approach was derived from SDK examples, the AE SDK mailing list, or reverse-engineering work done during actual plugin development (specifically a 3D renderer plugin requiring f32 support). The method has proven reliable in production use.

*Tags: `debugging`, `layer-checkout`, `params`*

---

### What is the correct way to call AEGP_StartUndoGroup in After Effects 2025?

In After Effects 2025 beta and later, AEGP_StartUndoGroup must be called with an empty string "" rather than null. Passing null will cause a crash, while passing an empty string works as expected and does not add an entry to the Undo stack.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `debugging`, `macos`, `undo`, `windows`*

---

### Why is the PF_ParamDef* params[] pointer always nullptr when handling PF_Cmd_EVENT?

When handling PF_Cmd_EVENT, you need to perform a switch on extra->e_type. Parameter modification is only possible for specific event types, particularly click and keyboard events. The params pointer may not be available in all event types, and you may need to use AEGP functions instead of directly accessing the params pointer depending on the event type.

```cpp
case PF_Cmd_EVENT:
    err = HandleEvent(in_data, out_data, params);
    // Switch on extra->e_type
    // Check if event is click or keyboard type
```

*Tags: `aegp`, `debugging`, `params`, `ui`*

---

### How can you determine the After Effects version at runtime?

You can use a 'dirty solution' by locating the path of the running binary host using AEGP_GetPluginPaths with AEGP_GetPathTypes_APP as the path type. From this you can extract the AE folder name or version from the binary's metadata. Alternatively, you can try mapping AEGP_GetDriverSpecVersion to the AE version number.

```cpp
AEGP_GetPluginPaths(plugin_id, AEGP_GetPathTypes_APP, &path_outH)
```

*Tags: `aegp`, `debugging`, `deployment`*

---

### Do you need to lock the handle when calling GetConstSequenceData for sequence data?

Yes, you need to lock the handle even when reading only. Use PF_LOCK_HANDLE(const_seq) to lock the handle before accessing it, and unlock it after use. This is necessary in multi-frame render (MFR) scenarios with multiple plugin applications, and the lack of locking can cause nullptr returns and crashes. While Adobe documentation doesn't clearly document this requirement, it appears to be essential for thread safety.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `aegp`, `debugging`, `memory`, `mfr`, `sequence-data`, `threading`*

---

### Does using AEGP_SetStreamValue to update dropdown parameters break the undo history in After Effects?

This is uncertain. The implementation includes a debug warning stating that using AEGP_SetStreamValue causes weird bugs in the undo history, but the actual extent and nature of these bugs is not definitively documented. Users should test this functionality in their specific implementation to understand the undo behavior.

*Tags: `aegp`, `debugging`, `params`*

---

### Do PF_LOCK_HANDLE and host_lock_handle functions actually do anything in After Effects?

According to Adobe engineers, PF_LOCK_HANDLE and host_lock_handle functions are dummy functions that do nothing at all. As stated by Jason Bartell from Adobe in a 2022 discussion: 'Executive summary is handle locking and unlocking has not done anything in After Effects since roughly CS6.'

*Tags: `aegp`, `debugging`, `memory`*

---

### How do you debug a plugin that fails to load in After Effects and appears offline in Premiere?

Check if the error occurs before global_setup is called. If it happens before that point, it may be due to a library dependency that AE loaded in a new version that your plugin uses. Verify whether the issue occurs on both Windows and macOS or only on specific platforms.

*Tags: `debugging`, `deployment`, `macos`, `pipl`, `windows`*

---

### What could cause a simple C++ plugin with no external dependencies to fail loading?

Even simple plugins without explicit dependencies can fail due to library dependencies loaded by After Effects itself in newer versions. The issue may be platform-specific, as it can work on one platform (like Windows) while failing on another (like macOS).

*Tags: `cross-platform`, `debugging`, `deployment`, `macos`, `windows`*

---

### Where is the DebugDatabase.txt file located in the latest After Effects beta release?

The DebugDatabase.txt file is in the wrong location in the latest beta release. The After Effects team is working on fixing this issue.

*Tags: `debugging`, `deployment`*

---

### What is the nature of the issues reported by the Red Giant team with the pre-release version?

The issues are related to code signature validation. Something is getting corrupted somewhere and the validation process is causing the problem. Red Giant is in contact with Adobe and the issue is being investigated, but a full picture is still being determined.

*Tags: `build`, `debugging`, `deployment`*

---

### Why does SmartRender never get called in Release builds even when PreRender returns PF_Err_NONE?

The issue was caused by a mismatch between the SDK code version used for building (2025.2) and the SDK headers (2023). The debug configuration worked fine because it didn't have this version mismatch. Ensure that both the SDK code and headers are from the same version when building.

*Tags: `build`, `debugging`, `release`, `smart-render`, `smartfx`*

---

### How do you get a stacktrace with symbols when using the Rust bindings for After Effects on macOS?

Use VSCode with the CodeLLDB extension and configure a launch.json file to attach the debugger to the After Effects application. Set up a launch configuration that points to the After Effects executable and runs a build task before launching.

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build"
    }
  ]
}
```

*Tags: `debugging`, `lldb`, `macos`, `rust`*

---

### Does the old GLator sample have memory leaks in its GL rendering and texture download functions?

Yes, the GLator sample has major memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, meaning bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBO's.

*Tags: `debugging`, `gpu`, `memory`, `opengl`*

---

### Does PF_Interrupt return a value that triggers the catch block when it's not PF_Err_NONE?

No, PF_Interrupt returns an error code, not an exception. A non-PF_Err_NONE error return does not trigger a catch block in C++.

*Tags: `aegp`, `debugging`*

---

### Why does After Effects crash when launched from lldb with certain launch.json configurations?

The crash was caused by missing the "sourceLanguages": ["rust"] line in the launch.json configuration. Adding this line to the debugger launch configuration prevents the crash from occurring.

```cpp
"sourceLanguages": ["rust"]
```

*Tags: `build`, `debugging`, `macos`, `windows`*

---

### Is there a reliable way to get the current After Effects or Premiere version number (e.g., 2025)?

The in_data->version major and minor fields do not appear to be reliably updated between versions, returning the same values on both 2024 and 2025. This approach may work on Mac AE but needs verification on Premiere and Windows.

*Tags: `debugging`, `deployment`, `macos`, `premiere`, `windows`*

---

### How should error handling be structured when developing After Effects plugins?

Use std::expected for function returns instead of traditional error codes, allowing for proper .then chaining. Wrap all AE handles in classes with proper construction and destruction via smart pointers. Implement an allocation class manager that clears allocations when exiting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug mode, use a cache validator to detect extra memory still allocated at endpoints.

*Tags: `aegp`, `debugging`, `error-handling`, `memory`*

---

### How can you implement custom debug assertions that trigger breakpoints without crashing the program?

Create assertion macros using preprocessor directives that automatically trigger a breakpoint if a debugger is attached, rather than stopping the program. This can be implemented using assembly calls with platform-specific code paths for Windows and macOS. C++26 will have similar built-in features.

*Tags: `cross-platform`, `debugging`, `macos`, `windows`*

---

### How do you debug ExtendScript in After Effects?

On Windows, you can use the standard debugging tools. On macOS, the situation was previously difficult because you needed to use the Intel version which was very slow, but Adobe updated the debugger about a month ago to be Apple Silicon native, which is a significant quality of life improvement.

*Tags: `apple-silicon`, `debugging`, `macos`, `scripting`*

---

### Is the path value converted properly for 16/32 bit processing?

Eric was investigating whether path values are being converted correctly between 8-bit and 16/32-bit processing paths. The issue was occurring during a blur function and was resolved by precomposing the layer with the effect first, suggesting it may be a bit-depth conversion detail.

*Tags: `debugging`, `memory`, `params`*

---

### What were the root causes of bugs in FastBoxBlur?

Two critical bugs were identified: (1) Incorrect bit depth detection causing memory corruption, and (2) Improper stride handling for padded buffers. The solution involved fixing bit depth detection to use rowbytes calculation and implementing proper stride handling throughout the blur algorithm.

*Tags: `debugging`, `memory`, `render-loop`*

---

### How should I debug the 'plugin wasn't installed correctly' error on Windows 11?

Use dumpbin from Visual C++ to check plugin dependencies. Run 'dumpbin /dependents plugin.aex' in a terminal to see what dependencies are listed. The issue may be related to missing Microsoft Visual C++ runtime package that needs to be installed on the user's system.

```cpp
dumpbin /dependents plugin.aex
```

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### How can you check if the After Effects renderer is running in headless mode?

Use the AppSuite4 API with the PF_IsRenderEngine function. Create a PF_Boolean variable and call suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine) to set it to true if running in render engine mode.

```cpp
PF_Boolean bIsRenderEngine = true;
suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine);
```

*Tags: `aegp`, `debugging`, `deployment`, `render-loop`*

---

### How can DirectX compute be enabled for debugging in After Effects?

DirectX compute can be enabled by setting the 'forcedirectxcompute' flag in the console panel within AE settings. However, a full computer restart is required for the setting to take effect; restarting AE alone is not sufficient. Once properly configured, it becomes possible to debug DirectX for future productions.

*Tags: `debugging`, `directx`, `gpu`, `windows`*

---

### How do you disable the crash warning dialog in After Effects 2026 on macOS?

Use cmd + F12 to switch to database view, then search for "crash window" or similar to locate and disable the crash warning.

*Tags: `debugging`, `macos`, `ui`*

---

### How can you prevent an After Effects plugin from being detected and displayed in Premiere's effects list?

Return an error in global setup, since Premiere runs global setup for all plugins during application startup. If the plugin returns an error during this initialization phase, it should be hidden from the effects list.

*Tags: `debugging`, `pipl`, `premiere`*

---

### How do 3D Camera Tracker and Warp Stabilizer output progress over frames with live updates without cache purging?

According to an After Effects developer, these built-in tools use internal APIs that are not available to regular plugin developers. The Warp Stabilizer effect 'cheats' and does not operate within the constraints of a normal AE Effect API plug-in. Adobe's internal team has access to secret suites that allow this functionality, which is not exposed to third-party developers. A feature request was made in 2023 to expose this capability to plugin developers, but it remains unavailable.

*Tags: `aegp`, `caching`, `debugging`, `mfr`, `smart-render`*

---

### Where can I find discussions about transform_world and motion blur issues in After Effects?

Rich shared a question posted on the Adobe After Effects community forum discussing transform_world and motion blur. The forum thread can be found at https://community.adobe.com/t5/after-effects-discussions/transform-world-amp-motion-blur/td-p/12918545

*Tags: `debugging`, `motion-blur`, `reference`, `transform`*

---

### How do you properly initialize ImGui with OpenGL on macOS?

When initializing ImGui with OpenGL on macOS, window flags must be set before window creation, not after. This differs from Windows behavior and is a common mistake. The OpenGL loader initialization fails if flags are set in the wrong order during the setup process.

*Tags: `debugging`, `macos`, `opengl`, `ui`*

---

### What does the C specification say about pointer alignment and undefined behavior?

According to the C specification (section 6.3.2.3, paragraph 7 of the C11 standard), a pointer to an object type may be converted to a pointer to a different object type, but if the resulting pointer is not correctly aligned for the referenced type, the behavior is undefined. Apple documents this issue in their Xcode documentation on misaligned pointers: https://developer.apple.com/documentation/xcode/misaligned-pointer

*Tags: `cross-platform`, `debugging`, `memory`, `reference`*

---

### How do you troubleshoot colorspace and iteration suite issues in Premiere?

When working with Premiere's colorspace and iteration suite, start by clarifying which specific colorspace was chosen, as this is often the root cause of issues. The problem may stem from colorspace selection or configuration within the iteration workflow.

*Tags: `color`, `debugging`, `premiere`*

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

### How can you prevent the compiler from optimizing a render function in an After Effects plugin?

Add the `__attribute__ ((optnone))` compiler attribute before the render function declaration. This tells the compiler to skip optimization for that specific function, even if the rest of the plugin is compiled with optimizations enabled. This workaround was found to resolve rendering issues in some cases.

```cpp
__attribute__ ((optnone))
void renderFunction() {
  // render code
}
```

*Tags: `build`, `debugging`, `macos`, `render-loop`, `windows`*

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

### Why does a plugin fail with 'Not able to acquire AEFX Suite' error when playing in Premiere?

The issue may occur during render/playback if the plugin calls AEFX Suite in a shared thread or has incorrect host application detection logic. Check if there is conditional code that calls AEFX Suite only when the host is NOT Premiere (e.g., `if (app_id != 'PrMr') => call AEFX_suite`), but the host detection returns an incorrect value in Premiere. This can cause the plugin to attempt AEFX Suite calls in Premiere where they are not available. Premiere can also have unexpected behavior with certain suites, such as colorspace-related ones.

*Tags: `aegp`, `debugging`, `premiere`, `threading`*

---

### If a resource fails to allocate, does it still need to be disposed?

No, if a resource fails to allocate, it does not need to be disposed of. Only successfully allocated resources require cleanup and disposal.

*Tags: `debugging`, `memory`, `resource-management`*

---

### How can you test whether memory disposal is working correctly in an After Effects plugin?

One practical approach is stress-testing by rendering a long video (e.g., 5 hours) to identify memory leaks. If the plugin runs without crashing and memory usage remains stable throughout the render, it indicates that necessary resources have been properly disposed. This method relies on observation rather than formal memory profiling tools.

*Tags: `debugging`, `disposal`, `memory`, `testing`*

---

### Why does a displacement map effect sometimes use the original frame instead of the previously cached frame?

Nate is experiencing inconsistent behavior with a displacement mapping effect that should feed the distorted output back as input for the next frame. The effect sometimes works correctly (distorting and caching each frame to use as the next input) but randomly reverts to using the original frame instead of the cached frame. This suggests a potential issue with the frame caching or smart render logic that determines which input frame gets used for displacement. The problem appears to be intermittent rather than systematic.

*Tags: `caching`, `debugging`, `render-loop`, `smart-render`*

---

### How can you debug why a cached frame is not being used even though it should be?

To diagnose caching issues, first determine whether the frame is always cached but not always used, or if it's not being cached consistently. If it's always cached but not always used, the problem lies in the conditional logic that checks cache validity. Start by disabling conditions in the if statement that verify cache availability (such as hash checks) to isolate which condition is returning false and preventing cache usage.

*Tags: `caching`, `debugging`, `memory`, `smart-render`*

---

### Why is a plugin exhibiting glitch behavior with Multi-Frame Rendering (MFR) even though the Multithreaded flag is not set?

MFR glitches can occur when different threads/CPUs are using different previous frames, causing inconsistent rendering behavior. Even without the Multithreaded flag explicitly set, the plugin may still attempt MFR, which can lead to these issues. Disabling MFR resolves the problem. The root cause appears to be improper frame state management across threads during multi-frame rendering.

*Tags: `debugging`, `mfr`, `render-loop`, `threading`*

---

### How do you hide an arbitrary parameter banner in the timeline while keeping it visible in the Effect Control Window?

When working with arbitrary parameters in After Effects, the visibility in the timeline versus the Effect Control Window (ECW) is tied to whether the parameter is keyframeable. Static banners that don't need keyframing can be hidden from the timeline while remaining visible in the ECW by ensuring they are not set up as keyframeable streams. The Color Grid effect demonstrates having parameters visible in both locations because they are keyframeable, but for static-only parameters, the visibility behavior differs.

*Tags: `arb-data`, `debugging`, `params`, `ui`*

---

### What causes an error when working with arbitrary parameters in After Effects plugins?

An error occurs when you don't provide a name for an arbitrary parameter (arb param). Always ensure that arbitrary parameters are given explicit names to avoid this error.

*Tags: `arb-data`, `debugging`, `params`*

---

### What causes an infinite progress bar at the end of an effect render?

An infinite progress bar can be triggered by an incoherent PF_Progress value. Setting impossible values like 200/100 (where the current progress exceeds the total) will display this infinite progress bar effect.

*Tags: `aegp`, `debugging`, `ui`*

---

### How can you diagnose whether a crash is related to RAM caching issues in After Effects plugins?

Try changing the size of the image or color depth of the project to see if the crash frame number changes. This can help determine if the issue is related to RAM access handling, particularly with time-varying data or conflicts between sequence data writing and MFR (Multiprocessing Framework).

*Tags: `caching`, `debugging`, `memory`, `mfr`, `sequence-data`*

---

### Is it illegal to checkout a parameter multiple times in After Effects plugins?

Checkout of parameters can be problematic in certain lifecycle stages. You cannot perform parameter checkout during sequence setup, resetup (when AE is loading the project), or sequence setdown. It's important to handle checkout errors properly and ensure that checkin is called immediately after using the parameter value, rather than holding the checkout open.

*Tags: `aegp`, `debugging`, `layer-checkout`, `params`*

---

### What should be done after checking out a parameter value in the render loop?

After getting a parameter value through checkout in a loop, you must call checkin immediately after using the value. Holding the checkout open or delaying checkin can cause errors like 'bad parameter passed to effect callback'. Always pair checkout with checkin as soon as the parameter data is no longer needed.

*Tags: `debugging`, `layer-checkout`, `params`, `render-loop`*

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

### How is it possible for a customer to have a newer GPU driver version than me on an older macOS version?

On Apple systems, drivers are typically bundled with the OS, making newer drivers on older OS versions unusual. One possibility is manual driver installation similar to Hackintosh setups, though this is not straightforward. Additionally, Apple may have issued driver downgrades in response to GPU issues in specific macOS versions (e.g., macOS 12.3 had known issues with AMD 5xxx and 6xxx GPUs). See: https://www.reddit.com/r/hackintosh/comments/ter3g2/psa_macos_monterey_123_and_amd_5xxx_and_6xxx_gpu/

*Tags: `debugging`, `gpu`, `macos`*

---

### How can I diagnose missing DLL dependencies in a Windows After Effects plugin?

Use Dependency Walker on Windows to check for missing DLLs and verify that all libraries and DLLs have the correct architecture (32-bit vs 64-bit) matching your plugin build. Also check system folders like System32 where driver-installed libraries like Vulkan may be located.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What does After Effects error 25:3 mean and what causes it?

Error 25:3 in After Effects indicates that an effect cannot be applied because it cannot be initialized. This error is typically associated with a corrupted installation or missing dependencies. It has been observed to occur specifically when users are running After Effects over Windows Remote Desktop software, suggesting potential issues with plugin initialization in remote desktop environments.

*Tags: `debugging`, `deployment`, `plugin-error`, `windows`*

---

### What causes After Effects error 25:3 when using plugins over remote desktop software?

Error 25:3 ('This effect cannot be applied because it cannot be initialized') typically indicates a corrupted installation or missing dependency. However, when the error occurs specifically for users accessing After Effects through Windows Remote Desktop, it may be related to remote desktop-specific issues rather than actual corruption. The problem appears to be isolated to remote desktop environments rather than standard installations.

*Tags: `debugging`, `deployment`, `windows`*

---

### How do you set up Vulkan development on macOS with MoltenVK compatibility?

To develop Vulkan plugins on macOS with MoltenVK: (1) Update to at least October release to get MoltenVK compatible with Vulkan 1.2. (2) For debug builds, create a target using ICD (Installable Client Driver) in environment variables and dylib linking, then add the "VK_KHR_portability_subset" instance extension (debug-only until MoltenVK uses it natively) along with debugging layers. (3) For release builds, link only to frameworks as static libraries to comply with Vulkan 1.2 limitations.

*Tags: `build`, `cross-platform`, `debugging`, `macos`, `vulkan`*

---

### Why does ImGui fail to initialize OpenGL on macOS with Apple Silicon (M1)?

OpenGL initialization failures on M1 Macs are often related to Apple Silicon compatibility issues. ImGui can run over Metal instead of OpenGL, though converting from OpenGL matrices to Metal requires additional work. The issue may be related to GLSL version differences between macOS and Windows, or Metal API requirements on Apple Silicon.

*Tags: `apple-silicon`, `debugging`, `macos`, `metal`, `opengl`*

---

### How can you obtain a drawing reference in After Effects plugin render loops without requiring user interaction?

The standard approach using EffectCustomUISuite1()->PF_GetDrawingReference() only provides a drawing context during PF_Cmd_EVENT callbacks triggered by user interaction. To draw during render without user interaction (similar to VFX Stabilizer behavior), you need an alternative method to access the drawing reference that doesn't depend on the extraPV parameter passed through PF_EventExtra. This requires investigating lower-level drawing APIs or background rendering approaches that don't rely on event-driven context delivery.

*Tags: `aegp`, `debugging`, `render-loop`, `ui`*

---

### How can you draw to the composition window during rendering without relying on user interaction events?

The challenge is that PF_GetDrawingReference() is only available through PF_EventExtra during PF_Cmd_EVENT, which requires user interaction with the effect parameters or composition preview. To draw during a render loop without user interaction (similar to VFX Stabilizer behavior), you need an alternative approach to obtain the drawing reference that isn't tied to user interaction events. This is a difficult problem with limited documented solutions; consulting experienced plugin developers like Shachar Gran is recommended.

*Tags: `aegp`, `debugging`, `render-loop`, `ui`*

---

### Where can I find discussion and solutions about PF_PROGRESS bar not showing in After Effects plugins?

Adobe Community has a discussion thread about PF_PROGRESS bar issues: https://community.adobe.com/t5/after-effects-discussions/progress-bar-won-t-show-using-pf-progress/m-p/10474224. The thread discusses limitations of progress reporting in the render function, particularly when the main thread is blocked during computation.

*Tags: `debugging`, `reference`, `ui`*

---

### How do you print to the console when debugging an After Effects plugin?

On Mac, use printf. On Windows, use OutputDebugStringA macro to output debug messages that will appear in Visual Studio's console. Note that AEGP_WriteToOSConsole from the utility suite exists but may not work reliably. Starting AE with the -debug flag can help, but OutputDebugStringA is the more reliable approach for Windows debugging.

```cpp
OutputDebugStringA("Debug message here");
```

*Tags: `aegp`, `debugging`, `macos`, `windows`*

---

### Why does a plugin render get interrupted with PF_Interrupt_CANCEL during queue rendering?

PF_Interrupt_CANCEL during render queue operations can occur under specific conditions. One user reported this happening after 10-15 frames when CAPS LOCK was off and the composition had not been previously previewed to RAM cache. The interrupt appears as 'User abort' but occurs without user action. The behavior did not occur when CAPS LOCK was on or when the composition was fully previewed in the RAM cache before rendering. This suggests the interrupt may be related to input handling or cache state during rendering.

*Tags: `debugging`, `render-loop`, `smartfx`*

---

### How do you force After Effects to re-check whether frames need invalidating when using sequence data?

When sequence data isn't flattened, After Effects may not refresh frames as expected even when the data updates properly (verified via breakpoints). A workaround is to give AE a "kick" by changing composition background color or toggling a parameter, which forces it to check for invalidation. Alternatively, see the Adobe Community discussion on forcing rendering on events in separate threads: https://community.adobe.com/t5/after-effects-discussions/force-rendering-on-an-event-in-a-separate-thread-coming-back-to-the-main-ui-thread-on-an-event/m-p/13628197

*Tags: `debugging`, `render-loop`, `sequence-data`, `threading`*

---

### How should DrawImage be called in drawbotSuites to ensure correct opacity and alpha rendering?

When using drawbotSuites.surface_suiteP->DrawImage, you must use an opacity of 1.0. If you don't use 1.0 opacity, the drawing will appear darker than intended and the alpha will still show as 1.0 since After Effects' UI will be behind it with solid alpha. This is important for correct compositing behavior.

```cpp
drawbotSuites.surface_suiteP->DrawImage(/* ... */, 1.0 /* opacity */, /* ... */)
```

*Tags: `aegp`, `debugging`, `drawing`, `ui`*

---

### Why does text appear faint when using FillPath and StrokePath in After Effects plugin UI?

When writing on UI using FillPath and StrokePath with opacity set to 1.0, faint text may result from After Effects UI brightness preferences or project color space compensation. The After Effects suite may automatically compensate for these settings, affecting how drawing operations are rendered on the UI.

*Tags: `aegp`, `debugging`, `ui`*

---

### How can you implement a custom text editor in the Effect Control Window that captures key events like Tab?

When implementing a custom text editor in the ECW using Custom UI events, PF_Event_KEYDOWN events are available but don't include an index for the particular param. Additionally, some keys like Tab don't reach the KEYDOWN handler. To solve this, you need to claim key-focus similar to how built-in slider numeric value editing works, which allows the plugin to intercept all keyboard input including Tab and other special keys that would normally be consumed by the host.

*Tags: `aegp`, `custom-ui`, `debugging`, `ui`*

---

### What causes the 'Invalid Filter 25::3' error on macOS in After Effects?

The 'Invalid Filter 25::3' error on macOS can be caused by a mismatch between the Deployment Target set in Xcode and the macOS version running After Effects. If the Deployment Target is set higher than the OS version (e.g., setting Deployment Target to 12.3 when running on Big Sur 11.17.9), it will cause this error.

*Tags: `build`, `debugging`, `deployment`, `macos`*

---

### How can you add conditional logging to an After Effects plugin without slowing down renders?

Implement a file-based toggle for logging by checking for a specific file (e.g., 'pixel_sorter_log.txt') in global setup at startup. This allows users to enable detailed logging only when needed for debugging, without impacting render performance during normal operation.

*Tags: `debugging`, `logging`, `macos`, `performance`, `plugin-development`*

---

### What logging library is recommended for After Effects plugins with file rotation capabilities?

spdlog is recommended for plugin logging. It provides file rotation functionality where you can define the maximum size of individual log files and how many log files to retain, making it suitable for long-running plugin operations.

*Tags: `debugging`, `logging`, `open-source`, `tool`*

---

### How can you debug plugin initialization issues by tracking function entry and exit points?

Add logging statements at the entry and exit of key plugin functions, particularly in global setup and parameter setup stages. This helps identify at which stage crashes or entrypoint errors are occurring, making it easier to distinguish between different error types.

*Tags: `aegp`, `debugging`, `plugin-development`*

---

### How do you completely clean the Xcode build cache for After Effects plugin development?

Use Cmd+Shift+K in Xcode to clean the build folder. This cleans everything in the build cache. Additionally, resetting After Effects preferences can help resolve persistent build issues.

*Tags: `build`, `debugging`, `macos`*

---

### Why is PF_Arbitrary_NEW_FUNC selector never received, causing arbitrary data to always be zero length initially?

This is a known behavior where the PF_Arbitrary_NEW_FUNC selector is not consistently triggered during arbitrary data initialization. The user reported that their arbitrary data always starts with zero length despite expecting the NEW_FUNC selector to be called. This appears to be an obscure aspect of the arbitrary data lifecycle that requires alternative initialization strategies.

*Tags: `aegp`, `arb-data`, `debugging`, `params`*

---

### What causes 'entry point not found' errors on Windows when an After Effects plugin loads?

This error typically indicates a missing dependency, most commonly a missing Visual C++ redistributable package. Even if Visual Studio is installed on the development machine, the target machine may not have the required Visual C++ runtime libraries. On Windows 10, installing the Visual C++ 2022 redistributable (https://aka.ms/vs/17/release/vc_redist.x64.exe) often resolves the issue. Debugging can be done by setting a breakpoint on global setup in debug mode to verify if the plugin initialization is being reached.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### Is it normal to see msvcp140d.dll when compiling an After Effects plugin with Visual Studio 2022 (v143)?

Yes, this is normal behavior. The msvcp140d.dll file (Visual C++ Debug Runtime) appears in the compiled output when using Visual Studio 2022. However, if you're encountering entry point errors, the issue is likely related to other missing dependencies or libraries that the plugin relies on, rather than the presence of this DLL.

*Tags: `build`, `debugging`, `windows`*

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

### Why is layer pixel data NULL after checkout_layer_pixels for a selected layer parameter?

When checkout_layer_pixels returns a non-NULL world pointer but the data field within it is NULL, this typically indicates that the layer parameter has no layer selected or the layer is disabled/invisible. Ensure that in your layer parameter definition you're using an appropriate default (like PF_LayerDefault_NONE or a specific layer) and verify in PreRender that the layer checkout succeeded. The issue may also resolve itself after rebuilding or reloading the plugin. Check that the layer parameter is actually populated with a valid layer selection before SmartRender is called.

```cpp
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `debugging`, `layer-checkout`, `params`, `smart-render`*

---

### Why is checkout_layer_pixels returning NULL data for a layer parameter even though the layer is selected?

Tim Constantinov reported an issue where after calling checkout_layer_pixels on a layer selected in a layer parameter, the data field equals NULL. The problem occurs in SmartRender after checkout_layer_pixels is called on GPU_SKELETON_LAYER. The issue appears to be intermittent and may be related to the Adobe-supplied GPU template. The exact cause wasn't definitively identified in the conversation, but Tim noted this issue resolved itself spontaneously in the past.

```cpp
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `debugging`, `gpu`, `layer-checkout`, `params`, `smartfx`*

---

### How can you debug a plugin crash that occurs during project loading in After Effects on Windows but not on Mac, seemingly triggered by an external library like Cineware?

Use breakpoints in EffectMain to trace the plugin's initialization sequence across GlobalSetup and ParamsSetup. If the crash occurs well after the plugin's code executes (with no breakpoints hit for several seconds), it suggests the plugin may be inadvertently affecting how other libraries like Cineware load, possibly through symbol name clashing. A workaround is to pre-load the problematic library (e.g., Cineware) onto a clip before opening the project. Additionally, try importing the project into a new empty project rather than directly opening it, as project corruption or platform-specific state issues may be the root cause. Cross-platform differences (Mac vs Windows) suggest OS-specific library loading or initialization issues.

*Tags: `cross-platform`, `debugging`, `macos`, `plugin-interaction`, `windows`*

---

### What is the correct C++ syntax for initializing multiple PF_Err variables in After Effects plugins?

When declaring multiple PF_Err variables, you must initialize each one explicitly. Incorrect: `PF_Err err, err2 = PF_Err_NONE;` (only err2 is initialized, err is undefined). Correct: `PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;` (both variables initialized). This matters because uninitialized error codes will not evaluate to 0 in conditional statements like `if(!err)`, causing logic failures.

```cpp
// Incorrect - err is undefined
PF_Err err, err2 = PF_Err_NONE;

// Correct - both initialized
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `aegp`, `debugging`, `pipl`*

---

### Why doesn't Instruments detect memory leaks from PF_EffectWorlds allocated in After Effects plugins?

Memory leaks from PF_EffectWorlds may not be detected by Instruments because these objects are allocated deep within the After Effects engine rather than directly within the plugin code. The leak detection tools only see allocations at the plugin level, not at the internal AE engine memory allocation level where EffectWorlds are actually created. This means that even significant memory leaks from unmanaged EffectWorlds won't show up in Instruments despite causing the application to run out of memory.

*Tags: `aegp`, `debugging`, `macos`, `memory`*

---

### Why does macOS Instruments fail to detect memory leaks from undisposed PF_EffectWorlds in After Effects plugins?

James Whiffin reported that while Instruments successfully detected memory leaks from data allocated in pre-render functions that weren't disposed, it failed to detect leaks from PF_EffectWorlds that were allocated but never disposed. Despite these allocations consuming gigabytes and causing After Effects to run out of memory, they didn't appear in Instruments' leak detection output even when sorted by size. This suggests that PF_EffectWorlds may be allocated through memory management mechanisms that Instruments cannot track, or that After Effects' memory management for effect worlds operates outside standard system memory tracking.

*Tags: `aegp`, `debugging`, `macos`, `memory`*

---

### How do you disable the crash warning popup in After Effects 2024?

Press CMD+F12 to open the Debug Database console. Click the hamburger menu next to the console and change to Debug Database. Search for 'ShowPreviousCrashWarning' and turn it off. This setting can also be configured in the prefs.txt file and can be scripted for automation.

*Tags: `debugging`, `macos`, `scripting`, `ui`*

---

### What is the relationship between macOS and Xcode versions for After Effects plugin development?

When updating macOS, you often need to update Xcode to a compatible version. For example, updating from macOS 13.x to macOS 14 required updating from Xcode 14 to Xcode 15. Windows and Visual Studio are more flexible in this regard. The best practice for plugin development is to maintain separate development and testing machines with different OS versions to catch version-specific bugs and unexpected behavior, though this is an expensive option.

*Tags: `build`, `cross-platform`, `debugging`, `macos`, `windows`*

---

### What is the challenge of timing OS and development tool updates during plugin development?

Developers face a difficult choice between updating to new versions quickly (to catch bugs and unexpected behavior that only occur on the latest versions) and waiting to ensure stability and compatibility with their products. Rolling back an entire OS is impractical, so alternatives include waiting for the next patch of Xcode or maintaining separate machines for development and testing at different version levels.

*Tags: `build`, `debugging`, `deployment`, `macos`*

---

### How can you work around SDK issues when using Xcode 14 with After Effects plugin development?

Run the Xcode binary directly from Terminal instead of launching Xcode.app. The command is '/Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode' (replace with your Xcode version). You can download specific Xcode versions from https://developer.apple.com/download/all/. For quick access, add the /Contents/MacOS folder to your Finder sidebar, then right-click the binary, hold Alt/Option, click 'Copy as Pathname', paste into Terminal and execute. This takes 3-4 seconds versus the normal Spotlight launch method.

*Tags: `build`, `debugging`, `macos`, `xcode`*

---

### Why is After Effects taking a very long time to load in the debugger after upgrading to Xcode 15?

The issue may be related to symbol loading being enabled in your debugger settings. Check your Xcode debugger options and disable symbol loading if it's turned on, as this can significantly slow down AE launch times. In Visual Studio, you can stop symbol loading during debugging to speed this up.

*Tags: `build`, `debugging`, `macos`, `performance`*

---

### How can a plugin distinguish between a render queue render and a user scrubbing the timeline in the composition to handle GPU memory errors appropriately?

James Whiffin raised this as an unsolved problem in AE plugin development. When a plugin returns PF_Err_OUT_OF_MEMORY, AE may not warn the user if they're scrubbing the timeline (resulting in a black frame), but the plugin cannot reliably distinguish between queue renders and interactive scrubbing. While returning the error code prevents render queue failures (allowing AE to retry with lower thread counts), there's currently no documented method to detect the render context and conditionally display user warnings via out_data. This remains a limitation when plugins need multiple 32bpc GPU buffers at high resolutions.

*Tags: `debugging`, `gpu`, `memory`, `mfr`*

---

### How should a plugin handle GPU out-of-memory errors differently for interactive scrubbing versus render queue operations?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects handles it differently depending on context. During MFR (Multi-Frame Rendering) queue renders, AE may retry with a lower thread count. However, during interactive timeline scrubbing, AE doesn't warn the user and may display a black frame instead. The challenge is that PF_Err_OUT_OF_MEMORY warnings in out_data will fail the render, and there is currently no documented method to distinguish between a render queue request versus interactive scrubbing. Plugins must accept that they cannot reliably warn users about resolution/bitdepth being too high for available GPU memory in interactive contexts without failing the frame.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `smartfx`*

---

### How can you display a warning alert in an After Effects plugin without blocking the render thread?

Create a static global boolean variable that is modified by a mutex in the render thread, then check and print the warning if the boolean is true in another thread such as pFupdate_param_UI. This approach prevents blocking the render thread while still allowing warnings to be displayed to the user.

```cpp
static bool g_warning_flag = false;
static std::mutex g_warning_mutex;

// In render thread:
{
  std::lock_guard<std::mutex> lock(g_warning_mutex);
  g_warning_flag = true;
}

// In pFupdate_param_UI thread:
if (g_warning_flag) {
  // Display warning alert
  std::lock_guard<std::mutex> lock(g_warning_mutex);
  g_warning_flag = false;
}
```

*Tags: `debugging`, `render-loop`, `threading`, `ui`*

---

### Why might an AEGP not appear in After Effects and fail to hit any breakpoints during debugging?

This is a Mac-specific AEGP question about plugin visibility and debugging. The issue could be related to security settings or allocation changes on macOS, particularly around code signing, entitlements, or plugin sandbox restrictions. Possible causes include: incorrect plugin bundle structure, code signing issues, missing entitlements, or the plugin not being properly registered in After Effects' plugin directory.

*Tags: `aegp`, `debugging`, `deployment`, `macos`*

---

### What operator caused a GPU driver issue in After Effects plugin development?

Using the /= operator instead of the longhand division assignment can cause GPU driver issues. James Whiffin reported a ticket where users' GPU drivers failed due to this operator choice, suggesting that drivers may have inconsistent support for compound assignment operators.

*Tags: `cross-platform`, `debugging`, `gpu`*

---

### What GLSL assignment operators are supported across different GPU platforms in After Effects plugins?

James Whiffin encountered an issue where a specific GLSL operator caused compilation failures. The error was identified through shader compilation feedback. The conversation indicates that while some assignment operators like += work correctly, the /= operator caused issues on at least one platform (Windows or Mac, GPU reference unknown). This suggests developers need to be cautious about which GLSL assignment operators are universally supported across different GPU architectures when developing AE plugins.

*Tags: `cross-platform`, `debugging`, `glsl`, `gpu`, `macos`, `opengl`, `windows`*

---

### Why does checking out a layer parameter in PreRender still return the layer before all effects are applied?

When using PF_CHECKOUT_PARAM to check out a layer (INPUT) in the PreRender callback, the layer is returned in its pre-effect state rather than with effects applied. This is expected behavior in the AE plugin architecture—the checkout at PreRender stage occurs before the effect chain is fully processed.

*Tags: `aegp`, `debugging`, `layer-checkout`, `render-loop`*

---

### How can you create a custom banner in the After Effects effects menu like FxFactory plugins do?

This appears to be a FxFactory-specific feature. The discussion suggests several investigation approaches: comparing AE installation folders before and after FxFactory installation to identify changes, examining the .aex plugin file and its PiPL resources for unknown SDK functions, or contacting FxFactory developers directly. It's unclear whether the banner is drawn through standard AEGP suite functions or through custom modifications to the plugin manager.

*Tags: `aegp`, `debugging`, `pipl`, `reverse-engineering`, `ui`*

---

### What are the challenges with using third-party GPU debuggers for After Effects plugins?

GPU debuggers like gDEBugger, RenderDoc, and others frequently crash or fail when tracing After Effects. The core issue is that these debuggers are designed for standalone games with permanent instances, whereas plugin architectures within host applications like After Effects or Premiere Pro create a more complex environment. Additionally, debuggers may capture the host application's graphics API (DirectX) instead of the plugin's rendering API (OpenGL, Vulkan), making them ineffective for plugin-specific debugging.

*Tags: `debugging`, `gpu`, `opengl`, `plugin-architecture`, `vulkan`*

---

### Why does a plugin sometimes render without the effect applied when queued to Media Encoder from After Effects?

Static global variables in the plugin and elements defined in globalData or during global initialization thread can cause issues with Media Encoder compatibility. These should be reviewed and potentially refactored to avoid state persistence issues when the plugin runs in ME's different execution context.

*Tags: `debugging`, `media-encoder`, `memory`, `threading`*

---

### How can you generate a compute cache key when the key generation function only accepts AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP but requires AEGP_HashSuite1 which needs in_data?

Tim Constantinov asked about the apparent conflict between the parameters accepted by the key generation function and the requirements for using AEGP_HashSuite1. The question highlights that to generate a key, you need access to in_data or in_data->pica_basicP, but the function signature seems to only provide AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP. This appears to be an unresolved technical question about the proper API usage pattern.

*Tags: `aegp`, `compute-cache`, `debugging`*

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

### How can I use RenderDoc to debug GPU code in After Effects plugins without crashes?

Use RenderDoc's in-application API with process injection. Enable process injection in Tools > Settings > "Enable process injection" (hidden by default). Launch AE fully, then inject RenderDoc via File > Inject into Process before applying your plugin. In your plugin's PF_Cmd_GLOBAL_SETUP, load the RenderDoc API and use the rdoc_api pointer to manually begin and end frame captures in your render function. This approach avoids the crashes that occur when launching RenderDoc normally with AE.

```cpp
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll"))
{
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
```

*Tags: `debugging`, `gpu`, `render-loop`, `windows`*

---

### What is an easy way to calculate and verify the correct bitmask values for PiPL outflags and outflags2 fields?

Tobias Fleischer created a quick online lookup/cheat sheet tool in JavaScript that allows developers to easily find and verify the correct bitmask values for PiPL outflags and outflags2 fields without needing to run the host application and receive warnings. The tool is available at: https://www.reduxfx.com/piplflags/

*Tags: `build`, `debugging`, `pipl`, `reference`, `tool`*

---

### What suites are unsafe to call from the Death Hook in After Effects plugins?

There are restrictions on which suites can be safely called from the Death Hook. The Utility Suite, specifically ReportInfo, has been reported to cause unhandled exceptions when called from Death Hook. This suggests that certain suites with external dependencies or state management are not safe to invoke during plugin shutdown, and developers should be cautious about suite usage in Death Hook contexts.

*Tags: `aegp`, `debugging`, `memory`, `threading`*

---

### How can you reverse-engineer and integrate a DLL into an After Effects plugin?

One approach is to use depends.exe to inspect the DLL's dependencies and exports, then reference existing samples (like the Illustrator sample) as a guide to understand the integration pattern. While not always a 1:1 match, samples can provide enough context to build your own wrapper. Modern approaches may involve directly linking to the DLL and providing a header file, but manual reverse-engineering through dependency analysis and sample-based implementation has proven effective.

*Tags: `build`, `debugging`, `reference`, `windows`*

---

### Can you display dialogs during the Global Setup phase of a plugin?

According to discussions in the community, it should be possible to show dialogs during Global Setup. However, there is some uncertainty about whether Global Setup is called during the AE loading screen when scanning plugins, or only when the effect is first applied. The capability exists, but the exact execution context may affect dialog behavior.

*Tags: `aegp`, `debugging`, `pipl`, `ui`*

---

### What is the official Adobe documentation for pixel sampling and layer manipulation in After Effects plugins?

Adobe provides comprehensive documentation at https://ae-plugins.docsforadobe.dev/effect-details/tips-tricks.html covering sampling pixels at X,Y coordinates and related plugin development techniques.

*Tags: `aegp`, `debugging`, `pipl`, `reference`*

---

### What are the common performance bottlenecks when working with large numbers of layers in After Effects plugins?

Performance issues with large layer counts (800+ layers) typically stem from UI rendering rather than the render engine itself. The bottleneck often comes from repeated After Effects API calls like 'get layer info' and 'get layer sprite', not from Artisan optimization. Additionally, After Effects has poor memory management with large layer counts, making it inefficient to handle thousands of layers.

*Tags: `aegp`, `debugging`, `memory`, `performance`, `ui`*

---

### How can you ensure sequence data stays synchronized across render threads when duplicating a plugin instance?

When duplicating a plugin and assigning a new unique ID to sequence data, the sequence data may become out of sync on render threads even though UpdateUI and UserChangeParams receive the correct values. Two potential solutions: (1) Check if the sequence is flattened in the project, as flattening can force render threads to get the correct sequence value. (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of relying on in_data->seq_data directly, as the latter is not updated correctly since CC2021/2022.

*Tags: `debugging`, `params`, `render-loop`, `sequence-data`, `threading`*

---

### What performance improvements were observed after Adobe fixed arb handling?

Adobe recently fixed an issue in arb (arbitrary data) handling that resolved copy/paste effect performance issues. A notable case involved applying Trapcode, which showed approximately 7x speed improvement after the fix, though this may have been an unintended consequence of the arb handling changes.

*Tags: `arb-data`, `debugging`, `performance`*

---

### What causes custom UI artifacts and glitching in After Effects 2025 when using DrawBot with arbitrary data parameters?

The issue is not directly related to arbitrary data (arb) parameters, but rather to how DrawBot's Image buffer compositing works. The problem occurs when the UI composites multiple images per plugin context. The culprit appears to be NewImageFromBuffer, which on every call sets the pixels of all retained DRAWBOT_ImageRef objects within the same context, causing visual artifacts like flashes, scrambled, or duplicated UI elements. The issue can be avoided by either not compositing any image in the UI, or by only compositing a single image per plugin context.

*Tags: `ae-2025`, `arb-data`, `debugging`, `drawbot`, `ui`*

---

### Is it expected to receive thousands of arb dispose calls when shutting down After Effects?

Yes, this is expected behavior. After Effects' allocators don't dispose of resources until available memory is saturated or on thread destruction/exit. You can test this behavior by artificially setting the available memory to a low value like 2GB, which will trigger more destruction calls during runtime.

*Tags: `aegp`, `arb-data`, `debugging`, `memory`*

---

### What change in After Effects 2025 beta affects AEGP_StartUndoGroup with null parameter?

Starting from After Effects 2025 beta, passing null to AEGP_StartUndoGroup will cause a crash. In previous versions, this did not cause adverse effects, but it is no longer safe to do so.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup(null);
```

*Tags: `aegp`, `api-change`, `crash`, `debugging`, `macos`, `windows`*

---

### What parameter should be passed to AEGP_StartUndoGroup to avoid crashes in After Effects 2025?

As of After Effects 2025 beta, passing null to AEGP_StartUndoGroup() causes a crash, whereas passing an empty string "" works correctly and functions as expected without adding an entry to the Undo stack. Previous versions did not crash with null, so this is a breaking change in AE 2025.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `debugging`, `macos`, `windows`*

---

### Why is the params pointer always nullptr when handling PF_Cmd_EVENT?

When handling PF_Cmd_EVENT, you need to check the event type via extra->e_type first. You can only modify param values for certain event types, specifically click and keyboard events. The params pointer may not be available in all event contexts, and you may need to use AEGP functions instead to modify parameters depending on the event type.

```cpp
case PF_Cmd_EVENT:
    err = HandleEvent(in_data, out_data, params);
    // Check extra->e_type for event type (click, keyboard, etc.)
    switch(extra->e_type) {
        case PF_Event_CLICK:
        case PF_Event_KEYDOWN:
            // Can modify params here
            break;
    }
```

*Tags: `aegp`, `debugging`, `params`, `ui`*

---

### How can you reliably get the After Effects version in a plugin when in_data->version major and minor don't update between releases?

James Whiffin noted that in_data->version major and minor fields are not being updated reliably between AE 2024 and 2025, returning the same values. This suggests developers should investigate alternative methods for version detection, such as querying the application directly or using other fields in the plugin data structure, though no specific solution was provided in this conversation.

*Tags: `aegp`, `debugging`, `macos`, `windows`*

---

### What causes the 'Cannot run a script while a modal dialog is waiting for response' error when calling scripting from a plugin?

This error occurs when attempting to run scripting from a plugin while a modal dialog is active and waiting for user response. It is a limitation of using scripting from plugins - the scripting engine cannot execute while After Effects has a modal dialog blocking the main thread. This can happen even after checking if scripting is available first, and may occur during parameter setup.

*Tags: `aegp`, `debugging`, `scripting`, `ui`*

---

### How can you determine the After Effects version at runtime in a plugin?

You can use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP as the path type to locate the running binary host, then extract the version from the AE folder name or binary metadata. Alternatively, you can try mapping AEGP_GetDriverSpecVersion to the AE version number, though it's unclear if AEGP version increments in parallel with the host version.

*Tags: `aegp`, `cross-platform`, `debugging`, `deployment`*

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

### How do you debug 'failed to load' errors in After Effects and 'filter offline' errors in Premiere Pro?

When debugging plugin loading failures, check whether the error occurs before global_setup is called. If it happens before global_setup, it may indicate a library dependency issue loaded by AE/Premiere in a newer version that conflicts with your plugin. Test on both Windows and macOS to determine if the issue is platform-specific, as the same plugin may work on one platform but fail on another.

*Tags: `cross-platform`, `debugging`, `deployment`, `macos`, `premiere`, `windows`*

---

### Why are plugins failing to load on Apple Silicon Macs in After Effects 25.2 and 25.3?

According to investigation by Maxon and Adobe, the issue appears to be related to code signing. Apple may have introduced stricter code-signing requirements or hardening behavior in newer Xcode versions. There seems to be a potential interaction between After Effects' own code signatures and plugin signatures, though the exact cause is still under investigation. Even plugins that are properly signed and notarized are experiencing load failures on Apple Silicon Macs.

*Tags: `apple-silicon`, `code-signing`, `debugging`, `deployment`, `macos`*

---

### Is there a reference Adobe forum post discussing plugin loading failures on macOS?

Yes, there is an Adobe Community forum post discussing plugin load failures in After Effects 25.2 and 25.3 on macOS: https://community.adobe.com/t5/after-effects-beta-discussions/my-plugins-fails-to-load-in-25-2-and-25-3-on-macos/m-p/15192946. This post documents the issue where plugins don't appear to be signed correctly, which is now a requirement on Apple Silicon Macs.

*Tags: `apple-silicon`, `debugging`, `deployment`, `macos`, `reference`*

---

### Why does SmartRender never get called in Release mode even when PreRender returns PF_Err_NONE?

SmartRender may not be called if there's a version mismatch between the SDK code used for building and the SDK headers being compiled against. In the reported case, the plugin was being built with 2025.2 SDK code but using 2023 SDK headers, which caused SmartRender to not be invoked in Release builds while Debug builds worked fine. Ensure that the SDK version used for compilation matches the SDK headers included in the project.

*Tags: `build`, `debugging`, `release`, `smart-render`*

---

### How do you debug Rust bindings for After Effects on macOS with symbols?

Use VSCode with the CodeLLDB extension. Configure a launch.json with the lldb debugger type pointing to the After Effects application bundle, and set up a preLaunchTask to build the plugin before launching. The configuration should specify the full path to the After Effects app (e.g., /Applications/Adobe After Effects 2024/Adobe After Effects 2024.app).

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build"
    }
  ]
}
```

*Tags: `debugging`, `macos`, `open-source`, `rust`*

---

### Does PF_Interrupt return a value that triggers catch blocks or just an error code?

PF_Interrupt returns an error code (PF_err), not an exception. A non-NULL PF_err does not trigger a catch block in the traditional sense—the error handling depends on how the calling code checks the error return value rather than exception-based control flow.

*Tags: `aegp`, `debugging`, `error-handling`*

---

### What is a good resource for understanding scope guards and their implementation in C++?

Alex Bizeau from maxon mentioned that scope guards are smart pointers for anything that can use lambda functions. They're valuable memory safety tools because they handle deletion automatically on throwing and return statements without requiring explicit deletion handling at every return point. Maxon has their own implementation, with examples available in their codebase.

*Tags: `debugging`, `memory`, `open-source`, `reference`, `tool`*

---

### How can you set up debugging for After Effects plugins in VSCode on macOS?

Use VSCode with the CodeLLDB extension and configure a launch.json file. Set the type to 'lldb', request to 'launch', and point the program path to the After Effects application bundle (e.g., '/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app'). Add a preLaunchTask to build the plugin before launching.

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build"
    }
  ]
}
```

*Tags: `build`, `debugging`, `macos`*

---

### Why does launching After Effects 2024/2025 from LLDB with certain launch configurations cause crashes?

The crash was traced to the launch.json configuration. Adding the line `"sourceLanguages": ["rust"]` to the launch.json prevents the crash from occurring when launching AE through LLDB. The exact mechanism is unclear, but this configuration change resolved the issue that was happening with both the user's and @fad's basic launch/tasks.json setup, even when mediacore was empty.

```cpp
"sourceLanguages": ["rust"]
```

*Tags: `build`, `debugging`, `macos`, `windows`*

---

### What is a command ID tool for After Effects plugin development?

Justin shared a command ID tool that helps developers work with After Effects commands. The tool was highlighted in a recent LinkedIn post by Justin and is useful for plugin development workflows.

*Tags: `aegp`, `debugging`, `reference`, `tool`*

---

### What should be considered when mapping After Effects command IDs to avoid conflicts?

When creating mappings of AE command IDs, avoid treating keys as unique since a single command name can map to multiple IDs with different meanings. For example, 'undo' maps to both ID 16 (the actual undo command) and ID 2371 (clear undo), but only one will display. Ensure the correct ID is shown for each command's actual function.

*Tags: `aegp`, `debugging`, `reference`, `scripting`*

---

### Are there duplicate command IDs in After Effects command ID mappings that developers should be aware of?

Yes, there are duplicate command IDs in After Effects command ID mappings. For example, 'undo' appears as both ID 16 and ID 2371 in JSON files, but they represent different commands—ID 16 is the actual 'undo' command while ID 2371 is 'clear undo'. Developers should avoid treating keys as unique and should verify which ID corresponds to the intended command functionality.

*Tags: `aegp`, `debugging`, `reference`*

---

### How can you reliably get the current After Effects or Premiere version number (e.g., 2025)?

James Whiffin noted that in_data->version major and minor fields are not being reliably updated between versions like 2024 and 2025, returning the same values. Jonah reported that accessing the version appears to work on macOS After Effects, though behavior on Premiere and Windows still needs verification.

*Tags: `aegp`, `cross-platform`, `debugging`, `macos`, `premiere`, `windows`*

---

### What causes error code 1397908844 when rendering with multiple effects in After Effects?

According to an Adobe Community post (https://community.adobe.com/t5/after-effects-discussions/error-while-rendering-with-multiple-effects-with-code-1397908844-using-pf-newworld-pf-disposeworld/td-p/14285656), this error occurs when a plugin is accidentally double-releasing a suite. The error has been documented since at least 2007 and can also occur in other Adobe applications like Illustrator.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

### What is a modern approach to error handling in After Effects SDK development?

Use std::expected for function returns instead of traditional error codes. This allows for easier error handling and proper chaining with .then() methods. Create type aliases like ExpectedAEGP and ExpectedPF to wrap the expected types for different AE APIs. Additionally, wrap all AE handles in classes with proper construction and destruction in smart pointers, and implement an allocation class manager that clears allocations when exiting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug builds, implement a cache validator to detect extra memory still allocated at endpoints.

```cpp
fn get_thing() -> Result<Thing, Error>;
```

*Tags: `aegp`, `debugging`, `error handling`, `memory`, `resource management`, `smart pointers`*

---

### How can you implement custom breakpoint assertions in C++ for After Effects plugin development without crashing the program?

Create custom assertion macros that automatically trigger breakpoints only when a debugger is attached, rather than using standard assertions that crash the program. This can be implemented using preprocessor macros with platform-specific code paths for Windows and macOS. While C++26 will have built-in support for this feature, it can be implemented today using inline assembly calls with preprocessor directives.

*Tags: `build`, `cross-platform`, `debugging`, `macos`, `windows`*

---

### What best practices should be followed for managing After Effects handles in C++ plugins?

Wrap all AE handles into classes with proper construction and destruction in destructors, and use smart pointers to manage them. Additionally, implement a greater allocation class manager that clears allocations when quitting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug mode, use a cache validator to verify no extra memory remains allocated at endpoints.

*Tags: `aegp`, `best-practices`, `debugging`, `memory`, `smart-pointers`*

---

### How do you debug ExtendScript for After Effects on macOS?

ExtendScript debugging on macOS has traditionally been difficult, requiring the Intel version which is slow. However, Adobe recently updated the debugger (about a month ago) to be Apple Silicon native, which is a significant quality of life improvement for macOS developers.

*Tags: `apple-silicon`, `debugging`, `macos`, `scripting`*

---

### How should autoKernType be set to read kerning property values in After Effects text layers?

The autoKernType property needs to be set to either 'none' or 'undefined' in order to read the kerning property. When autoKernType is set to 'metric' or 'optical' (which means automatic by the host), the kerning property may not read or write as expected. Manual kerning is indicated by setting autoKernType to 'undefined', but this behavior appears to have issues when keyframes are involved in AE 2024.3.

*Tags: `debugging`, `params`, `scripting`*

---

### Why does a blur function work differently between 8-bit and 16/32-bit color depths in After Effects?

When implementing blur effects that support multiple bit depths (8-bit, 16-bit, and 32-bit), subtle differences in how path values are converted between depth formats can cause discrepancies. One workaround is to precompose the layer with the effect first, which ensures consistent processing. The issue likely stems from improper conversion of path values when switching between 8-bit and 16/32-bit processing paths.

*Tags: `debugging`, `memory`, `mfr`, `render-loop`*

---

### What were the root causes of bugs in the FastBoxBlur implementation?

Two critical bugs were identified in FastBoxBlur: (1) Incorrect bit depth detection causing memory corruption, and (2) Improper stride handling for padded buffers. The solution involved fixing bit depth detection to use rowbytes calculation and implementing proper stride handling throughout the blur algorithm.

*Tags: `debugging`, `memory`, `output-rect`, `render-loop`*

---

### How can I debug the 'plugin wasn't installed correctly' error on Windows 11?

Use the dumpbin tool from Visual C++ to check your plugin's dependencies. Run 'dumpbin /dependents plugin.aex' in a terminal to see what dependencies are listed. If dependencies are missing, you may need to install the Microsoft Visual C++ package on the target system.

```cpp
dumpbin /dependents plugin.aex
```

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What should I do if a Rust-based After Effects plugin shows dependency errors on Windows?

Even though Rust plugins should include all crates in the final binary, Windows may still require the Microsoft Visual C++ redistributable package to be installed separately. This is a common cause of 'plugin wasn't installed correctly' errors. Ensure the target system has the appropriate Visual C++ package installed.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### How can you check if After Effects is running in headless render engine mode?

Use the AppSuite4 API to check the render engine flag. Call PF_IsRenderEngine() to determine if the renderer is running in headless mode, which is useful for plugins that need to behave differently during background rendering versus interactive use.

```cpp
PF_Boolean bIsRenderEngine = true;
suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine);
```

*Tags: `aegp`, `api`, `debugging`, `render-loop`*

---

### Is DirectX rendering currently implemented in After Effects production, and how can it be enabled for plugin development?

DirectX rendering support was added to the After Effects SDK with an official flag, though it appears to be in early stages. Adobe replaced the UI from OpenGL to DX12 in 2021. To enable DirectX compute for debugging and testing, use the 'forcedxcompute' setting in the After Effects console panel under settings, but note that a full computer restart (not just AE restart) is required for the setting to take effect. The feature is likely being developed primarily for Windows on ARM device support, as those devices have first-party support for DX12. Adobe has not yet officially supported Vulkan exposure to plugins.

*Tags: `apple-silicon`, `debugging`, `directx`, `gpu`, `windows`*

---

### Should developers use the After Effects GPU Suite for plugin development?

No, the AE GPU Suite should be avoided for plugin development. Instead, developers should implement custom GPU renderer backends that support Metal, DirectX, and OpenGL with proper GPU buffer interoperability, as demonstrated by modern plugin architectures like Red Giant Universe's approach.

*Tags: `best-practices`, `debugging`, `gpu`, `metal`, `opengl`*

---

### Why does a plugin work in the Plugins folder but not the Mediacore folder on Mac After Effects 26.0.0?

Multiple developers reported this issue with After Effects 26.0.0 on Mac where plugins load correctly from the Plugins folder but fail to load from the Mediacore folder. The Gaussian Splat team issued an update to address AE 26 compatibility, suggesting there are specific compatibility changes in AE 26 affecting Mediacore plugin loading on Mac. The issue also affects Premiere Pro 2026, where plugins in Mediacore don't appear at all, even though they work in earlier Premiere versions (2023-2025) and in AE 2026.

*Tags: `cross-platform`, `debugging`, `deployment`, `macos`, `mfr`*

---

### How do 3D Camera Tracker and Warp Stabilizer output progress and live update without purging cache?

According to an After Effects developer, these built-in effects use internal APIs that are not available to plugin developers. The effects "cheat" and do not operate within the constraints of a normal AE Effect API plug-in. Developers have attempted workarounds involving refresh kicks, but these have proven unreliable across macOS and Windows. A feature request was made in 2023 to expose this capability to third-party developers, but it remains unavailable through the public API.

*Tags: `aegp`, `caching`, `cross-platform`, `debugging`, `render-loop`, `smart-render`*

---

### Why are plugins not loading in After Effects 2026?

There is a known issue on Mac where plugins located in the Mediacore folder are not recognized by AE2026 and must be placed in After Effects' main plugin folder instead. This appears to be one of several compatibility issues affecting plugin loading in AE2026.

*Tags: `debugging`, `deployment`, `macos`, `pipl`*

---

### Why does ImGui fail to initialize OpenGL loader on macOS?

The issue was related to window flag initialization order. On macOS, window flags need to be set before window creation, not after. This differs from Windows behavior where setting flags after creation may work. Ensure all window configuration flags are applied before creating the window.

*Tags: `debugging`, `macos`, `opengl`, `ui`*

---

### Why is rowbytes alignment important for pointer dereferencing in After Effects plugins?

Dereferencing a pointer that is not correctly aligned for the referenced type results in undefined behavior according to the C specification (6.3.2.3 7 in the C spec https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3047.pdf). If rowbytes was not a multiple of alignof(Pixel Type), you would be dereferencing misaligned pointers everywhere, which is UB. Apple documents this issue at https://developer.apple.com/documentation/xcode/misaligned-pointer.

*Tags: `aegp`, `debugging`, `macos`, `memory`*

---

### Is there documentation available for debugging plugins in DaVinci Resolve?

Alex Bizeau from Maxon mentions there is official documentation available for debugging OpenFX plugins in DaVinci Resolve that can be helpful for developers working with that host.

*Tags: `debugging`, `openfx`, `reference`, `resolve`*

---

### Why were dropdown menus in a transition plugin malfunctioning during debugging?

The issue occurred when opening a Premiere project that already had the transition plugin applied. Opening a fresh instance of the plugin and applying it newly resolved the dropdown malfunction. The problem was related to the state of the plugin when loaded from an existing project versus a fresh application.

*Tags: `debugging`, `params`, `premiere`, `ui`*

---

### How do you debug a Premiere plugin from Xcode when symbols aren't loading and breakpoints don't work?

Ensure that symbol generation is enabled in your build settings. Set 'Generate Symbols' to 'Yes' in your Xcode scheme. If this setting is accidentally set to 'No', symbols won't be generated and debuggers won't be able to load them, preventing breakpoints from working properly.

*Tags: `build`, `debugging`, `premiere`, `xcode`*

---

### Why does Premiere pass NULL for input_world in Render when compiled with fast optimizations, but not in After Effects?

A developer reported that when compiling a plugin with fast optimizations enabled, Premiere passes input_world = NULL in the Render call, whereas without optimizations the data is present. The same code does not exhibit this problem in After Effects. This suggests the issue may be related to optimization levels affecting how Premiere handles or passes buffer data to plugins during rendering.

*Tags: `build`, `debugging`, `premiere`, `render-loop`, `smartfx`*

---

### How can you prevent the compiler from optimizing a render function that is causing issues in an After Effects plugin?

Add the `__attribute__ ((optnone))` compiler attribute before the render function declaration. This tells the compiler to skip optimization for that specific function, even if the rest of the plugin is optimized. This was identified as a workaround by the community, though it may indicate an underlying issue worth investigating further.

```cpp
__attribute__ ((optnone))
void render() {
  // render function implementation
}
```

*Tags: `build`, `debugging`, `optimization`, `render-loop`*

---

### How should ARGB_8u be defined in globalSetup for proper colorspace handling?

When using ARGB_8u colorspace, it needs to be properly defined in the globalSetup function. The user inquired about the correct definition method and suggested checking colorspace settings and adding null checks for inoutworld (if inoutworld is null return err) to see if Premiere will redo the render.

*Tags: `debugging`, `params`, `premiere`, `render-loop`*

---

### Why does my plugin report 'Not able to acquire AEFX Suite' when playing in Premiere?

This error typically occurs during playback/render in Premiere and may be caused by calling AEFX Suite functions in a shared thread or in code paths that execute in Premiere context. Check if you have conditional logic like `if (app_id != 'PrMr') => call AEFX_suite` where the app_id detection may be returning an incorrect value in Premiere, causing AEFX Suite calls to execute when they shouldn't. Additionally, Premiere can have quirks with certain suites (similar to known colorspace suite issues), so verify your host application detection logic and ensure AEFX Suite calls are not made from Premiere contexts.

*Tags: `aegp`, `debugging`, `premiere`, `threading`*

---

### How can I avoid noise artifacts when After Effects automatically converts 16/32-bit project input to 8-bit for my plugin?

Rather than relying on After Effects' automatic conversion, you should modify your plugin to accept 16 and 32 bits per channel (bpc) inputs and perform the conversion to 8-bit yourself. This approach provides several benefits: users will benefit from having the output remain in 16/32 bpc, you will avoid the warning sign next to your plugin in the UI, and you'll have full control over the conversion process to avoid dithering artifacts introduced by AE's automatic conversion.

*Tags: `aegp`, `debugging`, `output-rect`, `params`*

---

### How can I hide an effect parameter from both the effect window and timeline window?

Set the AEGP_DynStreamFlag_HIDDEN flag during the PF_Cmd_UPDATE_PARAMS_UI command. This command is called when a project loads and the effect is shown for the first time, ensuring hidden parameters remain hidden across project loads and new effect applications. Note: There is a known bug in After Effects where owner-drawn portions of hidden parameters may still display space in certain circumstances (e.g., when applying a new effect instance), though the stream itself will be correctly hidden.

*Tags: `aegp`, `debugging`, `params`, `ui`*

---

### How can I invoke an AEGP from ExtendScript without creating a menu item?

There are two main approaches: (1) Use AEGP_Menu_NONE with InsertMenuCommand to avoid creating a visible menu, though this may trigger an alert sound on startup. (2) Set a flag in the JavaScript global scope and have the AEGP check for that flag on idle_hook calls, which occur 20-50 times per second. This provides asynchronous communication without menu creation. Alternatively, use a C external object to bridge ExtendScript and AEGP directly, though this requires more complex setup.

*Tags: `aegp`, `debugging`, `scripting`*

---

### What is the correct first argument to pass to transform_world and other suite callbacks in After Effects plugins?

According to community expert Shachar Carmi, the Adobe documentation is incorrect about the first argument. Instead of passing in_data as documented, you should pass NULL or in_data->effect_ref. Passing NULL works because many suite callbacks accept null instead of effect_ref, and the PF_INTERRUPT macro internally uses effect_ref to make interrupt checks. The incorrect documentation may be legacy from before CC2015 when a separate rendering thread was introduced.

*Tags: `debugging`, `params`, `reference`, `sdk`, `transform_world`*

---

### How does the AddArc function work when drawing multiple circles in After Effects UI?

The AddArc function treats all drawing operations as a continuous path. When drawing multiple circles, you must call close() after each AddArc to properly close that shape before starting a new one. Without closing, the path continues from the end point of one circle back to the beginning of the next, creating unexpected intersections. Alternatively, you can draw each shape in separate paths with separate drawing calls. The center argument for AddArc should use absolute position coordinates.

```cpp
DRAWBOT_PointF32 center1 = { drawRectF.left + point.x , drawRectF.top + drawRectF.height - point.y };
suites.PathSuiteCurrent()->AddArc(plotPath, &center1, 4.0f, 0.0f, 360.0f);
suites.PathSuiteCurrent()->Close(plotPath);
DRAWBOT_PointF32 center2 = { drawRectF.left + point.x + 100 , drawRectF.top + drawRectF.height - point.y };
suites.PathSuiteCurrent()->AddArc(plotPath, &center2, 4.0f, 0.0f, 360.0f);
suites.PathSuiteCurrent()->Close(plotPath);
```

*Tags: `debugging`, `drawbot`, `path`, `ui`*

---

### Why does AEGP_GetLayerNumMasks return 0 when a mask is clearly set on the layer in After Effects?

The issue may stem from confusion between track mattes and masks. Track mattes are different from masks—a track matte uses the alpha or luma of one layer to affect another layer, while a mask is a vector shape drawn on a layer with the pen tool. If you're trying to use the Mask Suite on a layer that only has a track matte applied, AEGP_GetLayerNumMasks will correctly return 0 because there are no actual masks on that layer. Additionally, verify that the layer handle is valid and check the error value returned by AEGP_GetLayerNumMasks to diagnose the root cause.

```cpp
A_long numMasks = 0;
RECORD_ERROR(suites.MaskSuite6()->AEGP_GetLayerNumMasks(layerHandle, &numMasks));
for (int i = 0; i < numMasks; i++) {
  AEGP_MaskRefH maskHandle;
  RECORD_ERROR(suites.MaskSuite6()->AEGP_GetLayerMaskByIndex(layerHandle, i, &maskHandle));
  auto mask = ExportMask(context, maskHandle);
  masks.push_back(mask);
  RECORD_ERROR(suites.MaskSuite6()->AEGP_DisposeMask(maskHandle));
}
```

*Tags: `aegp`, `debugging`, `macos`, `mask`*

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

### Why doesn't AEGP_EffectCallGeneric() trigger PF_Cmd_COMPLETELY_GENERAL on Mac when trying to identify a specific effect instance?

According to community expert shachar carmi, AEGP_EffectCallGeneric() has limitations and cannot reliably call from one effect instance to another instance of the same type. The expert suggests that if you need inter-instance communication, you should create a separate AEGP plugin to handle these calls during idle processing, rather than calling directly between effect instances of the same type. As an alternative for identification that doesn't need to survive save/load, you can change a hidden parameter's name to be the identifier and read it via the stream suite.

```cpp
// Instead of calling between same-type effects, use a separate AEGP
// Or use hidden parameter name as identifier via stream suite
AEGP_SetStreamName()
```

*Tags: `aegp`, `cross-platform`, `debugging`, `macos`, `params`, `windows`*

---

### Why are global variables considered bad practice in plugin development?

Global variables are discouraged for several reasons: (1) They reduce code readability because function behavior depends on data outside their parameters; (2) They can cause CPU cache inefficiency as the processor must fetch data from different memory segments; (3) In multi-threaded scenarios, they require mutexes which impose performance penalties and make code non-thread-safe; (4) They are generally considered poor programming practice. However, they are widely used in practice, especially for constant data that is set once and never changes.

*Tags: `debugging`, `memory`, `threading`*

---

### How do you debug a plugin when it's loaded through After Effects Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible After Effects instance for rendering, separate from the main AE process. Breakpoints set in the main application won't be triggered during AME rendering. To debug in this scenario, you need to attach your debugger to the already-running subprocess. On Windows, use the Visual Studio debugger's attach-to-process feature: https://learn.microsoft.com/en-us/visualstudio/debugger/attach-to-running-processes-with-the-visual-studio-debugger?view=vs-2022. You can also observe these processes in Windows Task Manager.

*Tags: `debugging`, `deployment`, `media-encoder`, `windows`*

---

### Why does my map of effect matchnames to install keys return incorrect install keys when applying effects?

The issue was that the map was declared as `std::map<A_char, A_long>`, storing only a single character instead of the full matchname string. The map should be declared with a string type (e.g., `std::map<std::string, A_long>` or `std::map<A_char*, A_long>`) to properly store and look up the complete effect matchname. When accessing the map with `InstallKeys::keys[*myMatchName]`, only the first character of the matchname was being used as the key, causing incorrect lookups.

```cpp
// Incorrect:
std::map<A_char, A_long> keys;
InstallKeys::keys[*nextMatchName] = nextKey;

// Correct:
std::map<std::string, A_long> keys;
InstallKeys::keys[std::string(nextMatchName)] = nextKey;
```

*Tags: `aegp`, `c++`, `debugging`, `plugin-development`*

---

### Is it possible to hot reload After Effects plugins without restarting the application?

No, hot reloading is not practical for native C++ After Effects plugins. Each time you build C++ code, you must restart After Effects because the application calls parameter setup only once per session when the plugin is first loaded, so hot reloading cannot trigger re-initialization of parameters. However, ExtendScript and CEP plugins can be reloaded without restarting the entire AE application. Debugging through an IDE like Microsoft Visual Studio or Apple Xcode can make the restart cycle faster than manually launching AE.

*Tags: `build`, `debugging`, `params`, `scripting`, `sdk`*

---

### What resources are available for implementing hot reloading in Xcode development?

For Xcode development, refer to the Stack Overflow discussion on instant run or hot reloading: https://stackoverflow.com/questions/42529081/instant-run-or-hot-reloading-for-xcode. Note that hot reloading was more common in 32-bit builds but became less practical with the transition to 64-bit architecture. Limitations exist when hot reloading is used with After Effects plugins, particularly around parameter setup.

*Tags: `build`, `debugging`, `macos`, `reference`, `tool`*

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

### How can you force rendering from a separate thread in After Effects plugins?

You cannot directly set out_flags from a separate thread. Instead, use the idle hook, which fires 30-50 times per second on the main UI thread. From the idle hook, you can: (1) change an invisible parameter value to force re-render, (2) call command numbers to purge cache, or (3) use AEGP_EffectCallGeneric to have the effect instance act on itself. The idle hook is the recommended approach for syncing work from separate threads back to the main thread.

```cpp
// In idle hook (main thread)
// Change invisible param to trigger re-render
out_data->out_flags |= PF_OutFlag_REFRESH_UI | PF_OutFlag_FORCE_RERENDER;
```

*Tags: `aegp`, `debugging`, `render-loop`, `threading`, `ui`*

---

### How should I handle memory allocation when rendering multiple frames with NON_PARAM_VARY flag?

When using PF_OutFlag_NON_PARAM_VARY to render multiple frames, you must implement PF_Cmd_FRAME_SETDOWN to properly free memory allocated during PF_Cmd_FRAME_SETUP. This prevents memory accumulation as FrameSetup and FrameSetDown are called for each frame. Without implementing FRAME_SETDOWN, allocated memory from each frame setup will accumulate rather than being released between frames.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

### Where can you find documentation and definitions for After Effects SDK data types like A_Time and A_FpLong?

Most AE SDK types are not documented in the PDF reference. The best approach is to examine the SDK header files directly. In your IDE, right-click on any AE type and select 'Go to Definition' to jump to the header where the type is defined. For example, Pf_fp_long is defined as a double. The main documentation resource is https://ae-plugins.docsforadobe.dev/index.html, though it may have gaps.

*Tags: `aegp`, `debugging`, `reference`*

---

### What causes a custom arbitrary parameter to display as <error> in the timeline?

The <error> message occurs when an arbitrary parameter doesn't have a proper name assigned to it. Ensure that the custom arb param is given a descriptive name string in the parameter definition.

```cpp
PF_ADD_ARBITRARY2("Banner Name",
  A_long(PF_FpLong(BANNER_WIDTH) * 0.5),
  A_long(PF_FpLong(BANNER_HEIGHT) * 1.0),
  PF_ParamFlag_CANNOT_TIME_VARY,
  PF_PUI_CONTROL,
  def.u.arb_d.dephault,
  BANNER_ID,
  0);
```

*Tags: `arb-data`, `debugging`, `params`, `ui`*

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

### How can a C++ plugin read the time slider position when its own UI window is displayed and plugin parameters haven't changed?

Use the AEGP suite to query the current time directly instead of relying on in_data->current_time, which only updates when EffectMain commands are triggered. Call AEGP_GetActiveItem() to get the active item, then use AEGP_GetItemCurrentTime() to retrieve the current time value. Alternatively, use an idle hook on the plugin side combined with a timer on the window side to synchronize playback between the AE timeline and the custom win32 window.

```cpp
A_long time_ae(PF_InData * in_data)
{
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  A_Time time;
  AEGP_ItemH myitem;
  suites.ItemSuite8()->AEGP_GetActiveItem(&myitem);
  suites.ItemSuite8()->AEGP_GetItemCurrentTime(myitem, &time);
  return time.value;
}
```

*Tags: `aegp`, `debugging`, `params`, `ui`, `windows`*

---

### What causes an access violation error when calling AEGP_GetActiveItem() in a C++ plugin?

An access violation when calling AEGP_GetActiveItem() is most likely caused by an invalid in_data pointer or an invalid pica_basicP passed to the AEGP_SuiteHandler. Ensure that the function is being called during a valid AE command execution where in_data is guaranteed to be valid, and verify that the passed in_data pointer has not been corrupted or used outside its valid scope.

*Tags: `aegp`, `debugging`, `memory`*

---

### Why does memory consumption increase drastically when playing an After Effects plugin and not return to normal after removing the effect layers?

The memory increase is likely not a leak but rather After Effects caching rendered frames. To verify this, purge RAM after playing or previewing the composition—if memory returns to the pre-play level after purging, it confirms the memory was being held by the cache, not leaked by the plugin. If a parameter change should invalidate cached frames but doesn't, you can force invalidation by changing the value of an invisible parameter.

*Tags: `caching`, `debugging`, `memory`, `render-loop`*

---

### How do you properly handle undo/redo with sequence data and arbitrary data in After Effects plugins?

Sequence data does not participate in undo/redo, which makes it problematic as a base for calculations that depend on parameter changes. The recommended approach is to store a state identifier in both arbitrary data (arb data) and sequence data. When using sequence data, compare the state identifier in the arb to the one in the sequence data—if they mismatch, regenerate the sequence data and store the corresponding state identifier. Sequence data is best used only when other storage methods don't meet needs, such as for data that should survive a reset button, data that shouldn't trigger undo entries, or temporary data passed between modules.

*Tags: `arb-data`, `debugging`, `params`, `sequence-data`*

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

### Why is my After Effects plugin not showing up in the Effects menu after building it?

The plugin's build output location must be in a directory where After Effects looks for plugins. AE loads plugins from the Plug-ins folder in its installation directory, searching up to 5 folders deep. Either build directly into a subfolder of AE's Plug-ins directory, or create a shortcut in that directory pointing to your build output. Using a shortcut is recommended as it allows testing across multiple AE versions without updating build preferences when installing new AE versions.

*Tags: `build`, `debugging`, `deployment`, `macos`, `windows`*

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

### Why does my After Effects plugin fail to load with Error 126 when launched directly, but works when loaded from Visual Studio?

Error 126 when loading an AE plugin typically indicates a dependency issue with missing DLLs. When running the plugin from Visual Studio's debugger, the debugger's environment may provide DLL paths that aren't available when After Effects is launched directly. To diagnose the issue, build a simple test plugin that loads correctly in both contexts and have it report its home directory path. If testing on the same machine, the problem is likely a difference in the process's working directory or DLL search paths between debugger and direct execution. Ensure all dependent DLLs (like those required by the cairo library) are either in the plugin directory, in a standard system path, or that your plugin's search paths are properly configured for non-debugger execution.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### Where did the 'About' button go for plugins in After Effects 2020?

The 'About' button was deliberately hidden in After Effects 2020. It is still accessible by right-clicking the effect name and choosing 'About' from the context menu. As an alternative, developers can use PF_SetOptionsButtonName() to implement an options button, which will be displayed next to the 'Reset' button.

*Tags: `debugging`, `macos`, `pipl`, `ui`*

---

### Why doesn't an After Effects plugin receive PF_Event_KEYDOWN when selected in the Effects Controls window?

The plugin needs to have at least one parameter with an ECW (Effects Controls Window) custom UI exposed and untwirled (not collapsed) to receive keyboard events. As a workaround, adding an invisible ARB parameter with custom UI can also fix the issue.

*Tags: `aegp`, `debugging`, `params`, `ui`*

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

### How can I find the command ID for Numpad+0 (play preview from start) in After Effects?

Setup a command hook with the 'all commands' flag. Once you've made some preliminary calls, you can press 0 and catch the command ID that is triggered. This method works because the Numpad+0 command doesn't have a corresponding menu item, so app.findMenuCommandId() won't work directly.

*Tags: `aegp`, `debugging`, `scripting`, `ui`*

---

### How can I debug intermittent C++ exceptions that occur between PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_RENDER?

According to community expert shachar carmi, intermittent exceptions at the same memory address are likely caused by stack overflow or buffer overrun. The recommended debugging strategy is to use binary search: disable half your code and test to see if the problem manifests, then progressively narrow down the suspect code by testing the remaining halves. This methodical approach helps isolate the problematic section without relying on traditional debuggers, which may not be effective for these types of memory issues.

*Tags: `debugging`, `memory`, `render-loop`, `windows`*

---

### Does using AEGP_SetSelection force a render, and how can I avoid it?

AEGP_SetSelection should not force a render by itself. If you're experiencing unwanted renders when calling this function, it's likely due to another plugin in the composition that relies on collection state as part of its pre-render GUID, or a bug in your own code (such as missing break statements in case statements). The unexpected render behavior is probably not caused by AEGP_SetSelection directly, but by dependent plugins or logic that responds to the selection change.

*Tags: `aegp`, `debugging`, `render-loop`*

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

### How can I detect when the user performs undo or redo actions in my After Effects effect plugin so I can update my custom UI?

One effective approach is to save a random number or state identifier with your data, then poll the effect data (stored in a data parameter) once or twice per second to check if the state index differs from what your UI window expects. If there's a mismatch, the data has changed via undo/redo or project load. To optimize performance, you can save the random number to a separate parameter like a slider. When saving data, use StartUndoGroup() and EndUndoGroup() to bundle operations into a single undo entry. This technique works well for effects with custom external UIs that manage internal parameters in a structure, which is then serialized to an AEGP data parameter using AEGP_SetStreamValue().

*Tags: `aegp`, `debugging`, `params`, `redo`, `ui`, `undo`*

---

### How should you diagnose a plugin crash that only occurs in After Effects CC 2015+ but not CC 2014, particularly when it involves custom UI?

Ask the user to send over their old After Effects preferences folder and then delete the pref folder before relaunching AE. If resetting the preferences solves the issue, try reproducing the issue on your machine using the user's old preferences. This approach has a high probability of resolving the issue. If successful, the old prefs can help identify what setting or cached data was causing the crash.

*Tags: `debugging`, `macos`, `preferences`, `ui`*

---

### Can After Effects plugins be compiled in Release mode, and what are the runtime linking requirements?

Yes, After Effects plugins should be compiled in Release mode for both performance and deployment ease. There are four C++ runtime linking options: MT (multithreaded static), MD (multithreaded DLL), MTd (multithreaded debug static), and MDd (multithreaded debug DLL). Adobe recommends using MD or MDd because there is a limit to how many MT-compiled plugins After Effects can load simultaneously. In Release mode, use either MD or MT; in Debug mode, use either MDd or MTd. There is no noticeable performance difference between MD and MT variants.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### Why won't my After Effects plugin compile in Release mode when using a third-party library compiled in Release?

If a plugin won't compile in Release mode, check the project settings for rogue '_DEBUG' macro definitions in the Preprocessor Definitions. Also inspect individual file preferences, as some files may have their own compile settings and additional macros that force Debug mode compilation. Remove any '_DEBUG' definitions that shouldn't be present in Release builds.

*Tags: `build`, `debugging`, `windows`*

---

### Why does adding keyframes from a modeless dialog fail in After Effects CC2015 and later?

Since After Effects 2015, there is a known issue with calling AE's suites while a modal loop is running. To work around this, retrieve the information you need from After Effects before running the modal loop, rather than attempting to call AE suites from within the modal dialog.

*Tags: `aegp`, `cross-platform`, `debugging`, `ui`*

---

### Why does AEGP_ExecuteScript cause a crash on macOS but not Windows when closing After Effects?

The crash is likely caused by something in the script itself, not the AEGP_ExecuteScript function. Use binary search debugging: delete half of your script and run again, repeating until the error disappears to isolate the problematic code. In the reported case, the issue was caused by using dlg.close() instead of dlg.hide() on macOS, which behaves differently between platforms.

*Tags: `aegp`, `cross-platform`, `debugging`, `macos`, `scripting`*

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

### How do you set AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 values in the PiPL.r file?

To set these flags correctly, place a breakpoint in the plugin's global setup function and observe the numerical values being applied to the outflags and outflags2 variables. Copy these exact numerical values to the corresponding fields in your PiPL.r file. The values must match what your plugin sets during initialization, or a mismatch error will occur at plugin launch. After making changes, perform a clean rebuild to ensure the pipl_tool regenerates the .rc file from your updated .r file.

*Tags: `build`, `debugging`, `params`, `pipl`*

---

### Why doesn't PF_Event_IDLE run after restoring a project until a control is touched?

PF_Event_IDLE is only called when an effect instance is the sole selected item. When you apply an effect from the menu, it automatically becomes selected alone, so idle events trigger. However, when loading a project, effects are not automatically selected, so idle events don't fire until a control is manually touched. This is expected behavior and cannot be overridden.

*Tags: `aegp`, `debugging`, `params`, `ui`*

---

### How can I handle UI synchronization tasks that need to run even when an effect isn't selected after project restore?

Use AEGP_RegisterIdleHook() to register a general-purpose idle hook that receives regular calls not tied to a specific effect instance. During these calls, you can scan the current comp or entire project for your effect instances using AEGP_GetActiveItem(), AEGP_GetFirstProjItem(), AEGP_GetNextProjItem(), AEGP_GetCompFromItem(), AEGP_GetCompLayerByIndex(), AEGP_GetLayerEffectByIndex(), and AEGP_GetInstalledKeyFromLayerEffect() to identify your effects. You can then programmatically select your effect using AEGP_NewCollection(), AEGP_CollectionPushBack(), and AEGP_SetSelection() so your regular effect process picks it up.

*Tags: `aegp`, `debugging`, `params`, `ui`*

---

### How can I debug iterate_generic functions properly in Visual Studio without erratic step-over behavior?

When debugging iterate_generic functions in Visual Studio, hit the breakpoint for the first time, then disable the breakpoint before stepping line by line. This prevents the debugger from jumping between threads (BEE WorkQueue and system DLLs) and allows normal line-by-line stepping.

*Tags: `debugging`, `iterate-generic`, `threading`, `windows`*

---

### What is a good reference for dynamically modifying popup parameter strings in After Effects plugins?

Adobe's forum thread at https://forums.adobe.com/message/2446142#2446142 contains detailed explanation on how to modify popup entry names and manage hidden popups for dynamic parameter changes. Additionally, the thread 'Re: Questions before writing my plugin' discusses similar techniques for modifying parameters during ParamChange() and UI() calls.

*Tags: `debugging`, `params`, `reference`, `ui`*

---

### How can a plugin detect when a RAM preview starts in After Effects?

Use command_hook with command number 2285 to detect when a RAM preview begins. Command 2415 can be used to detect Play (spacebar) events. Note that PF_Cmd_RENDER may not be called for cached frames during RAM preview.

*Tags: `aegp`, `caching`, `debugging`, `render-loop`*

---

### What does the PF_OutFlag2_SUPPORTS_QUERY_DYNAMIC_FLAGS error mean and how do you fix it?

The error 'effect cannot change non-dynamic flag bits during PF_Cmd_QUERY_DYNAMIC_FLAGS' occurs when there's a mismatch between the PiPL (Plug-In Property List) resource and the PF_Cmd_GLOBAL_SETUP implementation. The PF_OutFlag2_SUPPORTS_QUERY_DYNAMIC_FLAGS flag must be set consistently in both the PiPL and during PF_Cmd_GLOBAL_SETUP. On macOS, you can edit PiPL resources using Resorcerer with the PiPL template from the SDK, or use DeRez to create a .r file. The behaviors indicated during global setup must match those in the PiPL exactly. On Windows, ensure the .rc file is regenerated by cleaning and rebuilding the project.

*Tags: `build`, `debugging`, `macos`, `pipl`, `windows`*

---

### How should you debug a plugin that stops rendering prematurely when After Effects runs out of memory?

When a plugin doesn't properly handle out-of-memory conditions, first clarify whether it crashes or just stops rendering. If you cannot reproduce the problem in a debug session, write data to file from strategic points in the code and reproduce the error in release mode. This file-based logging approach will help pinpoint where memory is not being freed properly.

*Tags: `ae-plugin`, `debugging`, `memory`*

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

### Why does my PF_Cmd_DO_DIALOG case handler run constantly instead of only when the dialog button is clicked?

The issue is likely a missing break statement on the case above PF_Cmd_DO_DIALOG. Without a break, execution falls through to the dialog case, causing it to run repeatedly. Ensure each case statement in your switch block has a proper break statement to prevent fall-through execution.

```cpp
case PF_Cmd_DO_DIALOG:
  err = Register(in_data, out_data);
  break;
```

*Tags: `debugging`, `sdk`, `ui`*

---

### How do you debug an After Effects plugin in Xcode?

Set the Xcode executable path to the After Effects application, then hit Cmd+R to build and run a debug session. Make sure the build location is the same place from which AE loads the plugin—you can create an alias from AE's plugins directory to the build location, and AE will scan the alias. Note that param and global setup are only done when you first apply an effect to a layer, so changes to those require restarting AE. Quick fix/continue functionality was removed in the transition to x64 and did not return.

*Tags: `build`, `debugging`, `macos`*

---

### Can you keep After Effects open between debug builds when developing a plugin?

Parameter and global setup are only performed when you first apply an effect to a layer and are shared for all instances throughout the session, so if you change code in those areas you must restart AE. The quick fix/continue feature that once allowed code changes while running was removed during the transition to x64 architecture.

*Tags: `build`, `debugging`*

---

### How do I use PF_SPRINTF to format messages in an After Effects plugin?

PF_SPRINTF is a macro that expects the same arguments as the standard C sprintf() function. It expands to (*in_data->utils->ansi.sprintf)(args...). The function takes a buffer (like out_data->return_msg) as the first argument, followed by a format string and any values to format. Do not pass in_data or other non-format arguments where sprintf() expects a format string. The macro must be used in a context where the in_data variable is available.

```cpp
PF_SPRINTF(out_data->return_msg, "test hello");
PF_SPRINTF(out_data->return_msg, "value: %d", some_int);
```

*Tags: `debugging`, `params`, `reference`, `ui`*

---

### How do you set up debugging for an After Effects SDK example project in Visual Studio?

In the project properties window, set the command for debugging by providing the path to the afterfx.exe file. You can ignore any symbols-related warnings that appear.

*Tags: `build`, `debugging`, `sdk`, `windows`*

---

### How do you output the current frame number being rendered in an After Effects plugin using printf?

When accessing the current frame in a PF_Cmd_SMART_RENDER command, use the correct printf format specifier. The in_data->current_time field is an integer, so use %i instead of %s. Using %s (string format) will cause undefined behavior, including null values, blank output, and corrupted data like random ASCII characters appearing in the output.

```cpp
printf("\nRendering frame %i", in_data->current_time);
```

*Tags: `aegp`, `debugging`, `smart-render`*

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

### What causes an 'invalid filter' error when loading a compiled After Effects plugin?

The invalid filter error is usually a dependency issue. Check your plug-in using Dependency Walker to verify all required components are present on your machine. This error also commonly occurs when you compile your plug-in in Debug mode and try to run it on a non-development machine where the debug DLLs don't exist. Compiling in Release mode resolves this issue.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### How can I validate image files before importing them into After Effects through a plugin?

There are two approaches: (1) Check the file using OS tools—both GDI+ (Windows) and Quartz (macOS) offer tools for reading image files in various formats to validate them before importing. (2) Use After Effects to check—use AEGP_StartQuietErrors to suppress user-visible errors, perform a test render to detect errors silently, then call AEGP_EndQuietErrors when done. Based on the result, you can decide whether to dispose of the image or notify the user.

*Tags: `aegp`, `debugging`, `image`, `macos`, `validation`, `windows`*

---

### Why does an After Effects effect plugin crash when dragging camera geometry values instead of typing them?

The crash occurs when keyframing masks during frameSetup after getting camera data, especially when the camera layer is positioned before the current layer in the composition hierarchy. The issue was specific to After Effects CS5.5 and disappeared in CS6 and later versions. A workaround is to check the version and deselect the camera or layer if needed when working with older versions.

```cpp
ERR(suites.PFInterfaceSuite1()->AEGP_ConvertEffectToCompTime(	in_data->effect_ref,
	in_data->current_time,
	in_data->time_scale,
	&comp_timeT));
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectCamera(	in_data->effect_ref,
	&comp_timeT,
	&camera_layerH));
if (camera_layerH){
	ERR(suites.LayerSuite6()->AEGP_GetLayerToWorldXformFromView(	camera_layerH,
		&comp_timeT,
		&comp_timeT,
		&matrix));
	AEGP_StreamVal2		camVal;
	ERR(suites.StreamSuite4()->AEGP_GetLayerStreamValue( camera_layerH, AEGP_LayerStream_ZOOM, AEGP_LTimeMode_CompTime, &comp_timeT, FALSE, &camVal, NULL));
	zoom = camVal.one_d;
}
```

*Tags: `aegp`, `camera`, `debugging`, `macos`, `windows`*

---

### How should you safely read camera data from an effect plugin to avoid crashes during parameter dragging?

Use PF_ABORT before reading camera data to check if an abort signal has been triggered. If you get an abort signal, skip reading the data. This helps prevent crashes that can occur during parameter dragging operations, particularly when dealing with composition cameras and keyframing operations.

*Tags: `aegp`, `debugging`, `params`, `render-loop`*

---

### Why does calling AEGP_ExecuteScript from within an effect plugin crash After Effects?

Executing script commands that modify the scene state (like applying a preset to a layer) from within an effect plugin causes a crash because the script invalidates locked/checked-out memory objects that are currently passed to the effect plugin via in_data. The effect is in mid-call when the scene state changes, creating memory conflicts. You cannot add an effect to a layer while your effect is currently executing. The solution is to defer scene modifications until after the effect has finished execution.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

### Why does ProjDumper throw an 'unexpected match name searched for in group' error when trying to get a layer's audio stream?

The error occurs when attempting to get an audio stream from a layer that doesn't have audio. Before calling AEGP_GetNewLayerStream with AEGP_LayerStream_AUDIO, you should first check if the layer's source item has audio by calling AEGP_GetLayerSourceItem followed by AEGP_GetItemFlags with AEGP_ItemFlag_HAS_AUDIO. Note that some layers (cameras, lights, text layers, and vector shapes) don't have a source item at all.

```cpp
AEGP_ItemH itemH(NULL);
ERR(suites.LayerSuite8()->AEGP_GetLayerSourceItem(layerH, &itemH));
if (itemH != NULL) {
  A_long flags;
  ERR(suites.ItemSuite9()->AEGP_GetItemFlags(itemH, &flags));
  if (flags & AEGP_ItemFlag_HAS_AUDIO) {
    AEGP_StreamH streamH(NULL);
    ERR(suites.StreamSuite2()->AEGP_GetNewLayerStream(S_my_id, layerH, AEGP_LayerStream_AUDIO, &streamH));
  }
}
```

*Tags: `aegp`, `debugging`, `reference`, `sdk`*

---

### How can I hide layers in a composition using AEGP without getting internal verification errors?

Use AEGP_SetDynamicStreamFlag to hide layers, but suppress errors with AEGP_StartQuietErrors() and reset the error variable (err = NULL) after each call. The ERR() macro will skip subsequent commands if an error has occurred, so resetting err is necessary to continue loop execution. Additionally, check the stream type before calling AEGP_SetDynamicStreamFlag() to ensure you're only operating on valid layer streams (AEGP_StreamType_LAYER_ID).

```cpp
ERR(suites.DynamicStreamSuite4()->AEGP_SetDynamicStreamFlag(streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, TRUE));
err = NULL;
ERR2(suites.StreamSuite4()->AEGP_DisposeStream(streamH));
```

*Tags: `aegp`, `debugging`, `layer-checkout`*

---

### What is the difference between using ERR() macro and direct error assignment in AEGP plugin code?

The ERR() macro checks if an error has already occurred (err != NULL) and skips the function call if one has. This prevents cascading errors but also skips subsequent operations. Direct assignment (err = doStuff()) executes the function regardless of prior errors. ERR() is the preferred technique as it provides better error handling flow control, but you must manually reset err = NULL if you want to continue execution after an error.

```cpp
ERR(suites.SomeFunction());
// vs
err = suites.SomeFunction();
```

*Tags: `aegp`, `debugging`, `sdk`*

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

### How do you hide and show parameter groups dynamically in After Effects plugins when a checkbox is toggled?

Use AEGP_GetNewEffectStreamByIndex to get stream references for parameters in each group, then call AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to toggle visibility. Handle this in response to PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_UPDATE_PARAMS_UI commands. Note that you must hide/unhide every individual parameter within a group, not just the group topic itself, otherwise bugs can occur such as parameters appearing in the timeline when keyframed.

```cpp
AEGP_StreamRefH wfm_streamH = NULL;
A_Boolean hide_wfmB = !params[QPGA_SCOPE_OPTION_WFM]->u.bd.value;
AEGP_EffectRefH meH = NULL;
AEGP_SuiteHandler suites(in_data->pica_basicP);
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(NULL, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(NULL, meH, QPGA_WFM_TOPIC_START, &wfm_streamH));
ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(wfm_streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, hide_wfmB));
if (meH){
  ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
}
if (wfm_streamH){
  ERR2(suites.StreamSuite2()->AEGP_DisposeStream(wfm_streamH));
}
```

*Tags: `aegp`, `debugging`, `params`, `ui`*

---

### What does the 'invalid filter 25::3' error mean when loading SDK examples in After Effects?

The 'invalid filter 25::3' error typically indicates a dependency issue with the compiled plugin. This often occurs when the Visual Studio compiler version or runtime libraries don't match what After Effects expects. For CS5 and above, plugins must be compiled as x64 (64-bit), not x86 (32-bit), as After Effects no longer loads x86 plugins. Use Dependency Walker to verify all dependencies are present and correct. Additionally, ensure you have the correct Visual C++ runtime libraries installed, such as MSVCR100D.DLL from the Microsoft Visual C++ Redistributable package.

*Tags: `build`, `debugging`, `pipl`, `sdk`, `windows`*

---

### What is Dependency Walker and how is it useful for debugging After Effects plugin issues?

Dependency Walker is a diagnostic tool that analyzes executable files and their dependencies. For After Effects plugin development, it helps identify missing or mismatched DLL files and runtime libraries that can cause plugins to fail loading. By running your compiled plugin through Dependency Walker, you can see all unresolved dependencies, which helps troubleshoot errors like 'invalid filter 25::3'. The tool is available at http://www.dependencywalker.com/

*Tags: `build`, `debugging`, `tool`, `windows`*

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

### Why does buffer height calculation blow up to around 500 when iterating through rows with zero gutter values?

The issue was caused by incorrect casting of the world as the base address for pixels. The correct approach is to use AEGP_GetBaseAddr32(worldH, &baseAddress) to properly obtain the base address for pixel access, rather than casting the world handle directly.

```cpp
ERR(ws3P->AEGP_GetBaseAddr32(worldH, &baseAddress));
```

*Tags: `aegp`, `buffer`, `debugging`, `memory`, `render-loop`*

---

### Why is a newly added layer not visible in the composition, showing the source layer instead on the current frame?

Adding a layer during a PF_Cmd_RENDER call causes rendering issues because the layers involved in the render are listed and checked prior to the render call. When you add a new layer during rendering, you mess with pre-made lists of render items. The solution is to add the new layer to the composition during other calls, preferably during PF_Cmd_UPDATE_PARAMS_UI or during idle process, not during render.

```cpp
ERR(suites.LayerSuite7()->AEGP_AddLayer(res_item, cResCompH, &res_layer));
```

*Tags: `aegp`, `debugging`, `layer-checkout`, `params`, `render-loop`*

---

### How can you detect when a user sets parenting on a layer in After Effects?

There is no direct event to detect parenting changes. However, you can use several workarounds: (1) Keep a hidden slider that tracks the parent layer's index and detect changes via re-render calls when the index changes, (2) Register a command hook with the 'ALL COMMANDS' flag to monitor all commands sent by AE and look for parenting-related commands, or (3) Use expressions to recursively detect parenting changes through the layer chain. A slider with expressions is a practical approach for tracking parent changes.

*Tags: `aegp_api`, `debugging`, `params`, `ui`*

---

### Why does After Effects report 'could not locate entrypoint' when loading a plugin?

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

*Tags: `aegp`, `build`, `debugging`, `pipl`, `windows`*

---

### What causes 'parameter count mismatch in plug-in effect' error in After Effects plugins?

This error occurs when out_data->num_params does not match the actual number of parameters defined in ParamsSetup. The num_params value must equal the last entry in your parameter enumeration (e.g., SHIFT_NUM_PARAMS), which counts all parameters including the base layer input at index 0. Additionally, each parameter must have a unique stream ID assigned for proper data persistence across plugin versions.

```cpp
enum {
    BASE = 0,           // layer input always at 0
    MY_SLIDER_1,
    MY_SLIDER_2,
    MY_POINT,
    NUM_PARAMS          // use this for out_data->num_params
};

out_data->num_params = NUM_PARAMS;
```

*Tags: `aegp`, `debugging`, `params`*

---

### How can I catch mouse wheel events in After Effects plugins?

After Effects does not supply effects with mouse wheel events through its standard API. The SDK only provides clicks, drags, and cursor enter/exit events for UI regions. If you need mouse wheel support on Windows, you can use the HWND descriptor approach, but this is only available for AEGP Palette plugins, not standard effects.

*Tags: `aegp`, `debugging`, `ui`, `windows`*

---

### How can I prevent PF_Param_POINT from intercepting mouse click and drag events near custom UI controls?

The issue is that After Effects' interface takes higher priority over plug-in interface and intercepts drag events. The solution is to disable the point parameter UI when needed by setting the PF_PUI_DISABLED flag using PF_UpdateParamUI. This makes the point parameter unselectable and prevents it from intercepting clicks and drags. You can set this flag selectively when the cursor is close to your custom control or inside the comp window.

```cpp
def.ui_flags = PF_PUI_DISABLED;
suites.ParamUtilsSuite1()->PF_UpdateParamUI(in_data->effect_ref, paramIndex, &def);
```

*Tags: `debugging`, `params`, `ui`*

---

### Why can't I draw a custom UI in PF_Window_LAYER even though it works in PF_Window_COMP?

This was a bug in After Effects CS3 and CS4 where custom UI drawing in layer windows was not supported. The plug-in would receive PF_Event_DRAW calls and return a buffer, but After Effects would ignore the rendered output. This issue was fixed in CS5. As a workaround, you may need to upgrade to CS5 or later, or reference how other plugins like 'meshwarp' or 'vector paint' handle this limitation.

*Tags: `ae-versions`, `custom-ui`, `debugging`, `ui`*

---

### Why does calling PF_ForceRerender() on a text layer result in 'layer does not have a source' error?

Text layers and shape layers do not have a source, unlike solids. Any source-related operation performed on a text layer will generate this error. As a workaround, instead of using PF_ForceRerender(), you can add a dummy parameter to the effect and call AEGP_SetStreamValue() to trigger rendering of all layers with the effect.

*Tags: `aegp`, `debugging`, `params`, `text-layer`*

---

### How do I resolve error 126 when users try to load my After Effects plugin?

Error 126 typically indicates a dependency issue. Use Dependency Walker to analyze your plugin and identify missing or problematic DLL dependencies. If your plugin requires MSVCR100D.DLL (debug runtime), change your C/C++ code generation runtime library setting: use /MT to statically link the runtime library into your plugin (no external DLLs needed, larger file size), or /MD to use the release version of MSVCR100 (requires distributing the DLL). Test your plugin on a non-developer machine using Dependency Walker to ensure no problematic dependencies remain. Avoid requiring debug runtime libraries in production builds.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What tool can I use to check for dependency issues in my After Effects plugin?

Dependency Walker is a useful tool for analyzing plugin dependencies on Windows. It can identify external DLL requirements and help diagnose loading errors. Run it on both 32-bit and 64-bit versions of your plugin to ensure compatibility across architectures.

*Tags: `debugging`, `deployment`, `tool`, `windows`*

---

### How do you configure Debug and Release build configurations in Xcode for After Effects plugins?

Duplicate the Debug configuration in Xcode and rename it to Release. Remove the '_DEBUG' preprocessor macro from the Release version. Enable 'Strip debug symbols during copy' in the Release configuration to reduce binary size. Additional optimizations may be applied depending on your specific needs.

*Tags: `build`, `debugging`, `macos`*

---

### Why can't VC++ 2008/2010 Express open After Effects SDK example projects?

The After Effects CS5 SDK example projects require 64-bit compiler configurations, which are not checked by default in the Visual Studio installer. You need to install the 64-bit compiling options during Visual Studio setup. Additionally, ensure the Windows SDK is installed and that Visual Studio is configured to look in the correct SDK paths. The Premiere Pro CS5 SDK works because it includes both 32-bit and 64-bit configurations to support 32-bit applications like Encore CS5.

*Tags: `build`, `debugging`, `sdk`, `windows`*

---

### What is the correct AEGP API method to access shape layer contents versus effects?

To get layer effects, use AEGP_GetLayerEffectByIndex. However, shape layer contents (like Polystar, ZigZag, Repeater) are not accessed through the stream suite or effect methods. Instead, use PF_PathDataSuite1 for path geometry, or employ workarounds such as rendering the shape yourself or using AEGP_GetReceiptWorld() on a duplicate composition containing only the shape layer.

*Tags: `aegp`, `debugging`, `params`, `shape-layer`*

---

### Why am I getting black pixels (0,0,0,0 RGBA) when rendering a full resolution frame with RenderAndCheckoutFrame?

The issue is likely that you're using AEGP_GetActiveItem and AEGP_GetActiveLayer, which don't necessarily refer to your effect's layer. Use AEGP_GetEffectLayer instead, which is guaranteed to return your effect's actual layer. The black pixels suggest the render is not finding the correct layer to render.

*Tags: `aegp`, `debugging`, `memory`, `smart-render`*

---

### How do you get resize callbacks for a dockable plugin window in macOS?

To receive resize callbacks for a dockable plugin window, you need to register for the kEventControlBoundsChanged event in your event handler specification. Set up your event handler with an EventTypeSpec array that includes {kEventClassControl, kEventControlBoundsChanged}, which will trigger whenever the control is resized. You may also want to register for kEventControlOwningWindowChanged to detect when the control's owning window changes.

```cpp
static const EventTypeSpec kControlEventSpec[] = {
  {kEventClassCommand, kEventProcessCommand},
  {kEventClassControl, kEventControlDraw},
  {kEventClassControl, kEventControlOwningWindowChanged},
  {kEventClassControl, kEventControlBoundsChanged}
};
OSErr err = InstallEventHandler(GetControlEventTarget(i_refH),
  NewEventHandlerUPP(S_EventHandler),
  GetEventTypeCount(kControlEventSpec),
  kControlEventSpec,
  (void*)this,
  NULL);
```

*Tags: `debugging`, `macos`, `ui`*

---

### How do I fix the "project.pbxproj is not writeable" error when working on a Skeleton plugin in Xcode on Mac?

If Xcode shows an error that the project.pbxproj file is not writable and cannot be saved, the issue may be related to SCM (source control management) settings in the project file. To fix this: (1) Back up the Skeleton project first, (2) Quit Xcode, (3) Navigate to the Skeleton.xcodeproj file in Finder, (4) Option-click it and select "Open Package Content" (since .xcodeproj is actually a package/folder), (5) Find and edit the project.pbxproj file in a text editor, (6) Search for and remove any paragraphs that reference SCM settings. Alternatively, check if the project files are located on a backup drive like Time Machine, as this can also cause write permission issues.

*Tags: `build`, `debugging`, `macos`, `xcode`*

---

### How do I fix the 'project file is not writeable' error when modifying After Effects SDK skeleton projects in Xcode?

The issue is typically that the project files have write-protection enabled. Check if files are locked by selecting them in Finder and opening Get Info to see if the Locked checkbox is checked. If so, uncheck it for each file. This write-locking is common when files are transferred from Windows due to read-only media assumptions. If files aren't locked, verify project settings don't have SCM (Source Control Management) enabled, as this can also cause writeability issues with .pbxproj files.

*Tags: `build`, `debugging`, `macos`, `skeleton`, `xcode`*

---

### How do you compile the Panelator sample to a release version that runs on non-developer machines?

When compiling Panelator for release, change the runtime library from /MD to /MT to make the plugin independent of external MSVC runtime DLLs. Additionally, ensure all supporting code files in the 'supporting code' folder have consistent compiler settings (not mixed MD/MDd). If using Visual Studio 2008 instead of the SDK-intended VS2005, be aware that 2008 uses MSVCR90.DLL instead of MSVCR80.DLL, which can cause compatibility issues. Using /MT will statically link all libraries into the plugin, making it dependent only on KERNEL32.DLL. Use Dependency Walker to verify the plugin's dependencies before deployment.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---
