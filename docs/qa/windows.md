# Q&A: windows

**118 entries** tagged with `windows`.

---

## How do you debug the 'plugin wasn't installed correctly' error on Windows?

Use dumpbin (included with Visual C++) to check plugin dependencies: 'dumpbin /dependents plugin.aex'. Common causes: (1) Missing Visual C++ Redistributable packages (download from https://aka.ms/vs/17/release/vc_redist.x64.exe). (2) Missing DLL dependencies. (3) Error in GlobalSetup. (4) Debug builds accidentally linking to debug CRT DLLs (msvcp140d.dll). Try setting a breakpoint on GlobalSetup in debug mode to see if the entry point is reached.

```cpp
dumpbin /dependents plugin.aex
```

*Contributors: [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2025-10-16 · Tags: `debugging`, `windows`, `dependencies`, `dumpbin`, `vcredist`, `installation`*

---

## Should .aex plugins be code signed on Windows?

.aex plugins are regular DLLs (just renamed), so they can be codesigned. Microsoft currently allows unsigned DLLs to be loaded from a signed process, but they have hinted this might change in the future (was supposed to happen with Win11). On macOS, a plugin is a bundle (renamed folder), and you don't need to sign the actual plugin binary inside it as long as the bundle itself is signed.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2023-12-13 · Tags: `code-signing`, `aex`, `dll`, `windows`, `macos`, `security`*

---

## How do you debug/log variables from a C++ AE plugin on Windows?

Use OutputDebugString to send debug output that can be viewed with tools like DebugView or Visual Studio's Output window.

```cpp
std::string log1 = "variable: " + std::to_string(variable) + "\n"; OutputDebugString(log1.c_str());
```

*Contributors: [**Jonah (Haligonian/Baskl)**](../contributors/jonah-haligonian-baskl/) · Source: aescripts discord · 2024-05-17 · Tags: `debugging`, `logging`, `outputdebugstring`, `windows`, `cpp`*

---

## What cross-platform GPU technology issues exist between Windows and macOS?

OpenCL-based plugins perform well on Windows but are significantly slower on macOS. This suggests that different GPU technologies have vastly different performance characteristics across platforms, requiring careful consideration when choosing GPU frameworks.

*Tags: `gpu`, `cross-platform`, `macos`, `windows`*

---

## What is the recommended approach for supporting GPU rendering across Windows and macOS?

In the future, maintaining two completely separate code bases may be necessary: one using Metal for macOS and one using OpenGL or another framework for Windows, due to significant platform-specific GPU compatibility issues.

*Tags: `gpu`, `metal`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## Did you try Metal first before using MoltenVK for cross-platform GPU development?

Yes, Metal was tried first, but MoltenVK with Vulkan was chosen because it reduces code duplication between Windows and Mac platforms. However, there are some Mac-specific limitations to handle. Using Metal directly alongside DirectX/Vulkan would be like writing two separate plugins, and the developer encountered inconsistent results with SPIR-V to MSL versus SPIR-V to Vulkan without MoltenVK.

*Tags: `gpu`, `metal`, `vulkan`, `cross-platform`, `macos`, `windows`*

---

## What GPU APIs are being used for Windows and Mac support?

For cross-platform development, MoltenVK (a Vulkan layer over Metal) is used, which allows shared code between Windows and Mac. Alternatively, some developers keep OpenGL on Windows and Metal on Mac. OpenGL 4.x and 3.3 are still in use by major plugins like Element 3D and Helium, though there are concerns about Apple potentially removing OpenGL support entirely.

*Tags: `gpu`, `opengl`, `metal`, `vulkan`, `cross-platform`, `macos`, `windows`*

---

## How does the VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension avoids the need to allocate a staging buffer entirely. It allows you to turn a CPU-side void* directly into a VkDeviceMemory object, enabling direct data copying from an EffectWorld onto the GPU and allowing the GPU to write into it as well. This provides significant gains in both memory and speed, especially for large 4K frames. However, MoltenVK does not currently support this extension, though it is worth conditionally taking advantage of on Windows.

*Tags: `vulkan`, `memory`, `gpu`, `windows`, `cross-platform`*

---

## What causes a plugin to fail loading during After Effects startup with a 'bad filter' error when the entry point cannot be found?

This type of error typically indicates a dependency issue rather than a driver or OS problem. The error occurs before the Vulkan instance initialization (as evidenced by no Vulkan errors in logs), suggesting the problem happens during plugin loading or initialization of global data. Common causes include missing or incompatible dependencies, incorrect plugin structure, or issues in the main entry point implementation.

*Tags: `debugging`, `build`, `deployment`, `windows`*

---

## What can cause main entry point errors in After Effects plugins?

Main entry point errors can happen when a dll is missing from the system. Additionally, the libraries/dlls need to have the right architecture (32-bit vs 64-bit) to match the plugin and After Effects installation.

*Tags: `windows`, `build`, `deployment`, `debugging`*

---

## How can you verify that all required dependencies are present in a Windows plugin build?

Use Dependency Walker on Windows to check for missing shared libraries and ensure all external dependencies are properly resolved.

*Tags: `windows`, `build`, `debugging`, `deployment`*

---

## How do you print to the console when debugging an After Effects plugin in Visual Studio?

On Mac, use printf. On Windows, use the OutputDebugStringA macro. You can also use AEGP_WriteToOSConsole from the utility suite, though this may not work in Premiere. OutputDebugStringA is the recommended approach for Windows console output during debugging.

```cpp
OutputDebugStringA("debug message");
```

*Tags: `debugging`, `windows`, `macos`*

---

## What causes an 'entry point not found' error on Windows after a plugin is loaded?

Two main possibilities: a missing dependency (such as a Visual C++ redistributable package) or an error in global setup. The issue may be resolved by installing the Visual C++ redistributable package. In debug mode, you can set a breakpoint on global setup to verify if it's being called.

*Tags: `windows`, `deployment`, `build`, `debugging`*

---

## Is it normal to see msvcp140d.dll when compiling with Visual Studio 2022 (v143)?

Yes, this is normal. The msvcp140d.dll appears in compilations with Visual Studio 2022 (v143). However, if the entry point is not found, it typically indicates a dependency issue rather than a problem with this DLL itself.

*Tags: `windows`, `build`, `deployment`*

---

## How should memory be freed differently on Mac versus Windows?

Memory should be freed later in another thread on Mac but not on Windows, due to platform-specific threading and memory management differences.

*Tags: `memory`, `macos`, `windows`, `threading`*

---

## How can I debug a crash in After Effects on Windows that occurs during project loading when my plugin is installed, but not on Mac?

Use a debugger with breakpoints on EffectMain to trace when the crash occurs relative to plugin initialization. The crash appears to happen after GlobalSetup and ParamsSetup complete successfully (around 85% project load) with no plugin code executing at that point, suggesting a symbol clash or state corruption rather than direct plugin code failure. Try importing the project into a new empty project as a workaround, or preload the conflicting library (Cineware in this case) before opening the project.

*Tags: `debugging`, `windows`, `macos`, `crash`*

---

## Why might a plugin cause a third-party library like Cineware to crash during After Effects project loading even when the plugin code isn't executing at the crash point?

The plugin may be defining a symbol that clashes with the third-party library's loading process, or it may be causing some state corruption in After Effects that affects library initialization later. The fact that preloading Cineware before opening the project allows it to work suggests a symbol collision or initialization order issue rather than a direct code bug.

*Tags: `debugging`, `windows`, `memory`*

---

## How can I use RenderDoc to debug GPU code in After Effects without crashing?

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

*Tags: `gpu`, `debugging`, `windows`*

---

## What is the correct way to call AEGP_StartUndoGroup in After Effects 2025?

In After Effects 2025 beta and later, AEGP_StartUndoGroup must be called with an empty string "" rather than null. Passing null will cause a crash, while passing an empty string works as expected and does not add an entry to the Undo stack.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `undo`, `macos`, `windows`, `debugging`*

---

## How can a plugin require AVX2 support or handle the AVX2 cutoff in After Effects?

After Effects has required AVX2 since version 2023. Given that Steam hardware survey shows 98.25% support for AVX and 96.30% support for AVX2 (with growing adoption rates), targeting AVX2 is practical for modern plugins. Rather than wrapping every dispatch with if(HasAVX()) checks, you can compile your plugin with AVX2 as a baseline requirement, allowing you to leverage AVX2 optimizations throughout your codebase without conditional branching. This approach can potentially double performance compared to targeting x86-64-v2 (up to SSSE3).

*Tags: `build`, `deployment`, `windows`, `performance`*

---

## How can a plugin support AVX2 requirements given After Effects 2023+ requires it, and should developers worry about older CPU compatibility?

After Effects has required AVX2 since version 2023. Based on Steam hardware survey data, 96.30% of systems support AVX2 (growing 1.63% per month) and 98.25% support AVX (growing 0.77% per month), though this may not directly overlap with video editing users. Most developers using After Effects in 2025, even older versions, are unlikely to encounter machines without AVX2 support. For selective CPU feature support, ISPC (Intel SPMD Program Compiler) can automatically compile multiple versions and switch between them at runtime.

*Tags: `build`, `cpu`, `deployment`, `cross-platform`, `windows`*

---

## How should dynamic dropdown lists be updated in After Effects plugins as of CC2025.2 to avoid crashes and empty display?

Use the AEGP StreamSuite to update dropdown parameters via AEGP_SetStreamValue instead of directly modifying the param union during param_ui thread. Call AEGP_GetNewEffectStreamByIndex to get the stream reference, create an AEGP_StreamValue with the new value, use AEGP_SetStreamValue to apply it, and dispose the stream with AEGP_DisposeStream. This approach is more reliable than strncpy_s manipulation of namesptr, though it may have undo history implications that should be tested.

```cpp
PF_Err setFloatOrBoolOrDropdownParamViaAEGP(uint16_t i_param_to_change, float val,
                                     AEGP_PluginID my_id, AEGP_EffectRefH *effect_refHP, AEGP_SuiteHandler *suitesP, GlobalData* global_data)
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

*Tags: `popup`, `aegp`, `params`, `ui`, `windows`*

---

## How do you debug a plugin that fails to load in After Effects and appears offline in Premiere?

Check if the error occurs before global_setup is called. If it happens before that point, it may be due to a library dependency that AE loaded in a new version that your plugin uses. Verify whether the issue occurs on both Windows and macOS or only on specific platforms.

*Tags: `debugging`, `deployment`, `macos`, `windows`, `pipl`*

---

## What could cause a simple C++ plugin with no external dependencies to fail loading?

Even simple plugins without explicit dependencies can fail due to library dependencies loaded by After Effects itself in newer versions. The issue may be platform-specific, as it can work on one platform (like Windows) while failing on another (like macOS).

*Tags: `debugging`, `deployment`, `cross-platform`, `macos`, `windows`*

---

## Why are some plugin files becoming 0 bytes after upgrading to After Effects 2025.2?

Multiple plugins were found to be 0 bytes (emptied) after upgrading to AE 2025.2, while released versions work properly. This appears to be a bug in the 2025.2 release that corrupts plugin files in the MediaCore folder during upgrade, though the root cause was not determined in the conversation.

*Tags: `deployment`, `macos`, `windows`, `build`*

---

## Is the After Effects code signing issue affecting binary plugins on Windows as well as macOS?

The binary code signing issue appears to be Mac-only. It's related to After Effects being built with the latest Xcode, which applies stricter code signature checks than before. There was a separate ZXP signing issue that affected both macOS and Windows platforms, but that's a different problem related to the ZXPSignCmd tool.

*Tags: `macos`, `windows`, `deployment`, `codesign`*

---

## Why does After Effects crash when launched from lldb with certain launch.json configurations?

The crash was caused by missing the "sourceLanguages": ["rust"] line in the launch.json configuration. Adding this line to the debugger launch configuration prevents the crash from occurring.

```cpp
"sourceLanguages": ["rust"]
```

*Tags: `debugging`, `build`, `windows`, `macos`*

---

## Is there a reliable way to get the current After Effects or Premiere version number (e.g., 2025)?

The in_data->version major and minor fields do not appear to be reliably updated between versions, returning the same values on both 2024 and 2025. This approach may work on Mac AE but needs verification on Premiere and Windows.

*Tags: `premiere`, `macos`, `windows`, `deployment`, `debugging`*

---

## How can you implement custom debug assertions that trigger breakpoints without crashing the program?

Create assertion macros using preprocessor directives that automatically trigger a breakpoint if a debugger is attached, rather than stopping the program. This can be implemented using assembly calls with platform-specific code paths for Windows and macOS. C++26 will have similar built-in features.

*Tags: `debugging`, `cross-platform`, `macos`, `windows`*

---

## How should I debug the 'plugin wasn't installed correctly' error on Windows 11?

Use dumpbin from Visual C++ to check plugin dependencies. Run 'dumpbin /dependents plugin.aex' in a terminal to see what dependencies are listed. The issue may be related to missing Microsoft Visual C++ runtime package that needs to be installed on the user's system.

```cpp
dumpbin /dependents plugin.aex
```

*Tags: `windows`, `debugging`, `build`, `deployment`*

---

## When building a Rust SDK plugin with 200-300 crates, are all dependencies included in the final binary?

The dependencies should be included in the final binary when building with the Rust SDK, but installation errors can occur if required system libraries like Microsoft Visual C++ runtime are missing on the target system.

*Tags: `build`, `windows`, `deployment`*

---

## Is DirectX rendering already implemented in After Effects production, and has anyone made a plugin using it?

DirectX rendering was not widely available for plugins as of the conversation date. Adobe replaced the UI from OpenGL to DX12 in 2021, and the latest SDK has an official flag for it, but it's likely not yet ready to expose to plugins. It may be added for Windows on ARM device support, where DirectX is the first-party option. Adobe has not been supporting Vulkan, possibly because the version of AFX that passes DX12 handles to plugins hasn't been released yet.

*Tags: `gpu`, `directx`, `windows`, `apple-silicon`, `sdk`*

---

## How can DirectX compute be enabled for debugging in After Effects?

DirectX compute can be enabled by setting the 'forcedirectxcompute' flag in the console panel within AE settings. However, a full computer restart is required for the setting to take effect; restarting AE alone is not sufficient. Once properly configured, it becomes possible to debug DirectX for future productions.

*Tags: `gpu`, `directx`, `debugging`, `windows`*

---

## How can you install and activate Adobe After Effects CS6 on Windows 11?

CS6 may have compatibility issues with Windows 11. Try running the installer in compatibility mode, or consider using Windows 10 instead if Windows 11 proves problematic. Adobe's activation servers for CS6 may no longer be functional, which can prevent license key activation on new machines.

*Tags: `windows`, `cross-platform`, `deployment`, `reference`*

---

## What is the recommended approach for multithreading in Premiere plugins when the iterate suite doesn't work?

If the Iterate8Suite doesn't work reliably in Premiere (particularly with certain colorspace configurations), you should implement your own multithreading code instead of relying on the suite. Standard approaches include using std::parallel on Windows and equivalent threading libraries for Mac (like Grand Central Dispatch or pthreads). The iterate suite should theoretically work for 8-bit ARGB, but if you encounter issues, custom multithreading implementations are a stable fallback.

*Tags: `premiere`, `threading`, `multithreading`, `performance`, `bgra`, `macos`, `windows`*

---

## How can you prevent the compiler from optimizing a render function in an After Effects plugin?

Add the `__attribute__ ((optnone))` compiler attribute before the render function declaration. This tells the compiler to skip optimization for that specific function, even if the rest of the plugin is compiled with optimizations enabled. This workaround was found to resolve rendering issues in some cases.

```cpp
__attribute__ ((optnone))
void renderFunction() {
  // render code
}
```

*Tags: `render-loop`, `build`, `debugging`, `macos`, `windows`*

---

## What are the main challenges when updating GPU plugins for MFR compatibility?

GPU plugins face significant challenges with MFR that CPU plugins don't encounter. Issues include crashes that occur only after long rendering periods (making them hard to debug), GPU caching complications, and sequence data handling. OpenGL-based plugins particularly struggle with Mac-specific issues. James Whiffin reported that even using Adobe's GLator sample, Mac OpenGL compatibility remains problematic. The recommendation is to consider maintaining completely separate codebases: one for Metal on Mac and one for Windows, as OpenGL performance on Mac can be severely degraded compared to Windows.

*Tags: `mfr`, `gpu`, `opengl`, `metal`, `macos`, `windows`, `caching`, `sequence-data`*

---

## Should you use Metal directly or MoltenVK for cross-platform GPU plugin development on macOS?

Using Vulkan with MoltenVK (a Vulkan layer over Metal) reduces code duplication between Windows and macOS compared to maintaining separate Metal and DirectX implementations. However, there are some macOS-specific limitations to account for. Alternatively, keeping OpenGL on Windows and Metal on macOS is viable, but carries the risk that Apple may eventually remove OpenGL support entirely, as they have deprecated it in favor of Metal.

*Tags: `gpu`, `metal`, `vulkan`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## How can VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension (available on Windows) avoids the need to allocate a staging buffer by allowing a CPU-side void* to be turned directly into a VkDeviceMemory object. This enables copying data directly from an EffectWorld onto the GPU and allows the GPU to write back into it without redundant upload/download copies, resulting in significant gains in both memory and speed, especially for large 4K frames.

*Tags: `vulkan`, `gpu`, `memory`, `windows`, `optimization`*

---

## How can I diagnose missing DLL dependencies in a Windows After Effects plugin?

Use Dependency Walker on Windows to check for missing DLLs and verify that all libraries and DLLs have the correct architecture (32-bit vs 64-bit) matching your plugin build. Also check system folders like System32 where driver-installed libraries like Vulkan may be located.

*Tags: `windows`, `debugging`, `build`, `deployment`*

---

## What should I verify when using external libraries like Vulkan in an After Effects plugin?

Ensure that external libraries are properly installed on the user's system (e.g., Vulkan drivers install libraries to System32), check for any transitive dependencies from licensing tools or other libraries, and verify that all linked DLLs match the architecture of your plugin build.

*Tags: `windows`, `vulkan`, `deployment`, `build`*

---

## What does After Effects error 25:3 mean and what causes it?

Error 25:3 in After Effects indicates that an effect cannot be applied because it cannot be initialized. This error is typically associated with a corrupted installation or missing dependencies. It has been observed to occur specifically when users are running After Effects over Windows Remote Desktop software, suggesting potential issues with plugin initialization in remote desktop environments.

*Tags: `debugging`, `windows`, `deployment`, `plugin-error`*

---

## What causes After Effects error 25:3 when using plugins over remote desktop software?

Error 25:3 ('This effect cannot be applied because it cannot be initialized') typically indicates a corrupted installation or missing dependency. However, when the error occurs specifically for users accessing After Effects through Windows Remote Desktop, it may be related to remote desktop-specific issues rather than actual corruption. The problem appears to be isolated to remote desktop environments rather than standard installations.

*Tags: `windows`, `debugging`, `deployment`*

---

## How do you print to the console when debugging an After Effects plugin?

On Mac, use printf. On Windows, use OutputDebugStringA macro to output debug messages that will appear in Visual Studio's console. Note that AEGP_WriteToOSConsole from the utility suite exists but may not work reliably. Starting AE with the -debug flag can help, but OutputDebugStringA is the more reliable approach for Windows debugging.

```cpp
OutputDebugStringA("Debug message here");
```

*Tags: `debugging`, `windows`, `macos`, `aegp`*

---

## Has anyone ported the After Effects plugin build system to use CMake instead of Visual Studio and Xcode?

The question was asked but no one in the conversation provided a definitive answer or shared an existing CMake-based build system for AE plugins. This remains an open challenge in the community.

*Tags: `cmake`, `build`, `cross-platform`, `xcode`, `windows`*

---

## What is the correct string format for specifying x64 architecture in After Effects plugin build configurations?

The correct string for x64 architecture is 'x86_64', not 'x64'. Using an unrecognized architecture string may cause the build system to fall back to your native architecture instead of the intended target.

*Tags: `build`, `cross-platform`, `windows`, `macos`*

---

## What causes 'entry point not found' errors on Windows when an After Effects plugin loads?

This error typically indicates a missing dependency, most commonly a missing Visual C++ redistributable package. Even if Visual Studio is installed on the development machine, the target machine may not have the required Visual C++ runtime libraries. On Windows 10, installing the Visual C++ 2022 redistributable (https://aka.ms/vs/17/release/vc_redist.x64.exe) often resolves the issue. Debugging can be done by setting a breakpoint on global setup in debug mode to verify if the plugin initialization is being reached.

*Tags: `windows`, `deployment`, `debugging`, `build`*

---

## Is it normal to see msvcp140d.dll when compiling an After Effects plugin with Visual Studio 2022 (v143)?

Yes, this is normal behavior. The msvcp140d.dll file (Visual C++ Debug Runtime) appears in the compiled output when using Visual Studio 2022. However, if you're encountering entry point errors, the issue is likely related to other missing dependencies or libraries that the plugin relies on, rather than the presence of this DLL.

*Tags: `windows`, `build`, `debugging`*

---

## Why is there a memory leak when using AEGP_memorysuite on macOS but not Windows during export?

A user reported a memory leak when using AEGP_memorysuite to allocate, lock, memcpy to/from, and unlock memory handles during export (not preview render). The leak was macOS-specific and did not occur on Windows. Testing with different flags (mem_NONE, mem_CLEAR, mem_QUIET) showed the leak was less severe with mem_QUIET. Replacing the AEGP memory handle with a temporary pointer using new/delete eliminated the leak. Additionally, replacing memcpy with strncpy solved the issue when copying from the mem pointer to a buffer, but not in the opposite direction. This suggests a potential platform-specific issue with how AEGP_memorysuite handles memory during the export pipeline, possibly related to threading or memory alignment differences between macOS and Windows.

*Tags: `memory`, `aegp`, `macos`, `windows`, `debugging`, `threading`*

---

## How can you track and debug memory allocation and deallocation issues in After Effects plugins?

Wrap memory allocation and deallocation in custom logging wrapper functions to get a definitive record of what happened. Additionally, create a base class (like ObjectBase) that all C++ classes descend from to keep track of live instances of each type, which helps identify memory leaks and allocation mismatches. This approach also makes it easy to report current counts of allocations versus frees.

*Tags: `memory`, `debugging`, `threading`, `macos`, `windows`*

---

## How can you debug a plugin crash that occurs during project loading in After Effects on Windows but not on Mac, seemingly triggered by an external library like Cineware?

Use breakpoints in EffectMain to trace the plugin's initialization sequence across GlobalSetup and ParamsSetup. If the crash occurs well after the plugin's code executes (with no breakpoints hit for several seconds), it suggests the plugin may be inadvertently affecting how other libraries like Cineware load, possibly through symbol name clashing. A workaround is to pre-load the problematic library (e.g., Cineware) onto a clip before opening the project. Additionally, try importing the project into a new empty project rather than directly opening it, as project corruption or platform-specific state issues may be the root cause. Cross-platform differences (Mac vs Windows) suggest OS-specific library loading or initialization issues.

*Tags: `debugging`, `windows`, `macos`, `cross-platform`, `plugin-interaction`*

---

## What is the relationship between macOS and Xcode versions for After Effects plugin development?

When updating macOS, you often need to update Xcode to a compatible version. For example, updating from macOS 13.x to macOS 14 required updating from Xcode 14 to Xcode 15. Windows and Visual Studio are more flexible in this regard. The best practice for plugin development is to maintain separate development and testing machines with different OS versions to catch version-specific bugs and unexpected behavior, though this is an expensive option.

*Tags: `macos`, `build`, `debugging`, `cross-platform`, `windows`*

---

## What GLSL assignment operators are supported across different GPU platforms in After Effects plugins?

James Whiffin encountered an issue where a specific GLSL operator caused compilation failures. The error was identified through shader compilation feedback. The conversation indicates that while some assignment operators like += work correctly, the /= operator caused issues on at least one platform (Windows or Mac, GPU reference unknown). This suggests developers need to be cautious about which GLSL assignment operators are universally supported across different GPU architectures when developing AE plugins.

*Tags: `gpu`, `opengl`, `glsl`, `debugging`, `cross-platform`, `macos`, `windows`*

---

## How can I use RenderDoc to debug GPU code in After Effects plugins without crashes?

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

*Tags: `gpu`, `debugging`, `windows`, `render-loop`*

---

## What is plugplug.DLL and how can it be used for inter-plugin communication?

plugplug.DLL is a DLL that allows After Effects plugins to communicate with each other. You can call the extern "C" functions exposed by plugplug.DLL from your C++ plugin to enable inter-plugin communication. This concept is related to loading plugins from within other plugins, as discussed in the Adobe community post: https://community.adobe.com/t5/after-effects-discussions/how-to-load-a-plugin-from-another-plugin/td-p/6738182

*Tags: `aegp`, `cross-platform`, `windows`, `deployment`, `reference`*

---

## How can you reverse-engineer and integrate a DLL into an After Effects plugin?

One approach is to use depends.exe to inspect the DLL's dependencies and exports, then reference existing samples (like the Illustrator sample) as a guide to understand the integration pattern. While not always a 1:1 match, samples can provide enough context to build your own wrapper. Modern approaches may involve directly linking to the DLL and providing a header file, but manual reverse-engineering through dependency analysis and sample-based implementation has proven effective.

*Tags: `debugging`, `build`, `windows`, `reference`*

---

## What change in After Effects 2025 beta affects AEGP_StartUndoGroup with null parameter?

Starting from After Effects 2025 beta, passing null to AEGP_StartUndoGroup will cause a crash. In previous versions, this did not cause adverse effects, but it is no longer safe to do so.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup(null);
```

*Tags: `aegp`, `debugging`, `crash`, `api-change`, `macos`, `windows`*

---

## What parameter should be passed to AEGP_StartUndoGroup to avoid crashes in After Effects 2025?

As of After Effects 2025 beta, passing null to AEGP_StartUndoGroup() causes a crash, whereas passing an empty string "" works correctly and functions as expected without adding an entry to the Undo stack. Previous versions did not crash with null, so this is a breaking change in AE 2025.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `debugging`, `macos`, `windows`*

---

## What is the minimum CPU instruction set support required for After Effects plugins in 2025?

After Effects has required AVX2 support since version 2023. Based on Steam hardware survey data, 98.25% of machines support AVX (growing 0.77% monthly) and 96.30% support AVX2 (growing 1.63% monthly). Even considering only video editing machines, the overlap suggests virtually no users in 2025 are on machines without AVX2 support. This makes it safe to target higher instruction sets like x86-64-v2 (up to SSSE3) or AVX2 for significant performance improvements without runtime checks.

*Tags: `build`, `cpu`, `performance`, `windows`*

---

## How can you reliably get the After Effects version in a plugin when in_data->version major and minor don't update between releases?

James Whiffin noted that in_data->version major and minor fields are not being updated reliably between AE 2024 and 2025, returning the same values. This suggests developers should investigate alternative methods for version detection, such as querying the application directly or using other fields in the plugin data structure, though no specific solution was provided in this conversation.

*Tags: `aegp`, `debugging`, `macos`, `windows`*

---

## How should dynamic dropdown lists be updated in After Effects plugins to avoid crashes in CC 2025.2?

Use the AEGP StreamSuite to update dropdown parameters instead of directly modifying the param_union. Create a function like setFloatOrBoolOrDropdownParamViaAEGP that uses AEGP_GetNewEffectStreamByIndex and AEGP_SetStreamValue to properly update the parameter value. Call this during UpdateParamsUI. Note: There is a known warning that this approach may cause issues with undo history, though the exact impact is unclear.

```cpp
PF_Err setFloatOrBoolOrDropdownParamViaAEGP(uint16_t i_param_to_change, float val,
                                     AEGP_PluginID my_id, AEGP_EffectRefH *effect_refHP, AEGP_SuiteHandler *suitesP, GlobalData* global_data)
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

*Tags: `params`, `ui`, `aegp`, `popup`, `windows`*

---

## How do you debug 'failed to load' errors in After Effects and 'filter offline' errors in Premiere Pro?

When debugging plugin loading failures, check whether the error occurs before global_setup is called. If it happens before global_setup, it may indicate a library dependency issue loaded by AE/Premiere in a newer version that conflicts with your plugin. Test on both Windows and macOS to determine if the issue is platform-specific, as the same plugin may work on one platform but fail on another.

*Tags: `debugging`, `deployment`, `cross-platform`, `premiere`, `macos`, `windows`*

---

## Are there known issues with ZXPSignCmd and plugin signing on Windows?

Yes, there have been reports of ZXPSignCmd segmentation faults and timestamp-related signing failures on Windows. These issues appear to be affecting plugin developers globally, and Adobe was working on releasing a patched version of the ZXPSignCmd tool to resolve them.

*Tags: `windows`, `deployment`, `tool`*

---

## Why does launching After Effects 2024/2025 from LLDB with certain launch configurations cause crashes?

The crash was traced to the launch.json configuration. Adding the line `"sourceLanguages": ["rust"]` to the launch.json prevents the crash from occurring when launching AE through LLDB. The exact mechanism is unclear, but this configuration change resolved the issue that was happening with both the user's and @fad's basic launch/tasks.json setup, even when mediacore was empty.

```cpp
"sourceLanguages": ["rust"]
```

*Tags: `debugging`, `build`, `macos`, `windows`*

---

## How can you reliably get the current After Effects or Premiere version number (e.g., 2025)?

James Whiffin noted that in_data->version major and minor fields are not being reliably updated between versions like 2024 and 2025, returning the same values. Jonah reported that accessing the version appears to work on macOS After Effects, though behavior on Premiere and Windows still needs verification.

*Tags: `aegp`, `cross-platform`, `macos`, `windows`, `premiere`, `debugging`*

---

## How can you implement custom breakpoint assertions in C++ for After Effects plugin development without crashing the program?

Create custom assertion macros that automatically trigger breakpoints only when a debugger is attached, rather than using standard assertions that crash the program. This can be implemented using preprocessor macros with platform-specific code paths for Windows and macOS. While C++26 will have built-in support for this feature, it can be implemented today using inline assembly calls with preprocessor directives.

*Tags: `debugging`, `cross-platform`, `windows`, `macos`, `build`*

---

## How can I debug the 'plugin wasn't installed correctly' error on Windows 11?

Use the dumpbin tool from Visual C++ to check your plugin's dependencies. Run 'dumpbin /dependents plugin.aex' in a terminal to see what dependencies are listed. If dependencies are missing, you may need to install the Microsoft Visual C++ package on the target system.

```cpp
dumpbin /dependents plugin.aex
```

*Tags: `windows`, `debugging`, `deployment`, `build`*

---

## What should I do if a Rust-based After Effects plugin shows dependency errors on Windows?

Even though Rust plugins should include all crates in the final binary, Windows may still require the Microsoft Visual C++ redistributable package to be installed separately. This is a common cause of 'plugin wasn't installed correctly' errors. Ensure the target system has the appropriate Visual C++ package installed.

*Tags: `windows`, `deployment`, `build`, `debugging`*

---

## Is DirectX rendering currently implemented in After Effects production, and how can it be enabled for plugin development?

DirectX rendering support was added to the After Effects SDK with an official flag, though it appears to be in early stages. Adobe replaced the UI from OpenGL to DX12 in 2021. To enable DirectX compute for debugging and testing, use the 'forcedxcompute' setting in the After Effects console panel under settings, but note that a full computer restart (not just AE restart) is required for the setting to take effect. The feature is likely being developed primarily for Windows on ARM device support, as those devices have first-party support for DX12. Adobe has not yet officially supported Vulkan exposure to plugins.

*Tags: `directx`, `gpu`, `windows`, `debugging`, `apple-silicon`*

---

## Should obsolete CPU architecture fields have been removed from in_data when After Effects went 64-bit only?

Yes, the fields in in_data related to old CPU architecture detection (such as Gestalt values for 68x and PowerPC) should have been removed from the plugin API when Adobe transitioned to 64-bit only support around CS5, rather than keeping legacy fields that are no longer used or meaningful.

*Tags: `aegp`, `deprecated`, `backwards-compatibility`, `macos`, `windows`*

---

## Is it possible to install and activate Adobe After Effects CS6 on Windows 11?

CS6 should work on Windows 11, but you may need to run it in compatibility mode. If Windows 11 causes issues, consider testing on Windows 10 instead. Activation may be problematic due to Adobe's legacy license servers, so compatibility mode is a recommended workaround.

*Tags: `windows`, `cross-platform`, `compatibility`, `deployment`*

---

## Why doesn't AEGP_EffectCallGeneric() trigger PF_Cmd_COMPLETELY_GENERAL on Mac when trying to identify a specific effect instance?

According to community expert shachar carmi, AEGP_EffectCallGeneric() has limitations and cannot reliably call from one effect instance to another instance of the same type. The expert suggests that if you need inter-instance communication, you should create a separate AEGP plugin to handle these calls during idle processing, rather than calling directly between effect instances of the same type. As an alternative for identification that doesn't need to survive save/load, you can change a hidden parameter's name to be the identifier and read it via the stream suite.

```cpp
// Instead of calling between same-type effects, use a separate AEGP
// Or use hidden parameter name as identifier via stream suite
AEGP_SetStreamName()
```

*Tags: `aegp`, `params`, `debugging`, `cross-platform`, `macos`, `windows`*

---

## How do you debug a plugin when it's loaded through After Effects Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible After Effects instance for rendering, separate from the main AE process. Breakpoints set in the main application won't be triggered during AME rendering. To debug in this scenario, you need to attach your debugger to the already-running subprocess. On Windows, use the Visual Studio debugger's attach-to-process feature: https://learn.microsoft.com/en-us/visualstudio/debugger/attach-to-running-processes-with-the-visual-studio-debugger?view=vs-2022. You can also observe these processes in Windows Task Manager.

*Tags: `debugging`, `media-encoder`, `windows`, `deployment`*

---

## How can I predict the required buffer size for string conversions in Windows?

Use the Windows API function MultiByteToWideChar with a null pointer for the output buffer parameter. This will return the required buffer size in wide characters without actually performing the conversion. This allows you to accurately allocate the necessary buffer before conversion.

*Tags: `sdk`, `windows`, `string-handling`, `memory`*

---

## How do I make the idle hook get called more frequently to support higher frame rate precomp playback?

Use the AEGP_CauseIdleRoutinesToBeCalled function instead of trying to manually set max_sleepPL. This will cause idle routines to be called more frequently. Note that on macOS the call rate appears to be capped at around 30 calls per second, while on Windows it can reach 60 calls per second.

*Tags: `aegp`, `idle-hook`, `macos`, `windows`, `cross-platform`*

---

## How do I resolve 'Cannot open source file' errors when moving After Effects SDK example projects to a new location?

When copying AE SDK example projects like the supervisor folder to a new location, include directories only apply to .h and .hpp header files, not to .c and .cpp source files. Source files use relative paths from the Visual Studio project file location. To fix the error, either re-import the missing files or maintain the original SDK folder structure relative to where your .vcxproj file is located. Simply adjusting the include directories in project properties will not resolve paths for source files.

*Tags: `build`, `windows`, `sdk`, `reference`*

---

## How can I display text in an After Effects plugin render buffer?

After Effects' API doesn't offer built-in text generating functions for the render buffer (unlike the UI buffer which has the drawbot suite). However, you can fill the render buffer using OS-native text tools: use GDI+ on Windows or Quartz on macOS to generate text in a native OS buffer, then copy that content back to After Effects' render buffer.

*Tags: `ui`, `windows`, `macos`, `render-loop`, `output-rect`*

---

## What language and tools are needed to create After Effects plugins?

After Effects plugins are written in C/C++. While you can work with other languages, all calls to the AE API must be in C/C++. The SDK is shipped with Visual Studio project files for Windows and Xcode project files for Mac. Effect controls windows and composition window overlay graphics are created using the DrawBot suite supplied by the AE SDK.

*Tags: `sdk`, `c/c++`, `windows`, `macos`, `ui`, `build`*

---

## How can a C++ plugin read the time slider position when its own UI window is displayed and plugin parameters haven't changed?

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

*Tags: `aegp`, `ui`, `params`, `windows`, `debugging`*

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

## Why is my After Effects plugin not showing up in the Effects menu after building it?

The plugin's build output location must be in a directory where After Effects looks for plugins. AE loads plugins from the Plug-ins folder in its installation directory, searching up to 5 folders deep. Either build directly into a subfolder of AE's Plug-ins directory, or create a shortcut in that directory pointing to your build output. Using a shortcut is recommended as it allows testing across multiple AE versions without updating build preferences when installing new AE versions.

*Tags: `deployment`, `build`, `debugging`, `windows`, `macos`*

---

## Why does my After Effects plugin fail to load with Error 126 when launched directly, but works when loaded from Visual Studio?

Error 126 when loading an AE plugin typically indicates a dependency issue with missing DLLs. When running the plugin from Visual Studio's debugger, the debugger's environment may provide DLL paths that aren't available when After Effects is launched directly. To diagnose the issue, build a simple test plugin that loads correctly in both contexts and have it report its home directory path. If testing on the same machine, the problem is likely a difference in the process's working directory or DLL search paths between debugger and direct execution. Ensure all dependent DLLs (like those required by the cairo library) are either in the plugin directory, in a standard system path, or that your plugin's search paths are properly configured for non-debugger execution.

*Tags: `debugging`, `deployment`, `windows`, `build`*

---

## How can I get the After Effects version string (e.g., 'After Effects CC 2017') in my plugin?

There is no direct SDK function to retrieve the version string. Instead, use PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH_W, wPath) to get the path to your plugin files, then decipher the AE version from the file path. Alternatively, look for the 'Presets' folder which is always in the same directory as AfterFX.exe. You can also check the Windows registry at HKEY_LOCAL_MACHINE\SOFTWARE\Adobe\After Effects\ or the macOS library at /Library/Preferences/com.Adobe.After Effects.paths.plist.

```cpp
A_UTF16Char wPath[AEFX_MAX_PATH];
PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH_W, wPath);
```

*Tags: `aegp`, `pipl`, `windows`, `macos`, `sdk`*

---

## How can I debug intermittent C++ exceptions that occur between PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_RENDER?

According to community expert shachar carmi, intermittent exceptions at the same memory address are likely caused by stack overflow or buffer overrun. The recommended debugging strategy is to use binary search: disable half your code and test to see if the problem manifests, then progressively narrow down the suspect code by testing the remaining halves. This methodical approach helps isolate the problematic section without relying on traditional debuggers, which may not be effective for these types of memory issues.

*Tags: `debugging`, `memory`, `render-loop`, `windows`*

---

## How can you detect which mouse button (left, middle, or right) was pressed in an After Effects plugin?

You cannot rely on the AeData structure to determine which mouse button was pressed. Instead, you must use the operating system API. For Windows, use the GetKeyState() function with virtual key codes: VK_LBUTTON for left button, VK_MBUTTON for middle button, and VK_RBUTTON for right button. Check if the result masked with 0x100 is non-zero to determine if the button is currently pressed.

```cpp
case button::left: return (GetKeyState(VK_LBUTTON) & 0x100) != 0;
case button::middle: return (GetKeyState(VK_MBUTTON) & 0x100) != 0;
case button::right: return (GetKeyState(VK_RBUTTON) & 0x100) != 0;
```

*Tags: `ui`, `windows`, `sdk`, `aegp`*

---

## How do I embed an image resource in a Windows .aex plugin file?

Add your image file as an image resource or generic binary resource from the Visual Studio File menu. You will get a custom identifier for this resource. Then use standard Windows resource functions to load it. Example code: HRSRC myResource = ::FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA); HGLOBAL myResourceData = ::LoadResource(NULL, myResource); void* pMyBinaryData = ::LockResource(myResourceData);

```cpp
HRSRC myResource = ::FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA);
HGLOBAL myResourceData = ::LoadResource(NULL, myResource);
void* pMyBinaryData = ::LockResource(myResourceData);
```

*Tags: `windows`, `build`, `params`, `ui`, `deployment`*

---

## How do you prevent sequence data from being lost during the flatten call when saving projects?

In After Effects versions before CC2017, you need to purge unwanted sequence data during the flatten call and reconstruct it during resetup. Starting with CC2017 and later, a new API call allows you to keep the original sequence data while delivering a flattened copy, preventing data loss. However, this new call is currently only implemented for UI event calls and not for copy, duplicate, or save operations. Eventually it will replace the older flatten call entirely.

*Tags: `sequence-data`, `aegp`, `params`, `macos`, `windows`*

---

## Can After Effects plugins be compiled in Release mode, and what are the runtime linking requirements?

Yes, After Effects plugins should be compiled in Release mode for both performance and deployment ease. There are four C++ runtime linking options: MT (multithreaded static), MD (multithreaded DLL), MTd (multithreaded debug static), and MDd (multithreaded debug DLL). Adobe recommends using MD or MDd because there is a limit to how many MT-compiled plugins After Effects can load simultaneously. In Release mode, use either MD or MT; in Debug mode, use either MDd or MTd. There is no noticeable performance difference between MD and MT variants.

*Tags: `build`, `windows`, `debugging`, `deployment`*

---

## Why won't my After Effects plugin compile in Release mode when using a third-party library compiled in Release?

If a plugin won't compile in Release mode, check the project settings for rogue '_DEBUG' macro definitions in the Preprocessor Definitions. Also inspect individual file preferences, as some files may have their own compile settings and additional macros that force Debug mode compilation. Remove any '_DEBUG' definitions that shouldn't be present in Release builds.

*Tags: `build`, `debugging`, `windows`*

---

## How can I debug iterate_generic functions properly in Visual Studio without erratic step-over behavior?

When debugging iterate_generic functions in Visual Studio, hit the breakpoint for the first time, then disable the breakpoint before stepping line by line. This prevents the debugger from jumping between threads (BEE WorkQueue and system DLLs) and allows normal line-by-line stepping.

*Tags: `debugging`, `iterate-generic`, `threading`, `windows`*

---

## What is the difference between .plugin and .aex file formats for After Effects plugins?

On Windows, After Effects plugins are built as .aex files. On macOS, plugins are built as .plugin files, which are actually bundles (not single files). Both formats are correct for their respective platforms and will work when built correctly according to the SDK instructions.

*Tags: `macos`, `windows`, `build`, `deployment`*

---

## What steps are necessary to convert an After Effects plugin from Windows to Mac?

If your C++ plugin code doesn't use Windows-specific native APIs or special library calls, porting to Mac should be straightforward—essentially just recompiling after adding all relevant project files. You'll need to use Xcode as your development environment on Mac instead of Visual Studio. Start from the Mac sample project (just as you did with the Windows sample project), but the plugin C/C++ code itself should require minimal to no changes if you've adhered to cross-platform APIs and the After Effects SDK.

*Tags: `cross-platform`, `windows`, `macos`, `build`, `deployment`*

---

## What does the PF_OutFlag2_SUPPORTS_QUERY_DYNAMIC_FLAGS error mean and how do you fix it?

The error 'effect cannot change non-dynamic flag bits during PF_Cmd_QUERY_DYNAMIC_FLAGS' occurs when there's a mismatch between the PiPL (Plug-In Property List) resource and the PF_Cmd_GLOBAL_SETUP implementation. The PF_OutFlag2_SUPPORTS_QUERY_DYNAMIC_FLAGS flag must be set consistently in both the PiPL and during PF_Cmd_GLOBAL_SETUP. On macOS, you can edit PiPL resources using Resorcerer with the PiPL template from the SDK, or use DeRez to create a .r file. The behaviors indicated during global setup must match those in the PiPL exactly. On Windows, ensure the .rc file is regenerated by cleaning and rebuilding the project.

*Tags: `pipl`, `macos`, `windows`, `build`, `debugging`*

---

## How do I port an AEIO plugin with a user options dialog from Windows to macOS?

To port an AEIO_UserOptionsDialog from Windows to macOS, you need to use native macOS UI frameworks instead of Windows dialogs. For Carbon implementation, refer to the Adobe forums thread at https://forums.adobe.com/thread/559946?tstart=0 which contains code examples and links for Cocoa. Alternatively, you can use AEGP_ExecuteScript() to open a dialog via JavaScript, which works cross-platform on both Windows and macOS without requiring separate implementations.

*Tags: `aeio`, `ui`, `cross-platform`, `macos`, `windows`*

---

## How do you set up debugging for an After Effects SDK example project in Visual Studio?

In the project properties window, set the command for debugging by providing the path to the afterfx.exe file. You can ignore any symbols-related warnings that appear.

*Tags: `debugging`, `build`, `windows`, `sdk`*

---

## What causes an 'invalid filter' error when loading a compiled After Effects plugin?

The invalid filter error is usually a dependency issue. Check your plug-in using Dependency Walker to verify all required components are present on your machine. This error also commonly occurs when you compile your plug-in in Debug mode and try to run it on a non-development machine where the debug DLLs don't exist. Compiling in Release mode resolves this issue.

*Tags: `debugging`, `build`, `windows`, `deployment`*

---

## How can I validate image files before importing them into After Effects through a plugin?

There are two approaches: (1) Check the file using OS tools—both GDI+ (Windows) and Quartz (macOS) offer tools for reading image files in various formats to validate them before importing. (2) Use After Effects to check—use AEGP_StartQuietErrors to suppress user-visible errors, perform a test render to detect errors silently, then call AEGP_EndQuietErrors when done. Based on the result, you can decide whether to dispose of the image or notify the user.

*Tags: `aegp`, `image`, `validation`, `windows`, `macos`, `debugging`*

---

## Why does an After Effects effect plugin crash when dragging camera geometry values instead of typing them?

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

*Tags: `aegp`, `debugging`, `camera`, `macos`, `windows`*

---

## How can I allocate memory for a Windows GDI+ Bitmap object in an After Effects plugin without using new/delete to avoid crashes?

Use placement new syntax to construct an object in pre-allocated memory. First allocate memory using host_new_handle(), lock it with host_lock_handle(), then use the placement new operator: `Bitmap *tmpImg = new(memV) Bitmap(width, height, bitmap_bytes_per_rowL, PixelFormat32bppPARGB, reinterpret_cast<BYTE *>(bitmap_dataP));` This avoids mixing memory allocation strategies between the host and the plugin, which can cause crashes especially in Windows release builds.

```cpp
PF_Handle bmpH = suites.HandleSuite1()->host_new_handle(sizeof(Bitmap));
void *memV = suites.HandleSuite1()->host_lock_handle(bmpH);
Bitmap *tmpImg = new(memV) Bitmap(width, height, bitmap_bytes_per_rowL, PixelFormat32bppPARGB, reinterpret_cast<BYTE *>(bitmap_dataP));
```

*Tags: `memory`, `windows`, `aegp`, `debugging`*

---

## What does the 'invalid filter 25::3' error mean when loading SDK examples in After Effects?

The 'invalid filter 25::3' error typically indicates a dependency issue with the compiled plugin. This often occurs when the Visual Studio compiler version or runtime libraries don't match what After Effects expects. For CS5 and above, plugins must be compiled as x64 (64-bit), not x86 (32-bit), as After Effects no longer loads x86 plugins. Use Dependency Walker to verify all dependencies are present and correct. Additionally, ensure you have the correct Visual C++ runtime libraries installed, such as MSVCR100D.DLL from the Microsoft Visual C++ Redistributable package.

*Tags: `sdk`, `build`, `windows`, `debugging`, `pipl`*

---

## What is Dependency Walker and how is it useful for debugging After Effects plugin issues?

Dependency Walker is a diagnostic tool that analyzes executable files and their dependencies. For After Effects plugin development, it helps identify missing or mismatched DLL files and runtime libraries that can cause plugins to fail loading. By running your compiled plugin through Dependency Walker, you can see all unresolved dependencies, which helps troubleshoot errors like 'invalid filter 25::3'. The tool is available at http://www.dependencywalker.com/

*Tags: `tool`, `debugging`, `windows`, `build`*

---

## Where can I download the Visual C++ runtime libraries needed for After Effects plugins?

Microsoft provides the Visual C++ Redistributable package at http://www.microsoft.com/en-us/download/details.aspx?id=8328 which contains essential runtime libraries like MSVCR100D.DLL required for plugins to run properly in After Effects.

*Tags: `build`, `windows`, `resource`*

---

## How can I draw anti-aliased shapes and text directly into an After Effects effect's output buffer?

DrawBot is not suitable for rendering into the output buffer—it only operates on AE's internal interface graphics buffers. Instead, use OS-native drawing tools: on macOS, use Quartz (Core Graphics) APIs like CGBitmapContextCreate(), CGContextShowTextAtPoint(), and CGContextDrawPath(); on Windows, use GDI+ or equivalent APIs. Create an OS graphics context in memory allocated via the Memory Suite, draw your shapes and text to that context, then copy the pixel data from the OS buffer to the effect's output buffer by locking the memory handle and iterating through pixels. See the CCU (Custom Color UI) sample code in the SDK for an example of accessing the output buffer directly in RAM.

```cpp
osBufferBaseAddress = suites.HandleSuite1()->host_lock_handle(osBufferMemHandle);
// Copy pixel data from OS context to output buffer
// Then unlock and free the OS graphics context
```

*Tags: `output-rect`, `macos`, `windows`, `drawbot`, `reference`*

---

## What is the difference between DrawBot and OS drawing libraries for After Effects plugin development?

DrawBot is designed exclusively for drawing custom UI overlays on AE's interface in CS5 and later. It provides built-in drawing tools and operates only on AE's internal interface graphics buffers. OS drawing libraries (Quartz on macOS, GDI+ on Windows) are general-purpose graphics APIs used to render into effect output images. If you need to draw shapes, lines, or text into an effect's output buffer rather than a UI overlay, you must use OS-native drawing tools combined with the Memory Suite to manage the graphics context, not DrawBot.

*Tags: `drawbot`, `ui`, `macos`, `windows`, `reference`*

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

## Are After Effects plugins cross-platform compatible between Mac and Windows?

Yes, plugins developed for After Effects are generally cross-platform compatible. Over 99% of the After Effects API is completely cross-platform without special notes. The main differences come when using OS-level APIs (such as opening OS-level windows for options). For core functionality like handling buffers, pixels, RAM, and API suites, the same code can be used on both platforms. However, there are several platform-specific considerations: Windows defines 'long' as 32-bit while Mac defines it as 64-bit (best practice is to use Adobe API types), Windows uses backslashes in paths while Mac uses forward slashes, Windows uses UTF-16 for file operations while Mac uses UTF-8, there are byte order issues for image manipulation, and the build outputs differ (Visual Studio generates a single file while Xcode generates a bundle). For best results, have developers familiar with each platform handle their respective ports.

*Tags: `cross-platform`, `macos`, `windows`, `api`*

---

## What type-definition practices should be used for cross-platform After Effects plugin development?

When developing cross-platform After Effects plugins, it is best practice to use Adobe API-defined types rather than native OS types. This is because Adobe has carefully designed their type definitions to handle platform differences correctly. For example, the native 'long' type is defined as 32-bit on Windows but 64-bit on Mac, which can cause compatibility issues. By using Adobe's API type definitions throughout your code, you avoid these pitfalls and ensure consistent behavior across platforms.

*Tags: `cross-platform`, `api`, `macos`, `windows`*

---

## How should I port a plug-in from CS4 to CS6 when PF_Context's cgrafptr and PF_GET_CGRAF_DATA macro are removed?

The drawing mechanism changed between CS4 and CS5. Direct drawing into a context is no longer supported. Instead, you must use the DrawBot suite to draw. The quickest conversion approach is to wrap the DrawBot commands under your old drawing command names. This change also makes the code cross-platform compatible.

*Tags: `mfr`, `ui`, `windows`, `cross-platform`, `porting`*

---

## How can I catch mouse wheel events in After Effects plugins?

After Effects does not supply effects with mouse wheel events through its standard API. The SDK only provides clicks, drags, and cursor enter/exit events for UI regions. If you need mouse wheel support on Windows, you can use the HWND descriptor approach, but this is only available for AEGP Palette plugins, not standard effects.

*Tags: `ui`, `aegp`, `windows`, `debugging`*

---

## How do I resolve error 126 when users try to load my After Effects plugin?

Error 126 typically indicates a dependency issue. Use Dependency Walker to analyze your plugin and identify missing or problematic DLL dependencies. If your plugin requires MSVCR100D.DLL (debug runtime), change your C/C++ code generation runtime library setting: use /MT to statically link the runtime library into your plugin (no external DLLs needed, larger file size), or /MD to use the release version of MSVCR100 (requires distributing the DLL). Test your plugin on a non-developer machine using Dependency Walker to ensure no problematic dependencies remain. Avoid requiring debug runtime libraries in production builds.

*Tags: `windows`, `deployment`, `build`, `debugging`*

---

## What tool can I use to check for dependency issues in my After Effects plugin?

Dependency Walker is a useful tool for analyzing plugin dependencies on Windows. It can identify external DLL requirements and help diagnose loading errors. Run it on both 32-bit and 64-bit versions of your plugin to ensure compatibility across architectures.

*Tags: `windows`, `debugging`, `tool`, `deployment`*

---

## Can Visual C++ Express be used to compile 64-bit After Effects plugins?

Yes, Visual C++ Express can be used to compile 64-bit plugins by downloading the Windows SDK and using it as the compiler. A patch is available that enables 64-bit compilation support in VC2008 Express with Windows 7 SDK.

*Tags: `build`, `windows`, `sdk`, `64-bit`*

---

## What is a guide for setting up Visual C++ Express to compile 64-bit applications?

A patch and setup guide for VC2008 Express 64-bit compilation is available at http://www.cppblog.com/xcpp/archive/2009/09/09/vc2008express_64bit_win7sdk.html, which explains how to configure Visual C++ Express with the Windows SDK to enable 64-bit plugin development.

*Tags: `build`, `windows`, `reference`, `tool`*

---

## How is the PiPL resource version number calculated in After Effects plugins?

The resource version is calculated using the PF_VERSION macro with bit shifting. The formula is: RESOURCE_VERSION = MAJOR_VERSION * 524288 + MINOR_VERSION * 32768 + BUG_VERSION * 2048 + STAGE_VERSION * 512 + BUILD_VERSION. In Visual Studio, you can hover over the enum value to see its calculated result and copy it to the resource file. On Windows, delete the .rc file before rebuilding to ensure a new resource version is generated when changes are made.

```cpp
out_data->my_version = PF_VERSION(MAJOR_VERSION, MINOR_VERSION, BUG_VERSION, STAGE_VERSION, BUILD_VERSION);
```

*Tags: `pipl`, `build`, `windows`*

---

## Why can't VC++ 2008/2010 Express open After Effects SDK example projects?

The After Effects CS5 SDK example projects require 64-bit compiler configurations, which are not checked by default in the Visual Studio installer. You need to install the 64-bit compiling options during Visual Studio setup. Additionally, ensure the Windows SDK is installed and that Visual Studio is configured to look in the correct SDK paths. The Premiere Pro CS5 SDK works because it includes both 32-bit and 64-bit configurations to support 32-bit applications like Encore CS5.

*Tags: `build`, `windows`, `sdk`, `debugging`*

---

## Why won't my After Effects plugin built with SDK 7.0 load on Mac CS4 when it works fine on Windows?

The issue is related to architecture support differences between SDK versions. AE7 was the last version to run on PPC processors; on Intel chips it ran emulated. CS3 was the first to run native on Intel chips and SDK 8.0 (CS3) was the first SDK to support plugins running native on Intel chipsets (and PPC). The CS3 SDK offers backwards compatibility to AE7, allowing plugins to work across AE7, CS3, and CS4 on both PC and Mac. In contrast, the CS4 SDK is not backwards compatible. To achieve cross-version compatibility on Mac, you should check the resource file declarations in the CS3 SDK samples, which contain separate entry point declarations for Intel and PPC architectures.

*Tags: `macos`, `windows`, `sdk`, `build`, `cross-platform`, `pipl`*

---

## How do you compile the Panelator sample to a release version that runs on non-developer machines?

When compiling Panelator for release, change the runtime library from /MD to /MT to make the plugin independent of external MSVC runtime DLLs. Additionally, ensure all supporting code files in the 'supporting code' folder have consistent compiler settings (not mixed MD/MDd). If using Visual Studio 2008 instead of the SDK-intended VS2005, be aware that 2008 uses MSVCR90.DLL instead of MSVCR80.DLL, which can cause compatibility issues. Using /MT will statically link all libraries into the plugin, making it dependent only on KERNEL32.DLL. Use Dependency Walker to verify the plugin's dependencies before deployment.

*Tags: `build`, `windows`, `deployment`, `debugging`*

---
