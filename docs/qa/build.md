# Q&A: build

**151 entries** tagged with `build`.

---

## Is there a skeleton/starter project for building an AE plugin with Vulkan GPU rendering?

Yes — AE_Skeleton_Vulkan by Eric CPFX (emlcpfx) is a complete CMake-based skeleton that incorporates the Vulkan SDK into an AE plugin. It also includes build scripts and a sample script for signing, notarizing, and stapling on Mac. Repository: https://github.com/emlcpfx/AE_Skeleton_Vulkan

*Source: curated · Tags: `vulkan`, `gpu`, `cmake`, `build`, `macos`, `code-signing`, `skeleton`*

---

## What was required to update a simple plugin for MFR compatibility?

For simple plugins without sequence data or fancy features, MFR compatibility mainly requires rebuilding the plugin. No major refactoring is necessary if the plugin doesn't use advanced features.

*Tags: `mfr`, `build`*

---

## Is it necessary to build on Apple Silicon hardware to support it?

It is not strictly required to build on Apple Silicon hardware, but it is best practice to test on the actual platform. Purchasing an M1 Mac mini is a practical and cost-effective way to validate Apple Silicon support.

*Tags: `apple-silicon`, `macos`, `build`*

---

## What is required to support M1 Macs when including external libraries like OpenCV in an After Effects plugin?

When building for M1 with external libraries, ensure the library is compiled with appropriate Mac support options. If using dynamic libraries (dylib), you need to embed them in the plugin file and specify rpath in Xcode. Static libraries (.a files) should be linked directly. The Missing Entry Point error typically occurs when the plugin is applied, not during loading, and may indicate a PiPL configuration issue rather than a linking problem. It's recommended to test on both M1 and x86 Mac architectures to isolate platform-specific issues, as the problem may be related to project configuration rather than the external library itself.

*Tags: `apple-silicon`, `macos`, `build`, `pipl`, `deployment`*

---

## Should external libraries in After Effects plugins be linked as static or dynamic libraries?

Both approaches are valid. Static libraries (.a files) are linked directly during compilation. Dynamic libraries (dylib) can be used but require embedding in the plugin file and specifying rpath in Xcode build settings to ensure proper loading at runtime.

*Tags: `build`, `deployment`*

---

## Has anyone used metal cpp bindings instead of Objective-C for After Effects plugin development?

No direct answer was provided in the conversation. One participant acknowledged the question but did not provide experience or recommendations.

*Tags: `metal`, `macos`, `build`*

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

## How can I embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications using MoltenVK?

You can embed MoltenVK in the plugin's framework folder as a dynamic library with the JSON configuration in resources/vulkan. However, this path approach doesn't work reliably for plugin bundles. An alternative is to use xcframework to embed it as a static library inside the plugin binary, similar to what the scaleUp plugin does. If Vulkan fails to create instances when using the static library approach, verify that the Vulkan layer configuration and initialization is correct for the statically-linked library within the plugin bundle context.

*Tags: `vulkan`, `macos`, `deployment`, `build`, `cross-platform`*

---

## How can MoltenVK be embedded inside an After Effects plugin bundle to avoid conflicts with other MoltenVK applications?

MoltenVK can be embedded as a static library using xcframework within the plugin binary, similar to how the scaleUp plugin does it. However, this approach requires careful configuration of the Vulkan loader and MoltenVK ICD to ensure proper function pointer resolution and layer/extension loading. The main challenge is ensuring the Vulkan loader correctly locates the MoltenVK ICD when statically linked, as opposed to using the standard /etc/vulkan or /usr/local installation paths. Testing should verify that the Vulkan loader is correctly configured for the static linking approach.

*Tags: `macos`, `vulkan`, `build`, `deployment`, `cross-platform`*

---

## How do you set up MoltenVK compatibility with Vulkan 1.2 on macOS for After Effects plugins?

Get at least the October update to get a MoltenVK compatible with Vulkan 1.2. For debug builds, create a target using ICD in environment with dylib linking, add "VK_KHR_portability_subset" to instance extensions (debug only for now until MoltenVK uses it), and include debugging layers. For release builds, link to frameworks only as static libraries to work within Vulkan 1.2 limitations.

*Tags: `vulkan`, `macos`, `apple-silicon`, `build`, `debugging`*

---

## What tools should be used for matrix operations and inversion when working with camera matrices in After Effects plugins?

The GLM (OpenGL Mathematics) library is recommended for performing matrix operations and inversions. It provides robust mathematical functions for handling matrix transformations needed when converting and manipulating camera coordinate systems.

*Tags: `aegp`, `opengl`, `matrix-conversion`, `camera`, `build`*

---

## What technology is used to bridge Python with After Effects for plugin development?

A Python/AE/C++ bridge was built to enable creation of Python plugins. The bridge is stable and well-developed but still being refined for cleaner implementation.

*Tags: `scripting`, `build`, `cross-platform`*

---

## Why does a plugin built for any architecture show a compatibility warning in After Effects even though it's not running in Rosetta mode?

The conversation identifies this as an issue but does not provide a resolution. The user reports that ArchiCheck confirms the plugin is built for any architecture, yet After Effects displays a compatibility warning despite not running in Rosetta mode.

*Tags: `macos`, `apple-silicon`, `build`*

---

## What is the correct architecture string to use when building for x64 on Apple Silicon?

The correct string for x64 is x86_64, not x64. If you put an unrecognized string, it builds for your native architecture instead.

*Tags: `build`, `apple-silicon`, `macos`*

---

## What should be cleaned when experiencing build issues in Xcode?

Use Cmd+shift+k to clean the build folder, which cleans everything there is to clean. You can also reset After Effects preferences if needed.

*Tags: `build`, `macos`, `debugging`*

---

## Why might a plugin fail to load even though it builds successfully?

The plugin's PiPL resource needs to include ARM64 architecture support. If ARM64 is not added to the PiPL, the plugin won't load properly even if the build succeeds.

*Tags: `pipl`, `apple-silicon`, `build`*

---

## What causes an 'entry point not found' error on Windows after a plugin is loaded?

Two main possibilities: a missing dependency (such as a Visual C++ redistributable package) or an error in global setup. The issue may be resolved by installing the Visual C++ redistributable package. In debug mode, you can set a breakpoint on global setup to verify if it's being called.

*Tags: `windows`, `deployment`, `build`, `debugging`*

---

## Is it normal to see msvcp140d.dll when compiling with Visual Studio 2022 (v143)?

Yes, this is normal. The msvcp140d.dll appears in compilations with Visual Studio 2022 (v143). However, if the entry point is not found, it typically indicates a dependency issue rather than a problem with this DLL itself.

*Tags: `windows`, `build`, `deployment`*

---

## How can you improve memory management for AESDK objects in After Effects plugins?

Create C++ wrappers around AESDK objects to enable automatic memory management. There are already some C++ wrappers of portions of the SDK available on GitHub that can be used for this purpose.

*Tags: `memory`, `build`, `aegp`*

---

## What macOS and Xcode version compatibility issues occur when updating macOS?

When updating macOS (e.g., from macOS 13.x to 14), you are often forced to update Xcode to a compatible version. For example, updating to macOS 14.1.1 required updating to Xcode 15. This tight coupling between macOS and Xcode versions is more restrictive than Windows/Visual Studio, which is more flexible about version compatibility.

*Tags: `macos`, `build`, `deployment`*

---

## What development strategy should be used to test plugin compatibility across macOS versions?

The best approach is to maintain separate development and testing machines: one dev Mac with the target macOS version and a separate test Mac updated to the latest version. This allows developers to confirm that products run on new OS versions and debug any version-specific bugs that only occur on the latest release. However, this is an expensive option.

*Tags: `macos`, `deployment`, `debugging`, `build`*

---

## How do you get Xcode 14 working with After Effects plugin development?

Run the Xcode binary directly from Terminal using the path /Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode (replacing .app with the actual version name). Download Xcode 14.3.1 from https://developer.apple.com/download/all/

*Tags: `macos`, `build`, `debugging`*

---

## What causes the 'dlsym cannot find symbol xSDKExport' error when launching After Effects from Xcode 15?

The error is related to MediaCore taking a long time to load all symbols. This output from Xcode started appearing after upgrading to Xcode 15. It may be related to symbol loading options being enabled, similar to a Visual Studio issue where selecting 'load symbols' option causes very long loading times.

```cpp
dlsym cannot find symbol xSDKExport in CFBundle 0x60000064ae60 </Applications/Adobe After Effects 2024/Adobe After Effects 2024.app/Contents/PlugIns/(MediaCore)/PlayerMediaCore.bundle> (bundle, loaded): dlsym(0xa68b4e61, xSDKExport): symbol not found
```

*Tags: `macos`, `debugging`, `build`*

---

## Is there a 'load symbols' option in the debugger that could be slowing down After Effects launch times?

Yes, there is a 'load symbols' option that can be selected in Visual Studio (and likely other debuggers) which causes very long loading times when After Effects is launched. This option can typically be disabled to speed up the launch process.

*Tags: `debugging`, `build`*

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

## Is it possible to include multiple PiPLs in the same file?

Yes, it is possible to include multiple plug-ins (both AEGPs and effects) in the same file using multiple PiPLs, but it is not recommended. If there are PiPLs for both AEGPs and effects in the same file, the AEGPs must come first. No other hosts, not even Premiere Pro, support multiple PiPLs pointing to multiple effects within the same .dll or code fragment. The recommendation is to use one PiPL and one plug-in per code fragment.

*Tags: `pipl`, `aegp`, `premiere`, `build`*

---

## Should I manually allocate memory in effect plugins or follow the sample plugins' approach?

You should follow what the sample plugins are doing rather than manually allocating memory. Manual allocation is problematic because After Effects won't know whether to use delete[] or free() for deallocation, and AE won't be able to track memory usage if you allocate it yourself.

*Tags: `memory`, `plugin`, `build`*

---

## Has anyone successfully compiled an After Effects plugin using cmake/ninja?

Yes, there are working examples available. The mobile-bungalow/after_effects_cmake repository provides a cmake setup based on vulkanator's work. Additionally, the virtualritz/after-effects project offers a cross-platform build system with its own PiPL compiler written in Rust as an alternative to using pipltool.exe.

*Tags: `build`, `cmake`, `pipl`, `cross-platform`*

---

## How do you compile .rc files on macOS for After Effects plugins?

On macOS, you can use the native macOS tool called Rez to compile .rc files. The virtualritz/after-effects project demonstrates this approach as part of its build system. The Rust PiPL compiler from that same project can also handle resource compilation as part of a cross-platform build setup.

*Tags: `build`, `macos`, `pipl`, `deployment`*

---

## How can you achieve incremental compilation for After Effects plugins to improve build times?

Using the Ninja generator with cmake can provide faster incremental compilation. The xcodebuild CLI tool does not appear to support incremental compilation effectively, resulting in full rebuilds taking around 50 seconds. Switching from Xcode project generation to Ninja as the cmake generator should enable proper incremental compilation.

*Tags: `build`, `cmake`, `macos`, `performance`*

---

## Is using f32 (32-bit float) data type safe and well-supported in After Effects plugin development?

Yes, f32 support is safe and essential for After Effects plugins, particularly those integrating 3D renderers. This has been validated through practical experience with production plugins like AtomKraft for AE and other Rust-based AE integrations that have run in production for extended periods.

*Tags: `build`, `deployment`, `gpu`, `threading`*

---

## How can a plugin require AVX2 support or handle the AVX2 cutoff in After Effects?

After Effects has required AVX2 since version 2023. Given that Steam hardware survey shows 98.25% support for AVX and 96.30% support for AVX2 (with growing adoption rates), targeting AVX2 is practical for modern plugins. Rather than wrapping every dispatch with if(HasAVX()) checks, you can compile your plugin with AVX2 as a baseline requirement, allowing you to leverage AVX2 optimizations throughout your codebase without conditional branching. This approach can potentially double performance compared to targeting x86-64-v2 (up to SSSE3).

*Tags: `build`, `deployment`, `windows`, `performance`*

---

## How can a plugin support AVX2 requirements given After Effects 2023+ requires it, and should developers worry about older CPU compatibility?

After Effects has required AVX2 since version 2023. Based on Steam hardware survey data, 96.30% of systems support AVX2 (growing 1.63% per month) and 98.25% support AVX (growing 0.77% per month), though this may not directly overlap with video editing users. Most developers using After Effects in 2025, even older versions, are unlikely to encounter machines without AVX2 support. For selective CPU feature support, ISPC (Intel SPMD Program Compiler) can automatically compile multiple versions and switch between them at runtime.

*Tags: `build`, `cpu`, `deployment`, `cross-platform`, `windows`*

---

## What is the minimal macOS SDK version required for After Effects plugins?

The minimal macOS SDK requirement was not specified in the conversation. The question was asked but not answered.

*Tags: `macos`, `deployment`, `build`*

---

## What is the nature of the issues reported by the Red Giant team with the pre-release version?

The issues are related to code signature validation. Something is getting corrupted somewhere and the validation process is causing the problem. Red Giant is in contact with Adobe and the issue is being investigated, but a full picture is still being determined.

*Tags: `deployment`, `build`, `debugging`*

---

## What macOS SDK was used to compile the plugins?

The conversation discusses whether plugins were compiled with a newer macOS SDK with an updated version, noting differences between Adobe's ExporterAIFF.plugin between versions 25.1 and 25.2, but no definitive answer is provided about which specific SDK version was used.

*Tags: `macos`, `build`, `deployment`*

---

## Why are some plugin files becoming 0 bytes after upgrading to After Effects 2025.2?

Multiple plugins were found to be 0 bytes (emptied) after upgrading to AE 2025.2, while released versions work properly. This appears to be a bug in the 2025.2 release that corrupts plugin files in the MediaCore folder during upgrade, though the root cause was not determined in the conversation.

*Tags: `deployment`, `macos`, `windows`, `build`*

---

## Why does SmartRender never get called in Release builds even when PreRender returns PF_Err_NONE?

The issue was caused by a mismatch between the SDK code version used for building (2025.2) and the SDK headers (2023). The debug configuration worked fine because it didn't have this version mismatch. Ensure that both the SDK code and headers are from the same version when building.

*Tags: `smartfx`, `smart-render`, `debugging`, `build`, `release`*

---

## How can installer leftovers from previous versions break code signing on macOS?

When an installer overwrites files instead of properly uninstalling, leftover files from the previous version remain in the package. Any extra files in a signed package will break the code signature. For example, if version 1 has file X and version 2 removes it, but the installer leaves file X behind, the code signature breaks. The solution is to completely uninstall and remove all files before installing the new version, rather than just overwriting files.

*Tags: `macos`, `deployment`, `build`, `codesign`*

---

## Why does After Effects crash when launched from lldb with certain launch.json configurations?

The crash was caused by missing the "sourceLanguages": ["rust"] line in the launch.json configuration. Adding this line to the debugger launch configuration prevents the crash from occurring.

```cpp
"sourceLanguages": ["rust"]
```

*Tags: `debugging`, `build`, `windows`, `macos`*

---

## What is the first version of After Effects that supports the new plugin format without PIPL?

Based on the conversation, support started with After Effects 2018, though the documentation still references it as a future release at the time of discussion.

*Tags: `pipl`, `deployment`, `build`*

---

## Is a PiPL resource still needed to register After Effects plugins, or can plugins be registered with just the URL info function?

PiPL is still needed even with the PF_Register_effect_ext2 function added in version 23. While you can overwrite some PiPL values in code, the resource file cannot be completely eliminated. After Effects still cannot find effects without a PiPL/rsrc file, though Adobe may update this in the future.

*Tags: `pipl`, `build`, `deployment`, `plugin-registration`*

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

## Why does a plugin work in the Plugins folder on Mac but not in the Mediacore folder on Mac AE 26.0.0?

This is a known compatibility issue with AE 26.0.0 on Mac. Several developers have reported the same problem. Gaussian Splat and other plugin developers have issued updates to address AE 26 compatibility, suggesting a breaking change in how AE 26 handles plugins in the Mediacore folder on macOS.

*Tags: `macos`, `deployment`, `build`*

---

## How do you build an After Effects plugin for Apple Silicon (M1) native support?

To build for M1 native (non-Rosetta), you need to: 1) Set the appropriate build configuration for Apple Silicon architecture, and 2) Add CodeMacARM64 {"EffectMain"} entry in the PiPL resource file. Without the PiPL entry, the plugin won't be recognized as native ARM64 compatible even if the binary is built correctly.

```cpp
CodeMacARM64 {"EffectMain"}
```

*Tags: `macos`, `apple-silicon`, `build`, `pipl`*

---

## How do you debug a Premiere plugin from Xcode and load debug symbols?

Ensure that you have 'Generate Symbols' set to 'Yes' in your Xcode scheme settings. If symbols are set to 'No', breakpoints will not work and Xcode will not be able to load the necessary debugging information for your Premiere plugin.

*Tags: `debugging`, `premiere`, `xcode`, `build`*

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

## How do you define ARGB_8u color space in globalSetup for After Effects plugins?

When using ARGB_8u as your color space, you need to properly define it in the globalSetup function. The specific definition depends on your plugin architecture and color space requirements.

*Tags: `mfr`, `params`, `build`*

---

## Can C++ be integrated as a backend for UXP plugins in Premiere Pro, or is IPC required for computational tasks?

There is a new hybrid UXP approach that combines HTML and C++ backends, though the exact implementation details were not fully explored in the conversation. This hybrid solution is currently in beta and represents a promising development for allowing computational tasks without requiring a separate service with IPC.

*Tags: `premiere`, `uxp`, `build`*

---

## Can you build After Effects plugins against older macOS SDKs using the latest Xcode?

James Whiffin reported that with the latest Xcode, there are compatibility issues when trying to build against older macOS SDKs (e.g., 10.10). This is a known limitation where newer Xcode versions may not support targeting legacy macOS versions.

*Tags: `macos`, `build`, `cross-platform`, `deployment`*

---

## How do you properly include external libraries like OpenCV in an After Effects plugin targeting Apple Silicon (M1)?

When including external libraries in AE plugins for M1, ensure you're linking against static libraries (.a files) rather than dynamic libraries. If using dynamic libraries, you may need to specify rpath in Xcode and embed the dylib in the plugin file. Verify that your PiPL file contains the correct entry point definition. If experiencing 'Missing Entry Point' errors when applying the effect, test on x86 Mac to isolate whether the issue is architecture-specific or a general PiPL configuration problem. Common causes include outdated project settings when migrating older projects to M1 support.

*Tags: `macos`, `apple-silicon`, `build`, `pipl`, `cross-platform`*

---

## How can I diagnose missing DLL dependencies in a Windows After Effects plugin?

Use Dependency Walker on Windows to check for missing DLLs and verify that all libraries and DLLs have the correct architecture (32-bit vs 64-bit) matching your plugin build. Also check system folders like System32 where driver-installed libraries like Vulkan may be located.

*Tags: `windows`, `debugging`, `build`, `deployment`*

---

## What should I verify when using external libraries like Vulkan in an After Effects plugin?

Ensure that external libraries are properly installed on the user's system (e.g., Vulkan drivers install libraries to System32), check for any transitive dependencies from licensing tools or other libraries, and verify that all linked DLLs match the architecture of your plugin build.

*Tags: `windows`, `vulkan`, `deployment`, `build`*

---

## How do you set up Vulkan development on macOS with MoltenVK compatibility?

To develop Vulkan plugins on macOS with MoltenVK: (1) Update to at least October release to get MoltenVK compatible with Vulkan 1.2. (2) For debug builds, create a target using ICD (Installable Client Driver) in environment variables and dylib linking, then add the "VK_KHR_portability_subset" instance extension (debug-only until MoltenVK uses it natively) along with debugging layers. (3) For release builds, link only to frameworks as static libraries to comply with Vulkan 1.2 limitations.

*Tags: `vulkan`, `macos`, `cross-platform`, `debugging`, `build`*

---

## What approach can be used to create Python-based After Effects plugins?

A Python/AE/C++ bridge can be built to enable Python plugin development for After Effects. This allows developers to write plugins in Python while maintaining compatibility with AE's native plugin system, though the implementation can be complex and require several months of development to achieve stability and polish.

*Tags: `scripting`, `build`, `cross-platform`, `aegp`*

---

## Has anyone ported the After Effects plugin build system to use CMake instead of Visual Studio and Xcode?

The question was asked but no one in the conversation provided a definitive answer or shared an existing CMake-based build system for AE plugins. This remains an open challenge in the community.

*Tags: `cmake`, `build`, `cross-platform`, `xcode`, `windows`*

---

## Is there an open-source After Effects plugin example that uses CMake for build system setup?

The aftereffects_spatial_media_plugins repository by Gorialis uses CMake for build configuration, allowing developers to set up the build system themselves rather than relying on platform-specific templates. This approach is appealing for creating new plugins with a more understandable build process. Repository: https://github.com/Gorialis/aftereffects_spatial_media_plugins/

*Tags: `cmake`, `build`, `open-source`, `reference`*

---

## What causes the 'Invalid Filter 25::3' error on macOS in After Effects?

The 'Invalid Filter 25::3' error on macOS can be caused by a mismatch between the Deployment Target set in Xcode and the macOS version running After Effects. If the Deployment Target is set higher than the OS version (e.g., setting Deployment Target to 12.3 when running on Big Sur 11.17.9), it will cause this error.

*Tags: `macos`, `debugging`, `build`, `deployment`*

---

## What is the minimum supported macOS version for After Effects plugin development?

High Sierra is a practical cutoff point for macOS support. Older versions like Sierra and earlier tend to have compatibility issues with many applications and libraries. Changing the macOS deployment target may not resolve underlying compatibility errors, and users may receive different error messages.

*Tags: `macos`, `build`, `deployment`, `cross-platform`*

---

## How should architecture settings be configured in Xcode build settings for ARM and Intel compatibility?

In Xcode build settings, there is an architecture option that allows you to set both ARM and Intel architectures, ARM only, or Intel only. These settings are separate from other build configuration options and should be reviewed carefully when troubleshooting cross-platform compatibility issues.

*Tags: `apple-silicon`, `macos`, `build`, `cross-platform`*

---

## How do you ensure ARM64 architecture support is properly configured in an After Effects plugin build?

You need to add ARM64 to your PiPL (Property List) file. Without this entry, the plugin may not build or load correctly for ARM64 architecture even if the C++ code and Xcode settings are correct.

*Tags: `pipl`, `macos`, `apple-silicon`, `build`*

---

## What is the correct string format for specifying x64 architecture in After Effects plugin build configurations?

The correct string for x64 architecture is 'x86_64', not 'x64'. Using an unrecognized architecture string may cause the build system to fall back to your native architecture instead of the intended target.

*Tags: `build`, `cross-platform`, `windows`, `macos`*

---

## How do you completely clean the Xcode build cache for After Effects plugin development?

Use Cmd+Shift+K in Xcode to clean the build folder. This cleans everything in the build cache. Additionally, resetting After Effects preferences can help resolve persistent build issues.

*Tags: `build`, `macos`, `debugging`*

---

## What causes 'entry point not found' errors on Windows when an After Effects plugin loads?

This error typically indicates a missing dependency, most commonly a missing Visual C++ redistributable package. Even if Visual Studio is installed on the development machine, the target machine may not have the required Visual C++ runtime libraries. On Windows 10, installing the Visual C++ 2022 redistributable (https://aka.ms/vs/17/release/vc_redist.x64.exe) often resolves the issue. Debugging can be done by setting a breakpoint on global setup in debug mode to verify if the plugin initialization is being reached.

*Tags: `windows`, `deployment`, `debugging`, `build`*

---

## Is it normal to see msvcp140d.dll when compiling an After Effects plugin with Visual Studio 2022 (v143)?

Yes, this is normal behavior. The msvcp140d.dll file (Visual C++ Debug Runtime) appears in the compiled output when using Visual Studio 2022. However, if you're encountering entry point errors, the issue is likely related to other missing dependencies or libraries that the plugin relies on, rather than the presence of this DLL.

*Tags: `windows`, `build`, `debugging`*

---

## What is a good approach to manage memory safety for After Effects SDK objects?

Create C++ wrapper classes around AESDK objects to enable automatic memory management. This approach leverages C++ features like constructors and destructors to ensure proper allocation and deallocation of SDK objects. Additionally, there are already existing C++ wrappers for portions of the After Effects SDK available on GitHub that can be referenced or reused for this purpose.

*Tags: `memory`, `open-source`, `reference`, `aegp`, `build`*

---

## What is the relationship between macOS and Xcode versions for After Effects plugin development?

When updating macOS, you often need to update Xcode to a compatible version. For example, updating from macOS 13.x to macOS 14 required updating from Xcode 14 to Xcode 15. Windows and Visual Studio are more flexible in this regard. The best practice for plugin development is to maintain separate development and testing machines with different OS versions to catch version-specific bugs and unexpected behavior, though this is an expensive option.

*Tags: `macos`, `build`, `debugging`, `cross-platform`, `windows`*

---

## What is the challenge of timing OS and development tool updates during plugin development?

Developers face a difficult choice between updating to new versions quickly (to catch bugs and unexpected behavior that only occur on the latest versions) and waiting to ensure stability and compatibility with their products. Rolling back an entire OS is impractical, so alternatives include waiting for the next patch of Xcode or maintaining separate machines for development and testing at different version levels.

*Tags: `build`, `debugging`, `deployment`, `macos`*

---

## How can you work around SDK issues when using Xcode 14 with After Effects plugin development?

Run the Xcode binary directly from Terminal instead of launching Xcode.app. The command is '/Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode' (replace with your Xcode version). You can download specific Xcode versions from https://developer.apple.com/download/all/. For quick access, add the /Contents/MacOS folder to your Finder sidebar, then right-click the binary, hold Alt/Option, click 'Copy as Pathname', paste into Terminal and execute. This takes 3-4 seconds versus the normal Spotlight launch method.

*Tags: `macos`, `debugging`, `build`, `xcode`*

---

## Why is After Effects taking a very long time to load in the debugger after upgrading to Xcode 15?

The issue may be related to symbol loading being enabled in your debugger settings. Check your Xcode debugger options and disable symbol loading if it's turned on, as this can significantly slow down AE launch times. In Visual Studio, you can stop symbol loading during debugging to speed this up.

*Tags: `debugging`, `macos`, `build`, `performance`*

---

## What are alternatives to Nuitka for compiling Python code in After Effects plugins, especially for resolving dependency conflicts on macOS?

Cython is a viable alternative to Nuitka for compiling Python code in AE plugins. It can be used to compile specific portions of Python code and has fewer dependency conflicts compared to Nuitka, particularly when dealing with libraries like NumPy on macOS (which can have issues with libopenblas.0.dylib).

*Tags: `scripting`, `macos`, `build`, `cross-platform`, `deployment`*

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

## How can you reverse-engineer and integrate a DLL into an After Effects plugin?

One approach is to use depends.exe to inspect the DLL's dependencies and exports, then reference existing samples (like the Illustrator sample) as a guide to understand the integration pattern. While not always a 1:1 match, samples can provide enough context to build your own wrapper. Modern approaches may involve directly linking to the DLL and providing a header file, but manual reverse-engineering through dependency analysis and sample-based implementation has proven effective.

*Tags: `debugging`, `build`, `windows`, `reference`*

---

## Why does adding another PiPL resource with a different name and matchname only show one plugin in After Effects?

When multiple PiPL resources are added to the same Xcode project, After Effects may only recognize one of them. This is typically due to incorrect PiPL configuration, resource ID conflicts, or the build process not properly including all PiPL resources in the final plugin binary. Ensure each PiPL has a unique resource ID, verify the matchname and name fields are correctly differentiated, and check that all PiPL resources are included in the target's Copy Bundle Resources build phase.

*Tags: `pipl`, `build`, `xcode`, `plugin-structure`*

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

## Is there an open-source reference for After Effects plugin development in Rust?

Yes, the virtualritz/after-effects repository (https://github.com/virtualritz/after-effects) provides Rust bindings and examples for After Effects plugin development. It includes practical code examples and API patterns learned from real plugin development, including handling 3D rendering integration and bit-depth detection. The crate was published as a result of production plugin development experience.

*Tags: `open-source`, `reference`, `build`, `tool`*

---

## Is there an open-source Rust crate for After Effects plugin development?

The virtualritz/after-effects repository (https://github.com/virtualritz/after-effects) is an open-source Rust crate for AE plugin development that includes f32 support. According to the repository owner, it was developed based on SDK examples, After Effects SDK mailing list discussions, and reverse-engineering from the AtomKraft for AE plugin. The crate was initially published after the author completed a year-long Rust plugin project integrating a 3D renderer, where f32 support proved essential.

*Tags: `open-source`, `rust`, `reference`, `build`*

---

## What is the minimum CPU instruction set support required for After Effects plugins in 2025?

After Effects has required AVX2 support since version 2023. Based on Steam hardware survey data, 98.25% of machines support AVX (growing 0.77% monthly) and 96.30% support AVX2 (growing 1.63% monthly). Even considering only video editing machines, the overlap suggests virtually no users in 2025 are on machines without AVX2 support. This makes it safe to target higher instruction sets like x86-64-v2 (up to SSSE3) or AVX2 for significant performance improvements without runtime checks.

*Tags: `build`, `cpu`, `performance`, `windows`*

---

## What tool can automatically compile multiple CPU instruction set versions and switch between them at runtime?

ISPC (Intel SPMD Program Compiler) can compile multiple versions of kernels with different CPU instruction sets (like AVX, AVX2) and automatically switch between them at runtime. This is useful for plugin developers who want to support a range of hardware while taking advantage of advanced SIMD instructions on capable processors.

*Tags: `build`, `cpu-optimization`, `cross-platform`, `performance`*

---

## Why does SmartRender never get called in Release mode even when PreRender returns PF_Err_NONE?

SmartRender may not be called if there's a version mismatch between the SDK code used for building and the SDK headers being compiled against. In the reported case, the plugin was being built with 2025.2 SDK code but using 2023 SDK headers, which caused SmartRender to not be invoked in Release builds while Debug builds worked fine. Ensure that the SDK version used for compilation matches the SDK headers included in the project.

*Tags: `smart-render`, `build`, `debugging`, `release`*

---

## Is there an open-source Rust binding for After Effects plugin development?

The virtualritz/after-effects repository on GitHub provides Rust bindings for After Effects plugin development. This project enables developers to write AE plugins using Rust instead of C++.

*Tags: `rust`, `open-source`, `reference`, `build`, `cross-platform`*

---

## How can you set up debugging for After Effects plugins in VSCode on macOS?

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

*Tags: `debugging`, `macos`, `build`*

---

## What causes code signature breaks in After Effects plugins on macOS?

Code signature breaks can occur when the plugin installer doesn't properly clean up old files before installing new versions. If v1 has a file that is removed in v2, an installer that overwrites rather than fully uninstalls will leave the old file in place, breaking the code signature. Modifying any file in a signed package (even adding a dummy.txt) will break the signature. The solution is to ensure complete removal of all old files before installing the new plugin version, not just overwriting existing files.

*Tags: `macos`, `deployment`, `build`*

---

## What is the relationship between After Effects build tools (Xcode) and plugin code signing issues?

After Effects being built with the latest Xcode version has introduced stricter code signature checks that were not enforced in previous builds. Apple's new Xcode performs code tampering detection differently, causing signature validation failures for plugins that previously loaded without issue. This is a macOS-specific binary signing problem, separate from CEP or ZXP signing issues.

*Tags: `macos`, `build`, `deployment`*

---

## Why does launching After Effects 2024/2025 from LLDB with certain launch configurations cause crashes?

The crash was traced to the launch.json configuration. Adding the line `"sourceLanguages": ["rust"]` to the launch.json prevents the crash from occurring when launching AE through LLDB. The exact mechanism is unclear, but this configuration change resolved the issue that was happening with both the user's and @fad's basic launch/tasks.json setup, even when mediacore was empty.

```cpp
"sourceLanguages": ["rust"]
```

*Tags: `debugging`, `build`, `macos`, `windows`*

---

## Can multiple effects be registered within a single DLL for After Effects plugins?

Yes, according to Alex Bizeau from maxon, it is possible to call multiple register effect functions in one DLL. This approach could be used to create a bootstrapper similar to what was done for OFX, allowing a single DLL to load multiple effects and reduce code duplication across OFX, After Effects, and AVX plugins.

*Tags: `pipl`, `aegp`, `plugin-architecture`, `deployment`, `build`*

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

## Is there an open-source skeleton project for Vulkan SDK integration with After Effects plugins?

Eric CPFX shared a Skeleton project designed for using the Vulkan SDK with After Effects plugin development. This project serves as a reference implementation for developers looking to integrate Vulkan into their AE plugins.

*Tags: `vulkan`, `skeleton`, `open-source`, `reference`, `build`*

---

## How do you build an After Effects plugin for Apple Silicon M1 Macs?

To build for M1, you need to: (1) Set the appropriate build configuration for Mac ARM64 architecture, and (2) Add CodeMacARM64 {"EffectMain"} to your PiPL (Plugin Property List) file to ensure the plugin is properly registered for native ARM64 execution rather than Rosetta emulation.

```cpp
CodeMacARM64 {"EffectMain"}
```

*Tags: `apple-silicon`, `macos`, `build`, `pipl`*

---

## What are the advantages of using Lua over Python for developing After Effects plugins?

Lua is significantly better suited for plugin development compared to Python. The main advantage is that Lua is much easier to compile in and integrate as a library. Python integration was problematic because it required asking permission to install libraries all over the user's system, which is intrusive. Lua avoids this dependency management issue entirely. Both languages require similarly tedious C bindings/code generation, but Lua's compilation and integration model is cleaner for plugin distribution.

*Tags: `scripting`, `plugin`, `build`, `deployment`*

---

## How can you bind C code to scripting languages for After Effects plugins?

There are multiple approaches to create C bindings for scripting languages in AE plugins. For Python, you can use existing bindings (like cairo/pycairo) and pass parameters as text using sprintf formatting (e.g., sprintf("%s=%s\n", paramname, value)), then use the language's C API for bitmap and layer parameter assignment. For Lua, you can use XML-based C-binder code generation tools. Both approaches are similarly tedious without automation or code-generation tools to facilitate the binding process.

```cpp
sprintf("%s=%s\n", paramname, value)
```

*Tags: `scripting`, `plugin`, `build`, `aegp`*

---

## How do you debug a Premiere plugin from Xcode when symbols aren't loading and breakpoints don't work?

Ensure that symbol generation is enabled in your build settings. Set 'Generate Symbols' to 'Yes' in your Xcode scheme. If this setting is accidentally set to 'No', symbols won't be generated and debuggers won't be able to load them, preventing breakpoints from working properly.

*Tags: `debugging`, `premiere`, `xcode`, `build`*

---

## Why does Premiere pass NULL for input_world in Render when compiled with fast optimizations, but not in After Effects?

A developer reported that when compiling a plugin with fast optimizations enabled, Premiere passes input_world = NULL in the Render call, whereas without optimizations the data is present. The same code does not exhibit this problem in After Effects. This suggests the issue may be related to optimization levels affecting how Premiere handles or passes buffer data to plugins during rendering.

*Tags: `premiere`, `smartfx`, `render-loop`, `debugging`, `build`*

---

## How can you prevent the compiler from optimizing a render function that is causing issues in an After Effects plugin?

Add the `__attribute__ ((optnone))` compiler attribute before the render function declaration. This tells the compiler to skip optimization for that specific function, even if the rest of the plugin is optimized. This was identified as a workaround by the community, though it may indicate an underlying issue worth investigating further.

```cpp
__attribute__ ((optnone))
void render() {
  // render function implementation
}
```

*Tags: `build`, `debugging`, `render-loop`, `optimization`*

---

## How can I make a plugin support 8-bit, 16-bit, and 32-bit projects while sharing the same internal codebase without duplicating logic?

Use C++ templates to create multiple instances of your algorithm with different data types and range constants. This allows you to maintain a single shared codebase while handling different bit depths. Instead of letting After Effects automatically convert bit depths (which introduces color noise), you can manually handle the conversion within your plugin by instantiating template versions for 8-bit, 16-bit, and 32-bit processing.

*Tags: `mfr`, `params`, `build`, `reference`*

---

## How do you handle Unicode characters in After Effects plugin parameter names and categories on macOS?

The name field is defined as 'A_char[32]', so you need to convert your string to UTF-8 and feed it to the param macro. Make sure the length of the resulting UTF-8 string is no more than 31 characters long (including multibyte characters), as the last character must be reserved for null termination. One successful approach mentioned was using ConvertUTF8ToGBK() in a UTF-8 encoded .cpp file when working with XCode.

*Tags: `macos`, `params`, `unicode`, `build`, `xcode`*

---

## How should I handle string conversions between std::string and After Effects A_char and A_UTF16Char types?

Create a utility class with static methods for bidirectional string conversion. Use strcpy_s for A_char conversion and std::wstring_convert with std::codecvt_utf8_utf16 for UTF-16 conversions. Always ensure null-termination and check buffer bounds before copying. Use try-catch blocks for UTF-16 conversion to handle range_error exceptions.

```cpp
#include <locale>
#include <codecvt>
#include <string>
class AEStringConverter {
public:
  static PF_Err StringToAChar(const std::string& inString, A_char* outAChar, size_t bufferSize) {
    PF_Err err = PF_Err_NONE;
    errno_t copyResult = strcpy_s(outAChar, bufferSize, inString.c_str());
    if (copyResult != 0) err = PF_Err_OUT_OF_MEMORY;
    return err;
  }
  static PF_Err StringToAUTF16Char(const std::string& inString, A_UTF16Char* outAChar, size_t maxOutChars) {
    PF_Err err = PF_Err_NONE;
    std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
    std::wstring utf16String = converter.from_bytes(inString);
    if (utf16String.size() + 1 > maxOutChars) return PF_Err_OUT_OF_MEMORY;
    std::copy(utf16String.begin(), utf16String.end(), outAChar);
    outAChar[utf16String.size()] = L'\0';
    return err;
  }
  static PF_Err AUTF16CharToString(const A_UTF16Char* inAUTF16Char, std::string* outString) {
    PF_Err err = PF_Err_NONE;
    try {
      std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
      *outString = converter.to_bytes((wchar_t*)inAUTF16Char);
    } catch (const std::range_error&) {
      err = PF_Err_OUT_OF_MEMORY;
    } catch (...) {
      err = PF_Err_OUT_OF_MEMORY;
    }
    return err;
  }
};
```

*Tags: `sdk`, `string-handling`, `character-encoding`, `error-handling`, `cross-platform`, `build`*

---

## Is it possible to hot reload After Effects plugins without restarting the application?

No, hot reloading is not practical for native C++ After Effects plugins. Each time you build C++ code, you must restart After Effects because the application calls parameter setup only once per session when the plugin is first loaded, so hot reloading cannot trigger re-initialization of parameters. However, ExtendScript and CEP plugins can be reloaded without restarting the entire AE application. Debugging through an IDE like Microsoft Visual Studio or Apple Xcode can make the restart cycle faster than manually launching AE.

*Tags: `params`, `build`, `debugging`, `sdk`, `scripting`*

---

## What resources are available for implementing hot reloading in Xcode development?

For Xcode development, refer to the Stack Overflow discussion on instant run or hot reloading: https://stackoverflow.com/questions/42529081/instant-run-or-hot-reloading-for-xcode. Note that hot reloading was more common in 32-bit builds but became less practical with the transition to 64-bit architecture. Limitations exist when hot reloading is used with After Effects plugins, particularly around parameter setup.

*Tags: `debugging`, `build`, `macos`, `reference`, `tool`*

---

## Why does an After Effects effect plugin crash when saving a project after upgrading to the latest SDK with MFR support?

The crash on save is likely related to sequence data flattening. When upgrading to newer SDKs, if you set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag during global setup but don't implement the PF_Cmd_SEQUENCE_FLATTEN selector, After Effects will crash when trying to save the project. The SEQUENCE_FLATTEN command is sent when saving or duplicating sequences and requires you to flatten sequence data containing pointers or handles so it can be written to disk. Additionally, verify you're allocating new handles when modifying instance sequence data rather than changing data in the same handle. Basic troubleshooting includes cleaning build intermediates, rebuilding from scratch, testing sample SDK projects, and progressively commenting out plugin code to isolate the problem.

*Tags: `mfr`, `sequence-data`, `sdk`, `debugging`, `build`*

---

## How do I resolve 'Cannot open source file' errors when moving After Effects SDK example projects to a new location?

When copying AE SDK example projects like the supervisor folder to a new location, include directories only apply to .h and .hpp header files, not to .c and .cpp source files. Source files use relative paths from the Visual Studio project file location. To fix the error, either re-import the missing files or maintain the original SDK folder structure relative to where your .vcxproj file is located. Simply adjusting the include directories in project properties will not resolve paths for source files.

*Tags: `build`, `windows`, `sdk`, `reference`*

---

## What language and tools are needed to create After Effects plugins?

After Effects plugins are written in C/C++. While you can work with other languages, all calls to the AE API must be in C/C++. The SDK is shipped with Visual Studio project files for Windows and Xcode project files for Mac. Effect controls windows and composition window overlay graphics are created using the DrawBot suite supplied by the AE SDK.

*Tags: `sdk`, `c/c++`, `windows`, `macos`, `ui`, `build`*

---

## How do I make a 3D plugin display the 3D icon and trigger rendering automatically when the camera moves in After Effects?

To enable 3D camera support in your After Effects plugin, you need to set the flags PF_OutFlag2_I_USE_3D_CAMERA and PF_OutFlag2_I_USE_3D_LIGHTS, and use AEGP_GetEffectCameraMatrix to access the camera matrix. However, you must also change the corresponding flags in the PiPL file. Additionally, on Windows, you need to clean and rebuild your project, as the PiPL only regenerates after a clean build.

*Tags: `3d`, `aegp`, `pipl`, `build`, `camera`*

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

## How do I embed an image resource in a Windows .aex plugin file?

Add your image file as an image resource or generic binary resource from the Visual Studio File menu. You will get a custom identifier for this resource. Then use standard Windows resource functions to load it. Example code: HRSRC myResource = ::FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA); HGLOBAL myResourceData = ::LoadResource(NULL, myResource); void* pMyBinaryData = ::LockResource(myResourceData);

```cpp
HRSRC myResource = ::FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA);
HGLOBAL myResourceData = ::LoadResource(NULL, myResource);
void* pMyBinaryData = ::LockResource(myResourceData);
```

*Tags: `windows`, `build`, `params`, `ui`, `deployment`*

---

## Can After Effects plugins be compiled in Release mode, and what are the runtime linking requirements?

Yes, After Effects plugins should be compiled in Release mode for both performance and deployment ease. There are four C++ runtime linking options: MT (multithreaded static), MD (multithreaded DLL), MTd (multithreaded debug static), and MDd (multithreaded debug DLL). Adobe recommends using MD or MDd because there is a limit to how many MT-compiled plugins After Effects can load simultaneously. In Release mode, use either MD or MT; in Debug mode, use either MDd or MTd. There is no noticeable performance difference between MD and MT variants.

*Tags: `build`, `windows`, `debugging`, `deployment`*

---

## Why won't my After Effects plugin compile in Release mode when using a third-party library compiled in Release?

If a plugin won't compile in Release mode, check the project settings for rogue '_DEBUG' macro definitions in the Preprocessor Definitions. Also inspect individual file preferences, as some files may have their own compile settings and additional macros that force Debug mode compilation. Remove any '_DEBUG' definitions that shouldn't be present in Release builds.

*Tags: `build`, `debugging`, `windows`*

---

## What is the function of the PiPL.r file in After Effects plugin development?

The PiPL.r (Plug In Property List) file was originally created to allow After Effects to get information about plug-ins without loading them, which was useful when computers had limited RAM (128MB). Nowadays it's mostly a legacy requirement. The values in the PiPL must correlate to the values set in the plugin's global setup call, or an error message will be sent during the plugin's launch indicating a mismatch. To set the correct values: put a breakpoint in the global setup function to see the numerical values applied to outflags and outflags2 variables, then copy these values to the PiPL file. Note that a clean rebuild is required for changes to take effect, as the pipl_tool only generates a new .rc file when one doesn't already exist.

*Tags: `pipl`, `plugin-properties`, `build`, `deployment`*

---

## How do you set AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 values in the PiPL.r file?

To set these flags correctly, place a breakpoint in the plugin's global setup function and observe the numerical values being applied to the outflags and outflags2 variables. Copy these exact numerical values to the corresponding fields in your PiPL.r file. The values must match what your plugin sets during initialization, or a mismatch error will occur at plugin launch. After making changes, perform a clean rebuild to ensure the pipl_tool regenerates the .rc file from your updated .r file.

*Tags: `pipl`, `params`, `build`, `debugging`*

---

## What is the difference between .plugin and .aex file formats for After Effects plugins?

On Windows, After Effects plugins are built as .aex files. On macOS, plugins are built as .plugin files, which are actually bundles (not single files). Both formats are correct for their respective platforms and will work when built correctly according to the SDK instructions.

*Tags: `macos`, `windows`, `build`, `deployment`*

---

## What steps are necessary to convert an After Effects plugin from Windows to Mac?

If your C++ plugin code doesn't use Windows-specific native APIs or special library calls, porting to Mac should be straightforward—essentially just recompiling after adding all relevant project files. You'll need to use Xcode as your development environment on Mac instead of Visual Studio. Start from the Mac sample project (just as you did with the Windows sample project), but the plugin C/C++ code itself should require minimal to no changes if you've adhered to cross-platform APIs and the After Effects SDK.

*Tags: `cross-platform`, `windows`, `macos`, `build`, `deployment`*

---

## How do you change the plugin name and menu in an After Effects SDK sample plugin?

The plug-in name and menu are set in the resource file (namePiPL.r). After you change it, you'll need to clean the build and re-build the project, otherwise the new settings will not apply in the compiled plug-in.

*Tags: `pipl`, `build`, `sdk`, `resource`*

---

## What does the PF_OutFlag2_SUPPORTS_QUERY_DYNAMIC_FLAGS error mean and how do you fix it?

The error 'effect cannot change non-dynamic flag bits during PF_Cmd_QUERY_DYNAMIC_FLAGS' occurs when there's a mismatch between the PiPL (Plug-In Property List) resource and the PF_Cmd_GLOBAL_SETUP implementation. The PF_OutFlag2_SUPPORTS_QUERY_DYNAMIC_FLAGS flag must be set consistently in both the PiPL and during PF_Cmd_GLOBAL_SETUP. On macOS, you can edit PiPL resources using Resorcerer with the PiPL template from the SDK, or use DeRez to create a .r file. The behaviors indicated during global setup must match those in the PiPL exactly. On Windows, ensure the .rc file is regenerated by cleaning and rebuilding the project.

*Tags: `pipl`, `macos`, `windows`, `build`, `debugging`*

---

## How do you set up After Effects plugin development with the SDK and compile it to .aex format?

To develop an After Effects plugin: 1) Download the SDK (e.g., CC 2015 or later). 2) Ensure you have the required Visual Studio C++ compiler installed (the SDK specifies which version is needed for your AE version). 3) Open one of the example projects from the SDK (such as Convolutrix, which is a good beginner example). 4) In Visual Studio project settings, configure the output directory to point to After Effects' plugin directory or a folder with a shortcut to it. This allows you to build directly into a folder AE reads and debug your plugin. 5) Build the project, which will generate the .aex file. 6) The compiled .aex plugin will then be readable by After Effects. Note that plugin development does not use the Object Model like ExtendScript does—it is C/C++ based SDK development.

*Tags: `aex`, `sdk`, `build`, `visual-studio`, `plugin-development`*

---

## How do you debug an After Effects plugin in Xcode?

Set the Xcode executable path to the After Effects application, then hit Cmd+R to build and run a debug session. Make sure the build location is the same place from which AE loads the plugin—you can create an alias from AE's plugins directory to the build location, and AE will scan the alias. Note that param and global setup are only done when you first apply an effect to a layer, so changes to those require restarting AE. Quick fix/continue functionality was removed in the transition to x64 and did not return.

*Tags: `debugging`, `macos`, `build`*

---

## Can you keep After Effects open between debug builds when developing a plugin?

Parameter and global setup are only performed when you first apply an effect to a layer and are shared for all instances throughout the session, so if you change code in those areas you must restart AE. The quick fix/continue feature that once allowed code changes while running was removed during the transition to x64 architecture.

*Tags: `debugging`, `build`*

---

## How do you set up debugging for an After Effects SDK example project in Visual Studio?

In the project properties window, set the command for debugging by providing the path to the afterfx.exe file. You can ignore any symbols-related warnings that appear.

*Tags: `debugging`, `build`, `windows`, `sdk`*

---

## What is the difference between scripting and plug-ins in After Effects?

Scripting and plug-ins are two different approaches to extending After Effects. Scripting is done with JavaScript and can perform operations in the project and create palettes, but cannot process layer pixels. It is simpler, does not require compiling, and is cross-platform. Plug-ins are created in C++, can do everything scripting does plus process layer pixels to create effect plug-ins, but require Visual Studio on Windows and Xcode on Mac, require C/C++ knowledge, and must be compiled separately for each platform.

*Tags: `scripting`, `cross-platform`, `build`*

---

## What file format does a compiled After Effects plug-in produce?

When you compile an After Effects plug-in using an SDK sample project as your base in Visual Studio or Xcode, it produces a .aex file. This file can be placed in After Effects' plug-ins folder and will appear in the effects menu when After Effects is launched.

*Tags: `build`, `deployment`*

---

## What causes an 'invalid filter' error when loading a compiled After Effects plugin?

The invalid filter error is usually a dependency issue. Check your plug-in using Dependency Walker to verify all required components are present on your machine. This error also commonly occurs when you compile your plug-in in Debug mode and try to run it on a non-development machine where the debug DLLs don't exist. Compiling in Release mode resolves this issue.

*Tags: `debugging`, `build`, `windows`, `deployment`*

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

## Why does the After Effects SDK build succeed but produce no .dll or .aex file?

By default in the CS4 SDK, plugins build to c:\program files\adobe\after effect 9.0\plug-ins\sdk (note "after effect 9.0" instead of "CS4"). If you don't see output in your expected directory, check the Visual Studio Linker options to verify the output directory is set correctly, or search your computer for the .aex file as it may be building to an unexpected location.

*Tags: `build`, `sdk`, `after_effects_sdk`, `visual_studio`, `deployment`*

---

## How do you fix a Skeleton project that won't build with a web deployment error?

If the Skeleton project shows a conversion warning about "Web deployment to the local IIS server is no longer supported" and won't build, this error disables your build settings. The simplest solution is to duplicate a working project from the SDK and replace only the .cpp and .h files with the Skeleton files, rather than trying to manually fix the corrupted build settings.

*Tags: `build`, `sdk`, `skeleton`, `visual_studio`, `after_effects_sdk`*

---

## How do I resolve error 126 when users try to load my After Effects plugin?

Error 126 typically indicates a dependency issue. Use Dependency Walker to analyze your plugin and identify missing or problematic DLL dependencies. If your plugin requires MSVCR100D.DLL (debug runtime), change your C/C++ code generation runtime library setting: use /MT to statically link the runtime library into your plugin (no external DLLs needed, larger file size), or /MD to use the release version of MSVCR100 (requires distributing the DLL). Test your plugin on a non-developer machine using Dependency Walker to ensure no problematic dependencies remain. Avoid requiring debug runtime libraries in production builds.

*Tags: `windows`, `deployment`, `build`, `debugging`*

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

## How do you create a modal dialog window on macOS using Interface Builder for an After Effects plugin options button?

Create a .nib file in Interface Builder and set the window class type to 'movable modal' to keep it in front of After Effects. Add the .nib file to your Xcode project and add it to the 'Copy Bundle Resources' build phase. Link CoreFoundation.framework and IOKit.framework. Update your plugin's .plugin-info.plist with a unique CFBundleIdentifier. Then use the Carbon API to load the .nib file: create a CFBundleRef, use CreateNibReferenceWithCFBundle, CreateWindowFromNib, and install event handlers. Call RunApplicationEventLoop to display the dialog and handle user interactions before closing.

```cpp
CFBundleRef bundleRef = CFBundleGetBundleWithIdentifier(CFSTR("com.adobe.AfterEffects.YourPlugName"));
IBNibRef theNibRef = NULL;
OSStatus theErr = CreateNibReferenceWithCFBundle(bundleRef, CFSTR("YourNibFileName"), &theNibRef);
WindowRef ActivationWindow = NULL;
theErr = CreateWindowFromNib(theNibRef, CFSTR("Window"), &ActivationWindow);
DisposeNibReference(theNibRef);
RepositionWindow(ActivationWindow, NULL, kWindowCascadeOnMainScreen);
ShowWindow(ActivationWindow);
theErr = InstallWindowEventHandler(ActivationWindow, EventHandlerFunction, GetEventTypeCount(kEvents), kEvents, (void*)ActivationWindow, NULL);
RunApplicationEventLoop();
```

*Tags: `ui`, `macos`, `build`*

---

## How do you configure Debug and Release build configurations in Xcode for After Effects plugins?

Duplicate the Debug configuration in Xcode and rename it to Release. Remove the '_DEBUG' preprocessor macro from the Release version. Enable 'Strip debug symbols during copy' in the Release configuration to reduce binary size. Additional optimizations may be applied depending on your specific needs.

*Tags: `build`, `macos`, `debugging`*

---

## Why can't VC++ 2008/2010 Express open After Effects SDK example projects?

The After Effects CS5 SDK example projects require 64-bit compiler configurations, which are not checked by default in the Visual Studio installer. You need to install the 64-bit compiling options during Visual Studio setup. Additionally, ensure the Windows SDK is installed and that Visual Studio is configured to look in the correct SDK paths. The Premiere Pro CS5 SDK works because it includes both 32-bit and 64-bit configurations to support 32-bit applications like Encore CS5.

*Tags: `build`, `windows`, `sdk`, `debugging`*

---

## What is the correct Mac OS X SDK version to use when porting an After Effects plugin from Windows to Mac?

When porting a plugin to Mac using Xcode, if you encounter an error about a missing SDK (such as 'There is no SDK with the name or path /Developer/SDKs/MacOSX10.4u.sdk'), you should update the project's Base SDK setting to a newer version like Mac OS X 10.6. The available choices typically include versions like 10.5, 10.6, and "Current Mac OS", and 10.6 was confirmed as a working choice for successful compilation.

*Tags: `macos`, `build`, `xcode`, `cross-platform`, `skeleton`*

---

## How do I fix the "project.pbxproj is not writeable" error when working on a Skeleton plugin in Xcode on Mac?

If Xcode shows an error that the project.pbxproj file is not writable and cannot be saved, the issue may be related to SCM (source control management) settings in the project file. To fix this: (1) Back up the Skeleton project first, (2) Quit Xcode, (3) Navigate to the Skeleton.xcodeproj file in Finder, (4) Option-click it and select "Open Package Content" (since .xcodeproj is actually a package/folder), (5) Find and edit the project.pbxproj file in a text editor, (6) Search for and remove any paragraphs that reference SCM settings. Alternatively, check if the project files are located on a backup drive like Time Machine, as this can also cause write permission issues.

*Tags: `macos`, `build`, `xcode`, `debugging`*

---

## How do I fix the 'project file is not writeable' error when modifying After Effects SDK skeleton projects in Xcode?

The issue is typically that the project files have write-protection enabled. Check if files are locked by selecting them in Finder and opening Get Info to see if the Locked checkbox is checked. If so, uncheck it for each file. This write-locking is common when files are transferred from Windows due to read-only media assumptions. If files aren't locked, verify project settings don't have SCM (Source Control Management) enabled, as this can also cause writeability issues with .pbxproj files.

*Tags: `skeleton`, `build`, `macos`, `xcode`, `debugging`*

---

## Why won't my After Effects plugin built with SDK 7.0 load on Mac CS4 when it works fine on Windows?

The issue is related to architecture support differences between SDK versions. AE7 was the last version to run on PPC processors; on Intel chips it ran emulated. CS3 was the first to run native on Intel chips and SDK 8.0 (CS3) was the first SDK to support plugins running native on Intel chipsets (and PPC). The CS3 SDK offers backwards compatibility to AE7, allowing plugins to work across AE7, CS3, and CS4 on both PC and Mac. In contrast, the CS4 SDK is not backwards compatible. To achieve cross-version compatibility on Mac, you should check the resource file declarations in the CS3 SDK samples, which contain separate entry point declarations for Intel and PPC architectures.

*Tags: `macos`, `windows`, `sdk`, `build`, `cross-platform`, `pipl`*

---

## How do you compile the Panelator sample to a release version that runs on non-developer machines?

When compiling Panelator for release, change the runtime library from /MD to /MT to make the plugin independent of external MSVC runtime DLLs. Additionally, ensure all supporting code files in the 'supporting code' folder have consistent compiler settings (not mixed MD/MDd). If using Visual Studio 2008 instead of the SDK-intended VS2005, be aware that 2008 uses MSVCR90.DLL instead of MSVCR80.DLL, which can cause compatibility issues. Using /MT will statically link all libraries into the plugin, making it dependent only on KERNEL32.DLL. Use Dependency Walker to verify the plugin's dependencies before deployment.

*Tags: `build`, `windows`, `deployment`, `debugging`*

---
