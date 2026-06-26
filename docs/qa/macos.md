# Q&A: macos

**162 entries** tagged with `macos`.

---

## What causes slow AE loading when debugging from Xcode 15?

After upgrading to macOS 14/Xcode 15, AE loading becomes ~3x slower due to 'dlsym cannot find symbol xSDKExport' messages for MediaCore bundles. This is Xcode trying to load symbols for all loaded bundles. Workarounds: (1) Run Xcode 14 binary directly from Terminal. (2) Open AE standalone first, then attach the debugger after AE is fully loaded. (3) Check if 'load symbols' option is accidentally selected in your IDE settings.

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**tlafo**](../contributors/tlafo/), [**gabgren**](../contributors/gabgren/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2023-11-29 · Tags: `xcode`, `debugging`, `performance`, `macos`, `symbol-loading`*

---

## What caused plugins to fail to load in AE 25.2 on macOS?

The issue was related to code signature validation. Adobe built AE 25.2 with a newer Xcode that enforces stricter code signature checks. Multiple root causes were identified: (1) Unsigned plugins now fail to load on Apple Silicon. (2) Installers that don't properly clean up old files can break code signatures - if v1 had file X and v2 removes it, but the installer leaves it, the signature breaks. (3) Some plugins had 0-byte .plugin files after the AE upgrade. Solution: Ensure proper code signing, notarization, and clean installation (remove all old files before installing new version).

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**gabgren**](../contributors/gabgren/), [**wunk**](../contributors/wunk/) · Source: adobe-plugin-devs · 2025-04-08 · Tags: `code-signing`, `macos`, `ae-25-2`, `notarization`, `installer`, `apple-silicon`*

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

*Contributors: [**fad**](../contributors/fad/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2025-04-18 · Tags: `rust`, `debugging`, `lldb`, `vscode`, `macos`*

---

## Should .aex plugins be code signed on Windows?

.aex plugins are regular DLLs (just renamed), so they can be codesigned. Microsoft currently allows unsigned DLLs to be loaded from a signed process, but they have hinted this might change in the future (was supposed to happen with Win11). On macOS, a plugin is a bundle (renamed folder), and you don't need to sign the actual plugin binary inside it as long as the bundle itself is signed.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2023-12-13 · Tags: `code-signing`, `aex`, `dll`, `windows`, `macos`, `security`*

---

## Why might a plugin fail to load on some Macs with 'couldn't find main entry point' error, even without missing dependencies?

While 'couldn't find main entry point' usually means missing dependencies, it can also be caused by library collisions. For example, if your plugin links against curl, check whether you're providing your own dylib or linking against Apple's system version. On macOS, curl is normally relatively unproblematic -- you can usually safely use the lib provided by Apple. Also check whether the issue appears only on Silicon or Intel Macs.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2025-12-12 · Tags: `macos`, `loading-error`, `entry-point`, `library-collision`, `curl`, `dylib`*

---

## Do Mac plugins need to be notarized in addition to codesigning for distribution via the aescripts app?

Yes, codesigning alone is not enough. The plugins also need to be notarized by Apple for distribution.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2026-02-16 · Tags: `macos`, `notarization`, `code-signing`, `distribution`, `aescripts`*

---

## After signing, notarizing, and stapling a Mac plugin, why does it become untrusted after uploading and re-downloading?

This is a known issue where the notarization appears successful (notarytool says Accepted) but spctl reports 'rejected / source=Unnotarized Developer ID' after download. The downloaded file may get an Apple quarantine xattr. This can happen with AEGP plugins that are .plugin bundles. The issue may be related to how the bundle is zipped/unzipped, or how the stapling interacts with the bundle structure. Check the xattr flags after download with 'xattr -l <path>' to see if quarantine is applied.

```cpp
codesign --options runtime --force --deep --timestamp -strict --sign "Developer ID Application: Name (TEAMID)" <path/to/.plugin/file>
xcrun notarytool submit <path/to/zipped/.plugin/file> --keychain-profile "Profile" --wait
xcrun stapler staple <path/to/.plugin/file>
```

*Source: aescripts discord · 2024-09-12 · Tags: `macos`, `notarization`, `code-signing`, `quarantine`, `spctl`, `aegp`*

---

## Is there a skeleton/starter project for building an AE plugin with Vulkan GPU rendering?

Yes — AE_Skeleton_Vulkan by Eric CPFX (emlcpfx) is a complete CMake-based skeleton that incorporates the Vulkan SDK into an AE plugin. It also includes build scripts and a sample script for signing, notarizing, and stapling on Mac. Repository: https://github.com/emlcpfx/AE_Skeleton_Vulkan

*Source: curated · Tags: `vulkan`, `gpu`, `cmake`, `build`, `macos`, `code-signing`, `skeleton`*

---

## Is it necessary to build on Apple Silicon hardware to support it?

It is not strictly required to build on Apple Silicon hardware, but it is best practice to test on the actual platform. Purchasing an M1 Mac mini is a practical and cost-effective way to validate Apple Silicon support.

*Tags: `apple-silicon`, `macos`, `build`*

---

## What cross-platform GPU technology issues exist between Windows and macOS?

OpenCL-based plugins perform well on Windows but are significantly slower on macOS. This suggests that different GPU technologies have vastly different performance characteristics across platforms, requiring careful consideration when choosing GPU frameworks.

*Tags: `gpu`, `cross-platform`, `macos`, `windows`*

---

## What is the recommended approach for supporting GPU rendering across Windows and macOS?

In the future, maintaining two completely separate code bases may be necessary: one using Metal for macOS and one using OpenGL or another framework for Windows, due to significant platform-specific GPU compatibility issues.

*Tags: `gpu`, `metal`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## What is required to support M1 Macs when including external libraries like OpenCV in an After Effects plugin?

When building for M1 with external libraries, ensure the library is compiled with appropriate Mac support options. If using dynamic libraries (dylib), you need to embed them in the plugin file and specify rpath in Xcode. Static libraries (.a files) should be linked directly. The Missing Entry Point error typically occurs when the plugin is applied, not during loading, and may indicate a PiPL configuration issue rather than a linking problem. It's recommended to test on both M1 and x86 Mac architectures to isolate platform-specific issues, as the problem may be related to project configuration rather than the external library itself.

*Tags: `apple-silicon`, `macos`, `build`, `pipl`, `deployment`*

---

## Has anyone used metal cpp bindings instead of Objective-C for After Effects plugin development?

No direct answer was provided in the conversation. One participant acknowledged the question but did not provide experience or recommendations.

*Tags: `metal`, `macos`, `build`*

---

## Did you try Metal first before using MoltenVK for cross-platform GPU development?

Yes, Metal was tried first, but MoltenVK with Vulkan was chosen because it reduces code duplication between Windows and Mac platforms. However, there are some Mac-specific limitations to handle. Using Metal directly alongside DirectX/Vulkan would be like writing two separate plugins, and the developer encountered inconsistent results with SPIR-V to MSL versus SPIR-V to Vulkan without MoltenVK.

*Tags: `gpu`, `metal`, `vulkan`, `cross-platform`, `macos`, `windows`*

---

## What GPU APIs are being used for Windows and Mac support?

For cross-platform development, MoltenVK (a Vulkan layer over Metal) is used, which allows shared code between Windows and Mac. Alternatively, some developers keep OpenGL on Windows and Metal on Mac. OpenGL 4.x and 3.3 are still in use by major plugins like Element 3D and Helium, though there are concerns about Apple potentially removing OpenGL support entirely.

*Tags: `gpu`, `opengl`, `metal`, `vulkan`, `cross-platform`, `macos`, `windows`*

---

## What are the limitations of MoltenVK before version 1.3?

MoltenVK before VK 1.3 had some limitations including unsupported texture swizzle and uniforms limitations, but these issues are now resolved in version 1.3 and later.

*Tags: `vulkan`, `metal`, `moltenvk`, `macos`*

---

## How can I embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications using MoltenVK?

You can embed MoltenVK in the plugin's framework folder as a dynamic library with the JSON configuration in resources/vulkan. However, this path approach doesn't work reliably for plugin bundles. An alternative is to use xcframework to embed it as a static library inside the plugin binary, similar to what the scaleUp plugin does. If Vulkan fails to create instances when using the static library approach, verify that the Vulkan layer configuration and initialization is correct for the statically-linked library within the plugin bundle context.

*Tags: `vulkan`, `macos`, `deployment`, `build`, `cross-platform`*

---

## How can MoltenVK be embedded inside an After Effects plugin bundle to avoid conflicts with other MoltenVK applications?

MoltenVK can be embedded as a static library using xcframework within the plugin binary, similar to how the scaleUp plugin does it. However, this approach requires careful configuration of the Vulkan loader and MoltenVK ICD to ensure proper function pointer resolution and layer/extension loading. The main challenge is ensuring the Vulkan loader correctly locates the MoltenVK ICD when statically linked, as opposed to using the standard /etc/vulkan or /usr/local installation paths. Testing should verify that the Vulkan loader is correctly configured for the static linking approach.

*Tags: `macos`, `vulkan`, `build`, `deployment`, `cross-platform`*

---

## What causes VK_ERROR_LAYER_NOT_PRESENT and VK_ERROR_EXTENSION_NOT_PRESENT errors when creating Vulkan instances with embedded MoltenVK in a plugin?

These errors typically indicate that the MoltenVK ICD is not being located correctly by the Vulkan loader, or there are environment configuration issues. When MoltenVK is statically embedded rather than installed system-wide, the Vulkan loader may not find the necessary validation layers and extensions. This is particularly challenging in sandboxed or non-standard installations where the LunarG Vulkan SDK is not in the default /usr/local path. The solution involves ensuring the Vulkan loader can properly locate the MoltenVK ICD configuration.

*Tags: `macos`, `vulkan`, `debugging`, `deployment`*

---

## How do you set up MoltenVK compatibility with Vulkan 1.2 on macOS for After Effects plugins?

Get at least the October update to get a MoltenVK compatible with Vulkan 1.2. For debug builds, create a target using ICD in environment with dylib linking, add "VK_KHR_portability_subset" to instance extensions (debug only for now until MoltenVK uses it), and include debugging layers. For release builds, link to frameworks only as static libraries to work within Vulkan 1.2 limitations.

*Tags: `vulkan`, `macos`, `apple-silicon`, `build`, `debugging`*

---

## How to fix OpenGL loader initialization failure on M1 Mac with ImGui?

The issue is likely related to M1 chip compatibility. ImGui can run over Metal instead of OpenGL, which would require conversion from OpenGL matrices but would be the proper approach for Apple Silicon. The error may be similar to previous issues with GLSL version differences between Mac and Windows, but specific to ARM64 architecture on M1.

*Tags: `opengl`, `metal`, `macos`, `apple-silicon`, `debugging`*

---

## How do you print to the console when debugging an After Effects plugin in Visual Studio?

On Mac, use printf. On Windows, use the OutputDebugStringA macro. You can also use AEGP_WriteToOSConsole from the utility suite, though this may not work in Premiere. OutputDebugStringA is the recommended approach for Windows console output during debugging.

```cpp
OutputDebugStringA("debug message");
```

*Tags: `debugging`, `windows`, `macos`*

---

## Can a deployment target mismatch in Xcode cause an 'Invalid Filter' error on macOS?

Yes, setting a deployment target higher than the user's OS version will cause that error. The user had deployment target 12.3 in Xcode but the user was on Big Sur 11.17.9, which is lower than the deployment target.

*Tags: `macos`, `deployment`, `debugging`*

---

## How can you debug plugin crashes when changing the macOS deployment target doesn't resolve the issue?

Add logging at function entry and exit points to determine where the crash occurs (global setup, param setup, etc). You can use conditional logging by checking for a log file's presence in global setup, so users only enable logging when needed. For production plugins, consider profiling performance impact of logging and use a library like spdlog with file rotation to manage log file sizes.

*Tags: `macos`, `debugging`, `pipl`*

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

## Why is there a memory leak when using AEGP_memorysuite on Mac but not Windows during export?

The memory leak appears to be platform-specific (Mac only) and related to how AEGP_memorysuite handles memory operations. The issue is less severe with the mem_QUIET flag. Using standard C++ memory allocation (new/delete) instead of AEGP_memorysuite resolves the leak, and replacing memcpy with strncpy partially solves the problem (from mem pointer to buffer works, but not in the reverse direction). This suggests the issue may be related to how memcpy interacts with AEGP's memory management on macOS, particularly when memory is freed from a different thread than where it was allocated.

*Tags: `memory`, `macos`, `aegp`, `threading`, `debugging`*

---

## How should memory be freed differently on Mac versus Windows?

Memory should be freed later in another thread on Mac but not on Windows, due to platform-specific threading and memory management differences.

*Tags: `memory`, `macos`, `windows`, `threading`*

---

## How can I debug a crash in After Effects on Windows that occurs during project loading when my plugin is installed, but not on Mac?

Use a debugger with breakpoints on EffectMain to trace when the crash occurs relative to plugin initialization. The crash appears to happen after GlobalSetup and ParamsSetup complete successfully (around 85% project load) with no plugin code executing at that point, suggesting a symbol clash or state corruption rather than direct plugin code failure. Try importing the project into a new empty project as a workaround, or preload the conflicting library (Cineware in this case) before opening the project.

*Tags: `debugging`, `windows`, `macos`, `crash`*

---

## Why doesn't macOS Instruments detect memory leaks from PF_EffectWorlds that are allocated but never disposed?

PF_EffectWorlds are likely allocated deep within the AE engine rather than directly within the plugin code, so memory leak detection tools like Instruments don't recognize them as true leaks since they're allocated at a different level in the AE memory management hierarchy.

*Tags: `memory`, `macos`, `debugging`, `aegp`*

---

## Why doesn't macOS Instruments detect memory leaks from undisposed PF_EffectWorlds even though After Effects runs out of memory?

PF_EffectWorlds allocated in After Effects plugins may not be detected by Instruments leak detection because After Effects itself may be managing the memory lifecycle through its own memory management systems, or the allocations may be attributed to After Effects' internal memory pools rather than the plugin's direct allocations. Instruments can detect direct memory leaks (e.g., data allocated in pre-render that isn't disposed), but large framework-managed allocations like PF_EffectWorlds may be tracked differently and not flagged as leaks even when they cause memory exhaustion.

*Tags: `memory`, `debugging`, `macos`, `plugin-leak-detection`*

---

## How do you disable the annoying popup that appeared in After Effects 2024?

Use CMD+F12 to open the console. Click the hamburger menu next to the console and change to Debug Database. Search for 'ShowPreviousCrashWarning' and turn it off. Alternatively, the setting can be modified in the prefs.txt file.

*Tags: `debugging`, `macos`, `ui`*

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

## Why might an AEGP not appear in AE and not hit any breakpoints in the entire project?

The questioner did not receive a definitive answer. The conversation moved to other topics without resolution.

*Tags: `aegp`, `debugging`, `macos`*

---

## What causes memory issues when using the Compute Cache API on macOS during export?

On macOS, when caching large amounts of data (e.g., 100MB per frame) during export, the memory handles are not destroyed quickly enough, causing RAM to become overloaded and eventually crash on long exports. The issue is that the delete function may not be called frequently enough to free the cached memory handles during the export process.

*Tags: `compute-cache`, `memory`, `macos`, `deployment`*

---

## Have you used nuitka for Python compilation on macOS with numpy dependencies?

One developer tried nuitka once but had limited experience with it. They preferred using Cython for compilation instead to handle their specific needs.

*Tags: `scripting`, `macos`, `deployment`*

---

## How do you compile .rc files on macOS for After Effects plugins?

On macOS, you can use the native macOS tool called Rez to compile .rc files. The virtualritz/after-effects project demonstrates this approach as part of its build system. The Rust PiPL compiler from that same project can also handle resource compilation as part of a cross-platform build setup.

*Tags: `build`, `macos`, `pipl`, `deployment`*

---

## How can you achieve incremental compilation for After Effects plugins to improve build times?

Using the Ninja generator with cmake can provide faster incremental compilation. The xcodebuild CLI tool does not appear to support incremental compilation effectively, resulting in full rebuilds taking around 50 seconds. Switching from Xcode project generation to Ninja as the cmake generator should enable proper incremental compilation.

*Tags: `build`, `cmake`, `macos`, `performance`*

---

## What is the correct way to call AEGP_StartUndoGroup in After Effects 2025?

In After Effects 2025 beta and later, AEGP_StartUndoGroup must be called with an empty string "" rather than null. Passing null will cause a crash, while passing an empty string works as expected and does not add an entry to the Undo stack.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `undo`, `macos`, `windows`, `debugging`*

---

## How do you debug a plugin that fails to load in After Effects and appears offline in Premiere?

Check if the error occurs before global_setup is called. If it happens before that point, it may be due to a library dependency that AE loaded in a new version that your plugin uses. Verify whether the issue occurs on both Windows and macOS or only on specific platforms.

*Tags: `debugging`, `deployment`, `macos`, `windows`, `pipl`*

---

## What could cause a simple C++ plugin with no external dependencies to fail loading?

Even simple plugins without explicit dependencies can fail due to library dependencies loaded by After Effects itself in newer versions. The issue may be platform-specific, as it can work on one platform (like Windows) while failing on another (like macOS).

*Tags: `debugging`, `deployment`, `cross-platform`, `macos`, `windows`*

---

## What is the minimal macOS SDK version required for After Effects plugins?

The minimal macOS SDK requirement was not specified in the conversation. The question was asked but not answered.

*Tags: `macos`, `deployment`, `build`*

---

## What macOS SDK was used to compile the plugins?

The conversation discusses whether plugins were compiled with a newer macOS SDK with an updated version, noting differences between Adobe's ExporterAIFF.plugin between versions 25.1 and 25.2, but no definitive answer is provided about which specific SDK version was used.

*Tags: `macos`, `build`, `deployment`*

---

## Why are some plugin files becoming 0 bytes after upgrading to After Effects 2025.2?

Multiple plugins were found to be 0 bytes (emptied) after upgrading to AE 2025.2, while released versions work properly. This appears to be a bug in the 2025.2 release that corrupts plugin files in the MediaCore folder during upgrade, though the root cause was not determined in the conversation.

*Tags: `deployment`, `macos`, `windows`, `build`*

---

## Why are After Effects plugins failing to load on Apple Silicon Macs in versions 25.2 and 25.3?

The investigation suggests code signing requirements have become stricter on Apple Silicon Macs. While entitlements haven't changed, Adobe may be using a newer Xcode version that enforces stricter code signing validation. Additionally, there may be a new hardening behavior in Apple's code-signing that creates a problematic interaction between After Effects' own signatures and plugin signatures.

*Tags: `macos`, `apple-silicon`, `code-signing`, `deployment`*

---

## How can you code sign and notarize After Effects plugins for macOS?

You need an Apple Developer account (costs $99/year). With this account, you can sign and notarize your plugins. The process involves using Xcode's code signing tools to sign the plugin executables and then submitting them to Apple's notarization service.

*Tags: `macos`, `deployment`, `code-signing`*

---

## How do you get a stacktrace with symbols when using the Rust bindings for After Effects on macOS?

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

*Tags: `rust`, `macos`, `debugging`, `lldb`*

---

## How can installer leftovers from previous versions break code signing on macOS?

When an installer overwrites files instead of properly uninstalling, leftover files from the previous version remain in the package. Any extra files in a signed package will break the code signature. For example, if version 1 has file X and version 2 removes it, but the installer leaves file X behind, the code signature breaks. The solution is to completely uninstall and remove all files before installing the new version, rather than just overwriting files.

*Tags: `macos`, `deployment`, `build`, `codesign`*

---

## Is the After Effects code signing issue affecting binary plugins on Windows as well as macOS?

The binary code signing issue appears to be Mac-only. It's related to After Effects being built with the latest Xcode, which applies stricter code signature checks than before. There was a separate ZXP signing issue that affected both macOS and Windows platforms, but that's a different problem related to the ZXPSignCmd tool.

*Tags: `macos`, `windows`, `deployment`, `codesign`*

---

## Does notarization ensure that adding files to a macOS package won't break code signing?

No. Notarization does not prevent code signature breakage from modified package contents. Even after notarization, if you add or leave extra files in a signed package (like a dummy.txt file), the code signature will break. Proper uninstallation and clean installation are required to maintain valid code signatures.

*Tags: `macos`, `deployment`, `codesign`*

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

## How do you debug ExtendScript in After Effects?

On Windows, you can use the standard debugging tools. On macOS, the situation was previously difficult because you needed to use the Intel version which was very slow, but Adobe updated the debugger about a month ago to be Apple Silicon native, which is a significant quality of life improvement.

*Tags: `scripting`, `debugging`, `macos`, `apple-silicon`*

---

## Why does a plugin work in the Plugins folder on Mac but not in the Mediacore folder on Mac AE 26.0.0?

This is a known compatibility issue with AE 26.0.0 on Mac. Several developers have reported the same problem. Gaussian Splat and other plugin developers have issued updates to address AE 26 compatibility, suggesting a breaking change in how AE 26 handles plugins in the Mediacore folder on macOS.

*Tags: `macos`, `deployment`, `build`*

---

## Does the Mediacore plugin loading issue affect Premiere Pro 2026?

Yes, plugins in the Mediacore folder work in Premiere Pro 2023, 2024, and 2025, but do not show up or load in Premiere Pro 2026, indicating a similar compatibility issue across Adobe applications.

*Tags: `premiere`, `macos`, `deployment`*

---

## How do you disable the crash warning dialog in After Effects 2026 on macOS?

Use cmd + F12 to switch to database view, then search for "crash window" or similar to locate and disable the crash warning.

*Tags: `macos`, `debugging`, `ui`*

---

## What are the causes of plugins not loading in After Effects 2026?

There is a known issue on Mac where plugins from the Mediacore folder are not recognized and only load from AE's plugin folder. Additionally, AE2026 has GPU compatibility detection issues where it may only detect onboard GPU and refuse installation if it deems the GPU incompatible.

*Tags: `macos`, `deployment`, `gpu`*

---

## How do you build an After Effects plugin for Apple Silicon (M1) native support?

To build for M1 native (non-Rosetta), you need to: 1) Set the appropriate build configuration for Apple Silicon architecture, and 2) Add CodeMacARM64 {"EffectMain"} entry in the PiPL resource file. Without the PiPL entry, the plugin won't be recognized as native ARM64 compatible even if the binary is built correctly.

```cpp
CodeMacARM64 {"EffectMain"}
```

*Tags: `macos`, `apple-silicon`, `build`, `pipl`*

---

## How do you properly initialize ImGui with OpenGL on macOS?

When initializing ImGui with OpenGL on macOS, window flags must be set before window creation, not after. This differs from Windows behavior and is a common mistake. The OpenGL loader initialization fails if flags are set in the wrong order during the setup process.

*Tags: `macos`, `opengl`, `ui`, `debugging`*

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

## Can you build After Effects plugins against older macOS SDKs using the latest Xcode?

James Whiffin reported that with the latest Xcode, there are compatibility issues when trying to build against older macOS SDKs (e.g., 10.10). This is a known limitation where newer Xcode versions may not support targeting legacy macOS versions.

*Tags: `macos`, `build`, `cross-platform`, `deployment`*

---

## How do you properly include external libraries like OpenCV in an After Effects plugin targeting Apple Silicon (M1)?

When including external libraries in AE plugins for M1, ensure you're linking against static libraries (.a files) rather than dynamic libraries. If using dynamic libraries, you may need to specify rpath in Xcode and embed the dylib in the plugin file. Verify that your PiPL file contains the correct entry point definition. If experiencing 'Missing Entry Point' errors when applying the effect, test on x86 Mac to isolate whether the issue is architecture-specific or a general PiPL configuration problem. Common causes include outdated project settings when migrating older projects to M1 support.

*Tags: `macos`, `apple-silicon`, `build`, `pipl`, `cross-platform`*

---

## How is it possible for a customer to have a newer GPU driver version than me on an older macOS version?

On Apple systems, drivers are typically bundled with the OS, making newer drivers on older OS versions unusual. One possibility is manual driver installation similar to Hackintosh setups, though this is not straightforward. Additionally, Apple may have issued driver downgrades in response to GPU issues in specific macOS versions (e.g., macOS 12.3 had known issues with AMD 5xxx and 6xxx GPUs). See: https://www.reddit.com/r/hackintosh/comments/ter3g2/psa_macos_monterey_123_and_amd_5xxx_and_6xxx_gpu/

*Tags: `macos`, `gpu`, `debugging`*

---

## Can you use Metal C++ bindings instead of Objective-C for After Effects plugin development on macOS?

James Whiffin asked about using Metal C++ bindings as an alternative to Objective-C for AE plugin development, noting that C++ would be more familiar for developers without Objective-C experience. This suggests that Metal C++ bindings are a viable option for macOS plugin development, offering a more accessible path for C++-focused developers.

*Tags: `metal`, `macos`, `cross-platform`, `gpu`*

---

## Should you use Metal directly or MoltenVK for cross-platform GPU plugin development on macOS?

Using Vulkan with MoltenVK (a Vulkan layer over Metal) reduces code duplication between Windows and macOS compared to maintaining separate Metal and DirectX implementations. However, there are some macOS-specific limitations to account for. Alternatively, keeping OpenGL on Windows and Metal on macOS is viable, but carries the risk that Apple may eventually remove OpenGL support entirely, as they have deprecated it in favor of Metal.

*Tags: `gpu`, `metal`, `vulkan`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## What GPU APIs are commonly used by commercial After Effects plugins?

Popular commercial AE plugins like Element 3D and Helium use OpenGL (Element 3D uses OpenGL 4.x and Helium uses OpenGL 3.3). Many plugin developers continue to rely on OpenGL support on macOS despite its deprecation, hoping that Apple will prepare alternatives before removing it entirely.

*Tags: `gpu`, `opengl`, `macos`, `reference`*

---

## What are the limitations of MoltenVK for After Effects plugin development?

MoltenVK before version 1.3 had several limitations including unsupported texture swizzle and uniforms limitations. However, these issues have been resolved in MoltenVK 1.3 and later versions.

*Tags: `vulkan`, `metal`, `moltenvk`, `macos`, `gpu`*

---

## How can you embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications?

When embedding MoltenVK in an AE plugin, the standard approach of placing it in the framework folder as a dynamic library with JSON in resources/vulkan doesn't work for plugin bundles. The scaleUp plugin demonstrates using xcframework to embed it as a static library inside the plugin binary, though this approach requires careful configuration as Vulkan instance creation may fail if not set up correctly. The key challenge is ensuring proper linking and resource resolution within the plugin bundle rather than relying on system-wide installation in /etc/vulkan.

*Tags: `vulkan`, `macos`, `deployment`, `plugin`, `reference`*

---

## How do you embed MoltenVK as a static library in an After Effects plugin bundle to avoid conflicts with system installations?

When embedding MoltenVK as a static library in an After Effects plugin (xcframework), you may encounter Vulkan instance creation issues. The errors typically relate to missing Vulkan layers (VK_LAYER_KHRONOS_validation) and extensions (VK_KHR_portability_enumeration). This often indicates the MoltenVK_icd is not being correctly located or loaded. Ensure the Vulkan loader is properly configured for static linking. Reference the MoltenVK documentation for macOS development: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos. The scaleUp plugin demonstrates a working approach with embedded MoltenVK and Vulkan as static libraries.

*Tags: `vulkan`, `macos`, `moltenvk`, `static-linking`, `plugin-deployment`, `apple-silicon`*

---

## What is the official MoltenVK documentation for developing Vulkan applications on macOS?

The KhronosGroup MoltenVK GitHub repository contains detailed documentation on developing Vulkan applications for macOS, iOS, and tvOS, with careful guidance on environment setup and integration. Available at: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos

*Tags: `vulkan`, `moltenvk`, `macos`, `reference`, `documentation`*

---

## How do you set up Vulkan development on macOS with MoltenVK compatibility?

To develop Vulkan plugins on macOS with MoltenVK: (1) Update to at least October release to get MoltenVK compatible with Vulkan 1.2. (2) For debug builds, create a target using ICD (Installable Client Driver) in environment variables and dylib linking, then add the "VK_KHR_portability_subset" instance extension (debug-only until MoltenVK uses it natively) along with debugging layers. (3) For release builds, link only to frameworks as static libraries to comply with Vulkan 1.2 limitations.

*Tags: `vulkan`, `macos`, `cross-platform`, `debugging`, `build`*

---

## Why does ImGui fail to initialize OpenGL on macOS with Apple Silicon (M1)?

OpenGL initialization failures on M1 Macs are often related to Apple Silicon compatibility issues. ImGui can run over Metal instead of OpenGL, though converting from OpenGL matrices to Metal requires additional work. The issue may be related to GLSL version differences between macOS and Windows, or Metal API requirements on Apple Silicon.

*Tags: `macos`, `apple-silicon`, `opengl`, `metal`, `debugging`*

---

## How do you print to the console when debugging an After Effects plugin?

On Mac, use printf. On Windows, use OutputDebugStringA macro to output debug messages that will appear in Visual Studio's console. Note that AEGP_WriteToOSConsole from the utility suite exists but may not work reliably. Starting AE with the -debug flag can help, but OutputDebugStringA is the more reliable approach for Windows debugging.

```cpp
OutputDebugStringA("Debug message here");
```

*Tags: `debugging`, `windows`, `macos`, `aegp`*

---

## What causes the 'Invalid Filter 25::3' error on macOS in After Effects?

The 'Invalid Filter 25::3' error on macOS can be caused by a mismatch between the Deployment Target set in Xcode and the macOS version running After Effects. If the Deployment Target is set higher than the OS version (e.g., setting Deployment Target to 12.3 when running on Big Sur 11.17.9), it will cause this error.

*Tags: `macos`, `debugging`, `build`, `deployment`*

---

## How can you add conditional logging to an After Effects plugin without slowing down renders?

Implement a file-based toggle for logging by checking for a specific file (e.g., 'pixel_sorter_log.txt') in global setup at startup. This allows users to enable detailed logging only when needed for debugging, without impacting render performance during normal operation.

*Tags: `debugging`, `macos`, `plugin-development`, `performance`, `logging`*

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

## Why doesn't Instruments detect memory leaks from PF_EffectWorlds allocated in After Effects plugins?

Memory leaks from PF_EffectWorlds may not be detected by Instruments because these objects are allocated deep within the After Effects engine rather than directly within the plugin code. The leak detection tools only see allocations at the plugin level, not at the internal AE engine memory allocation level where EffectWorlds are actually created. This means that even significant memory leaks from unmanaged EffectWorlds won't show up in Instruments despite causing the application to run out of memory.

*Tags: `memory`, `debugging`, `macos`, `aegp`*

---

## Why does macOS Instruments fail to detect memory leaks from undisposed PF_EffectWorlds in After Effects plugins?

James Whiffin reported that while Instruments successfully detected memory leaks from data allocated in pre-render functions that weren't disposed, it failed to detect leaks from PF_EffectWorlds that were allocated but never disposed. Despite these allocations consuming gigabytes and causing After Effects to run out of memory, they didn't appear in Instruments' leak detection output even when sorted by size. This suggests that PF_EffectWorlds may be allocated through memory management mechanisms that Instruments cannot track, or that After Effects' memory management for effect worlds operates outside standard system memory tracking.

*Tags: `memory`, `macos`, `debugging`, `aegp`*

---

## How do you disable the crash warning popup in After Effects 2024?

Press CMD+F12 to open the Debug Database console. Click the hamburger menu next to the console and change to Debug Database. Search for 'ShowPreviousCrashWarning' and turn it off. This setting can also be configured in the prefs.txt file and can be scripted for automation.

*Tags: `debugging`, `macos`, `scripting`, `ui`*

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

## Why might an AEGP not appear in After Effects and fail to hit any breakpoints during debugging?

This is a Mac-specific AEGP question about plugin visibility and debugging. The issue could be related to security settings or allocation changes on macOS, particularly around code signing, entitlements, or plugin sandbox restrictions. Possible causes include: incorrect plugin bundle structure, code signing issues, missing entitlements, or the plugin not being properly registered in After Effects' plugin directory.

*Tags: `aegp`, `macos`, `debugging`, `deployment`*

---

## What GLSL assignment operators are supported across different GPU platforms in After Effects plugins?

James Whiffin encountered an issue where a specific GLSL operator caused compilation failures. The error was identified through shader compilation feedback. The conversation indicates that while some assignment operators like += work correctly, the /= operator caused issues on at least one platform (Windows or Mac, GPU reference unknown). This suggests developers need to be cautious about which GLSL assignment operators are universally supported across different GPU architectures when developing AE plugins.

*Tags: `gpu`, `opengl`, `glsl`, `debugging`, `cross-platform`, `macos`, `windows`*

---

## What memory management issues can occur when using the Compute Cache API on macOS during export?

When caching large data structures (e.g., 100MB memory handles) per frame during export on macOS, memory handles may not be destroyed quickly enough in the delete callback function, leading to RAM accumulation and crashes during long exports. Proper memory management in the MyDeleteComputeValueFunc callback is critical to avoid this issue.

*Tags: `compute-cache`, `memory`, `macos`, `caching`*

---

## What are alternatives to Nuitka for compiling Python code in After Effects plugins, especially for resolving dependency conflicts on macOS?

Cython is a viable alternative to Nuitka for compiling Python code in AE plugins. It can be used to compile specific portions of Python code and has fewer dependency conflicts compared to Nuitka, particularly when dealing with libraries like NumPy on macOS (which can have issues with libopenblas.0.dylib).

*Tags: `scripting`, `macos`, `build`, `cross-platform`, `deployment`*

---

## How can you compile .rc resource files on macOS for After Effects plugins?

On macOS, use the native macOS tool Rez to compile .rc resource files. This approach can be integrated into build systems like CMake to handle resource compilation as part of the plugin build process.

*Tags: `build`, `macos`, `pipl`, `resource-compilation`*

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

## How can you reliably get the After Effects version in a plugin when in_data->version major and minor don't update between releases?

James Whiffin noted that in_data->version major and minor fields are not being updated reliably between AE 2024 and 2025, returning the same values. This suggests developers should investigate alternative methods for version detection, such as querying the application directly or using other fields in the plugin data structure, though no specific solution was provided in this conversation.

*Tags: `aegp`, `debugging`, `macos`, `windows`*

---

## How do you debug 'failed to load' errors in After Effects and 'filter offline' errors in Premiere Pro?

When debugging plugin loading failures, check whether the error occurs before global_setup is called. If it happens before global_setup, it may indicate a library dependency issue loaded by AE/Premiere in a newer version that conflicts with your plugin. Test on both Windows and macOS to determine if the issue is platform-specific, as the same plugin may work on one platform but fail on another.

*Tags: `debugging`, `deployment`, `cross-platform`, `premiere`, `macos`, `windows`*

---

## Why are plugins failing to load on Apple Silicon Macs in After Effects 25.2 and 25.3?

According to investigation by Maxon and Adobe, the issue appears to be related to code signing. Apple may have introduced stricter code-signing requirements or hardening behavior in newer Xcode versions. There seems to be a potential interaction between After Effects' own code signatures and plugin signatures, though the exact cause is still under investigation. Even plugins that are properly signed and notarized are experiencing load failures on Apple Silicon Macs.

*Tags: `macos`, `apple-silicon`, `code-signing`, `deployment`, `debugging`*

---

## Is there a reference Adobe forum post discussing plugin loading failures on macOS?

Yes, there is an Adobe Community forum post discussing plugin load failures in After Effects 25.2 and 25.3 on macOS: https://community.adobe.com/t5/after-effects-beta-discussions/my-plugins-fails-to-load-in-25-2-and-25-3-on-macos/m-p/15192946. This post documents the issue where plugins don't appear to be signed correctly, which is now a requirement on Apple Silicon Macs.

*Tags: `macos`, `apple-silicon`, `deployment`, `reference`, `debugging`*

---

## How do you debug Rust bindings for After Effects on macOS with symbols?

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

*Tags: `debugging`, `macos`, `rust`, `open-source`*

---

## Is there an open-source Rust binding library for After Effects plugin development?

The virtualritz/after-effects repository on GitHub provides Rust bindings for After Effects plugin development. It enables developers to write AE plugins in Rust with support for debugging on macOS using standard debuggers.

*Tags: `rust`, `open-source`, `reference`, `macos`*

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

## How can you reliably get the current After Effects or Premiere version number (e.g., 2025)?

James Whiffin noted that in_data->version major and minor fields are not being reliably updated between versions like 2024 and 2025, returning the same values. Jonah reported that accessing the version appears to work on macOS After Effects, though behavior on Premiere and Windows still needs verification.

*Tags: `aegp`, `cross-platform`, `macos`, `windows`, `premiere`, `debugging`*

---

## How can you implement custom breakpoint assertions in C++ for After Effects plugin development without crashing the program?

Create custom assertion macros that automatically trigger breakpoints only when a debugger is attached, rather than using standard assertions that crash the program. This can be implemented using preprocessor macros with platform-specific code paths for Windows and macOS. While C++26 will have built-in support for this feature, it can be implemented today using inline assembly calls with preprocessor directives.

*Tags: `debugging`, `cross-platform`, `windows`, `macos`, `build`*

---

## How do you debug ExtendScript for After Effects on macOS?

ExtendScript debugging on macOS has traditionally been difficult, requiring the Intel version which is slow. However, Adobe recently updated the debugger (about a month ago) to be Apple Silicon native, which is a significant quality of life improvement for macOS developers.

*Tags: `scripting`, `debugging`, `macos`, `apple-silicon`*

---

## Why does a plugin work in the Plugins folder but not the Mediacore folder on Mac After Effects 26.0.0?

Multiple developers reported this issue with After Effects 26.0.0 on Mac where plugins load correctly from the Plugins folder but fail to load from the Mediacore folder. The Gaussian Splat team issued an update to address AE 26 compatibility, suggesting there are specific compatibility changes in AE 26 affecting Mediacore plugin loading on Mac. The issue also affects Premiere Pro 2026, where plugins in Mediacore don't appear at all, even though they work in earlier Premiere versions (2023-2025) and in AE 2026.

*Tags: `macos`, `deployment`, `cross-platform`, `mfr`, `debugging`*

---

## What historical CPU architecture values were stored in the Gestalt API for After Effects plugins?

The Gestalt API values held gestalt values for CPU and FPU on 68x and PowerPC architectures. While the Gestalt API was deprecated on macOS since 10.8, Apple added values for Intel and Silicon ARM CPUs (values 10 and 20). However, Adobe no longer passes these values through to plugins.

*Tags: `macos`, `apple-silicon`, `deprecated`, `reference`, `architecture`*

---

## Should obsolete CPU architecture fields have been removed from in_data when After Effects went 64-bit only?

Yes, the fields in in_data related to old CPU architecture detection (such as Gestalt values for 68x and PowerPC) should have been removed from the plugin API when Adobe transitioned to 64-bit only support around CS5, rather than keeping legacy fields that are no longer used or meaningful.

*Tags: `aegp`, `deprecated`, `backwards-compatibility`, `macos`, `windows`*

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

## Why does ImGui fail to initialize OpenGL loader on macOS?

The issue was related to window flag initialization order. On macOS, window flags need to be set before window creation, not after. This differs from Windows behavior where setting flags after creation may work. Ensure all window configuration flags are applied before creating the window.

*Tags: `macos`, `opengl`, `ui`, `debugging`*

---

## Why is rowbytes alignment important for pointer dereferencing in After Effects plugins?

Dereferencing a pointer that is not correctly aligned for the referenced type results in undefined behavior according to the C specification (6.3.2.3 7 in the C spec https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3047.pdf). If rowbytes was not a multiple of alignof(Pixel Type), you would be dereferencing misaligned pointers everywhere, which is UB. Apple documents this issue at https://developer.apple.com/documentation/xcode/misaligned-pointer.

*Tags: `memory`, `aegp`, `macos`, `debugging`*

---

## How do you handle Unicode characters in After Effects plugin parameter names and categories on macOS?

The name field is defined as 'A_char[32]', so you need to convert your string to UTF-8 and feed it to the param macro. Make sure the length of the resulting UTF-8 string is no more than 31 characters long (including multibyte characters), as the last character must be reserved for null termination. One successful approach mentioned was using ConvertUTF8ToGBK() in a UTF-8 encoded .cpp file when working with XCode.

*Tags: `macos`, `params`, `unicode`, `build`, `xcode`*

---

## Why does AEGP_GetLayerNumMasks return 0 when a mask is clearly set on the layer in After Effects?

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

*Tags: `aegp`, `mask`, `debugging`, `macos`*

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

## What resources are available for implementing hot reloading in Xcode development?

For Xcode development, refer to the Stack Overflow discussion on instant run or hot reloading: https://stackoverflow.com/questions/42529081/instant-run-or-hot-reloading-for-xcode. Note that hot reloading was more common in 32-bit builds but became less practical with the transition to 64-bit architecture. Limitations exist when hot reloading is used with After Effects plugins, particularly around parameter setup.

*Tags: `debugging`, `build`, `macos`, `reference`, `tool`*

---

## How do I make the idle hook get called more frequently to support higher frame rate precomp playback?

Use the AEGP_CauseIdleRoutinesToBeCalled function instead of trying to manually set max_sleepPL. This will cause idle routines to be called more frequently. Note that on macOS the call rate appears to be capped at around 30 calls per second, while on Windows it can reach 60 calls per second.

*Tags: `aegp`, `idle-hook`, `macos`, `windows`, `cross-platform`*

---

## What is example code for converting seconds to an A_Time value in Objective-C?

Here is sample Objective-C code that converts from seconds to A_Time by calculating a timescale based on the composition's framerate: Create a timescale by multiplying framerate by 100, then construct the A_Time with `A_Time time = {seconds * timescale, timescale};`. You can retrieve the framerate using `AEGP_GetCompFramerate()` from the CompSuite.

```cpp
+ (A_Time) timeFromSeconds:(CFTimeInterval)seconds {
  AEGP_CompH comp = [self activeComposition];
  A_FpLong framerate = [self framerateFromComp:comp];
  A_FpLong timescale = framerate * 100;
  A_Time time = {seconds * timescale, timescale};
  return time;
}
```

*Tags: `aegp`, `params`, `macos`*

---

## How can I display text in an After Effects plugin render buffer?

After Effects' API doesn't offer built-in text generating functions for the render buffer (unlike the UI buffer which has the drawbot suite). However, you can fill the render buffer using OS-native text tools: use GDI+ on Windows or Quartz on macOS to generate text in a native OS buffer, then copy that content back to After Effects' render buffer.

*Tags: `ui`, `windows`, `macos`, `render-loop`, `output-rect`*

---

## What language and tools are needed to create After Effects plugins?

After Effects plugins are written in C/C++. While you can work with other languages, all calls to the AE API must be in C/C++. The SDK is shipped with Visual Studio project files for Windows and Xcode project files for Mac. Effect controls windows and composition window overlay graphics are created using the DrawBot suite supplied by the AE SDK.

*Tags: `sdk`, `c/c++`, `windows`, `macos`, `ui`, `build`*

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

## Where did the 'About' button go for plugins in After Effects 2020?

The 'About' button was deliberately hidden in After Effects 2020. It is still accessible by right-clicking the effect name and choosing 'About' from the context menu. As an alternative, developers can use PF_SetOptionsButtonName() to implement an options button, which will be displayed next to the 'Reset' button.

*Tags: `ui`, `macos`, `pipl`, `debugging`*

---

## How can I get the After Effects version string (e.g., 'After Effects CC 2017') in my plugin?

There is no direct SDK function to retrieve the version string. Instead, use PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH_W, wPath) to get the path to your plugin files, then decipher the AE version from the file path. Alternatively, look for the 'Presets' folder which is always in the same directory as AfterFX.exe. You can also check the Windows registry at HKEY_LOCAL_MACHINE\SOFTWARE\Adobe\After Effects\ or the macOS library at /Library/Preferences/com.Adobe.After Effects.paths.plist.

```cpp
A_UTF16Char wPath[AEFX_MAX_PATH];
PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH_W, wPath);
```

*Tags: `aegp`, `pipl`, `windows`, `macos`, `sdk`*

---

## How should you diagnose a plugin crash that only occurs in After Effects CC 2015+ but not CC 2014, particularly when it involves custom UI?

Ask the user to send over their old After Effects preferences folder and then delete the pref folder before relaunching AE. If resetting the preferences solves the issue, try reproducing the issue on your machine using the user's old preferences. This approach has a high probability of resolving the issue. If successful, the old prefs can help identify what setting or cached data was causing the crash.

*Tags: `ui`, `debugging`, `macos`, `preferences`*

---

## How do you prevent sequence data from being lost during the flatten call when saving projects?

In After Effects versions before CC2017, you need to purge unwanted sequence data during the flatten call and reconstruct it during resetup. Starting with CC2017 and later, a new API call allows you to keep the original sequence data while delivering a flattened copy, preventing data loss. However, this new call is currently only implemented for UI event calls and not for copy, duplicate, or save operations. Eventually it will replace the older flatten call entirely.

*Tags: `sequence-data`, `aegp`, `params`, `macos`, `windows`*

---

## Why does AEGP_ExecuteScript cause a crash on macOS but not Windows when closing After Effects?

The crash is likely caused by something in the script itself, not the AEGP_ExecuteScript function. Use binary search debugging: delete half of your script and run again, repeating until the error disappears to isolate the problematic code. In the reported case, the issue was caused by using dlg.close() instead of dlg.hide() on macOS, which behaves differently between platforms.

*Tags: `aegp`, `scripting`, `macos`, `debugging`, `cross-platform`*

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

## How do you debug an After Effects plugin in Xcode?

Set the Xcode executable path to the After Effects application, then hit Cmd+R to build and run a debug session. Make sure the build location is the same place from which AE loads the plugin—you can create an alias from AE's plugins directory to the build location, and AE will scan the alias. Note that param and global setup are only done when you first apply an effect to a layer, so changes to those require restarting AE. Quick fix/continue functionality was removed in the transition to x64 and did not return.

*Tags: `debugging`, `macos`, `build`*

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

## What sample plugin should be used as a reference for building custom output device plugins?

The EMP (External Monitor Preview) sample plugin is the recommended reference for creating custom output devices in After Effects. This sample has been discussed in the Adobe forums and is included in the SDK documentation. On macOS, developers may also want to examine the Quicktime VOUT sample for additional reference.

*Tags: `output-rect`, `aegp`, `reference`, `macos`, `open-source`*

---

## How can I access the plugin's file path from within a Mac After Effects AEGP plugin to load bundled resources?

For EFFECT plugins, use PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH, path) after including AE_EffectCB.h. This requires the in_data structure which is only available to EFFECT plugins, not AEGP plugins. You may need to convert colons to forward slashes and add "VOLUMES/" at the beginning on Mac. For AEGP plugins (which don't receive in_data), alternative approaches include using getcwd() to read/write files by name only without a full path, or on Windows using GetModuleFileName((HINSTANCE)&__ImageBase, folder, sizeof(folder)).

```cpp
char path[1024] = {'\0'};
PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH, path);
```

*Tags: `macos`, `aegp`, `plugin-resources`, `file-access`, `cross-platform`*

---

## Are After Effects plugins cross-platform compatible between Mac and Windows?

Yes, plugins developed for After Effects are generally cross-platform compatible. Over 99% of the After Effects API is completely cross-platform without special notes. The main differences come when using OS-level APIs (such as opening OS-level windows for options). For core functionality like handling buffers, pixels, RAM, and API suites, the same code can be used on both platforms. However, there are several platform-specific considerations: Windows defines 'long' as 32-bit while Mac defines it as 64-bit (best practice is to use Adobe API types), Windows uses backslashes in paths while Mac uses forward slashes, Windows uses UTF-16 for file operations while Mac uses UTF-8, there are byte order issues for image manipulation, and the build outputs differ (Visual Studio generates a single file while Xcode generates a bundle). For best results, have developers familiar with each platform handle their respective ports.

*Tags: `cross-platform`, `macos`, `windows`, `api`*

---

## What type-definition practices should be used for cross-platform After Effects plugin development?

When developing cross-platform After Effects plugins, it is best practice to use Adobe API-defined types rather than native OS types. This is because Adobe has carefully designed their type definitions to handle platform differences correctly. For example, the native 'long' type is defined as 32-bit on Windows but 64-bit on Mac, which can cause compatibility issues. By using Adobe's API type definitions throughout your code, you avoid these pitfalls and ensure consistent behavior across platforms.

*Tags: `cross-platform`, `api`, `macos`, `windows`*

---

## How do you handle coordinate system differences when drawing UI elements in Quartz on macOS After Effects plugins?

Quartz uses a different coordinate system than QuickDraw on macOS, with the Y-axis inverted and origin at bottom-left instead of top-left. To handle this, use CGContextConvertPointToUserSpace to get the flipped origin point, then adjust your Y coordinates accordingly. The solution involves getting the flip origin once per draw call and incorporating it into your transformation matrix: CGPoint zero = {0,0}; CGPoint flipOrigin = CGContextConvertPointToUserSpace(context, zero); flippedY = flipOrigin.y - (originaY - event_extraP->u.draw.update_rect.top);

```cpp
CGPoint zero = {0,0};
CGPoint flipOrigin = CGContextConvertPointToUserSpace(context, zero);
flippedY = flipOrigin.y - (originaY - event_extraP->u.draw.update_rect.top);
```

*Tags: `macos`, `ui`, `drawing`, `quartz`, `coordinate-systems`*

---

## How do you draw transformed circular handles in macOS After Effects plugins accounting for layer scale, rotation, and position?

Use Quartz context transformation matrices to apply layer-to-frame transformations. First get the layer2frame transformation matrix using get_layer2comp_xform and source_to_frame callbacks. Then create a CGAffineTransform that includes the Y-axis inversion correction, and apply it to the context using CGContextConcatCTM before drawing. The Y-inversion adjustment should negate the b and d components and adjust the ty translation by the flipped origin offset.

```cpp
float a = xform.mat[0][0];
float b = xform.mat[0][1];
float c = xform.mat[1][0];
float d = xform.mat[1][1];
float tx = xform.mat[2][0];
float ty = xform.mat[2][1];
CGPoint origin = CGPointMake(0,0);
CGPoint originFlipped = CGContextConvertPointToUserSpace(context, origin);
CGAffineTransform trans = CGAffineTransformMake (a, -b, c, -d, tx, originFlipped.y - (ty - event_extraP->u.draw.update_rect.top));
CGContextConcatCTM (context, trans);
CGRect rect = CGRectMake ( center.x - cr, center.y - cr, 2*cr, 2*cr);
CGContextAddEllipseInRect(context, rect);
CGContextStrokePath(context);
```

*Tags: `macos`, `ui`, `quartz`, `drawing`, `transformation`, `matrix`*

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

## What is an alternative to Interface Builder for creating simple dialogs on macOS in After Effects plugins?

Use the CFUserNotification API, which is C-based and simpler than Interface Builder and Objective-C. It's suitable for basic dialogs with string input and 1-2 text fields. Refer to Apple's CFUserNotification documentation at http://developer.apple.com/mac/library/documentation/CoreFoundation/Reference/CFUserNotificationRef/Reference/reference.html

*Tags: `ui`, `macos`, `reference`*

---

## How do you configure Debug and Release build configurations in Xcode for After Effects plugins?

Duplicate the Debug configuration in Xcode and rename it to Release. Remove the '_DEBUG' preprocessor macro from the Release version. Enable 'Strip debug symbols during copy' in the Release configuration to reduce binary size. Additional optimizations may be applied depending on your specific needs.

*Tags: `build`, `macos`, `debugging`*

---

## How do you get resize callbacks for a dockable plugin window in macOS?

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

*Tags: `ui`, `macos`, `debugging`*

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
