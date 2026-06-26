# Q&A: apple-silicon

**26 entries** tagged with `apple-silicon`.

---

## How do you build AE plugins for Apple Silicon (ARM64)?

You can build on an M1 Mac (even a cheap Mac Mini works as a build machine). It's not strictly required to build on M1, but it helps for testing. The key thing many people miss is that you must add ARM64 to your PiPL resource file - just building as Universal Binary in Xcode is not enough. Without the ARM64 entry in the PiPL, AE will show the 'not yet compatible' warning.

*Contributors: [**gabgren**](../contributors/gabgren/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/), [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2023-08-18 · Tags: `apple-silicon`, `arm64`, `pipl`, `xcode`, `mac`, `universal-binary`*

---

## What caused plugins to fail to load in AE 25.2 on macOS?

The issue was related to code signature validation. Adobe built AE 25.2 with a newer Xcode that enforces stricter code signature checks. Multiple root causes were identified: (1) Unsigned plugins now fail to load on Apple Silicon. (2) Installers that don't properly clean up old files can break code signatures - if v1 had file X and v2 removes it, but the installer leaves it, the signature breaks. (3) Some plugins had 0-byte .plugin files after the AE upgrade. Solution: Ensure proper code signing, notarization, and clean installation (remove all old files before installing new version).

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**gabgren**](../contributors/gabgren/), [**wunk**](../contributors/wunk/) · Source: adobe-plugin-devs · 2025-04-08 · Tags: `code-signing`, `macos`, `ae-25-2`, `notarization`, `installer`, `apple-silicon`*

---

## How do I build an After Effects plugin for Apple Silicon (M1/ARM64)?

You need to do two things: (1) Set the Xcode build architecture to include arm64 (ensure the architecture setting targets ARM), and (2) Add 'CodeMacARM64 {"EffectMain"}' to your PiPL resource file. Without the PiPL entry, AE won't recognize the plugin as ARM-native even if the binary is compiled for arm64.

*Contributors: [**Nate**](../contributors/nate/) · Source: adobe-plugin-devs · 2022-04-28 · Tags: `apple-silicon`, `m1`, `arm64`, `pipl`, `xcode`, `mac`, `build-configuration`*

---

## Is it necessary to build on Apple Silicon hardware to support it?

It is not strictly required to build on Apple Silicon hardware, but it is best practice to test on the actual platform. Purchasing an M1 Mac mini is a practical and cost-effective way to validate Apple Silicon support.

*Tags: `apple-silicon`, `macos`, `build`*

---

## What is required to support M1 Macs when including external libraries like OpenCV in an After Effects plugin?

When building for M1 with external libraries, ensure the library is compiled with appropriate Mac support options. If using dynamic libraries (dylib), you need to embed them in the plugin file and specify rpath in Xcode. Static libraries (.a files) should be linked directly. The Missing Entry Point error typically occurs when the plugin is applied, not during loading, and may indicate a PiPL configuration issue rather than a linking problem. It's recommended to test on both M1 and x86 Mac architectures to isolate platform-specific issues, as the problem may be related to project configuration rather than the external library itself.

*Tags: `apple-silicon`, `macos`, `build`, `pipl`, `deployment`*

---

## How do you set up MoltenVK compatibility with Vulkan 1.2 on macOS for After Effects plugins?

Get at least the October update to get a MoltenVK compatible with Vulkan 1.2. For debug builds, create a target using ICD in environment with dylib linking, add "VK_KHR_portability_subset" to instance extensions (debug only for now until MoltenVK uses it), and include debugging layers. For release builds, link to frameworks only as static libraries to work within Vulkan 1.2 limitations.

*Tags: `vulkan`, `macos`, `apple-silicon`, `build`, `debugging`*

---

## How to fix OpenGL loader initialization failure on M1 Mac with ImGui?

The issue is likely related to M1 chip compatibility. ImGui can run over Metal instead of OpenGL, which would require conversion from OpenGL matrices but would be the proper approach for Apple Silicon. The error may be similar to previous issues with GLSL version differences between Mac and Windows, but specific to ARM64 architecture on M1.

*Tags: `opengl`, `metal`, `macos`, `apple-silicon`, `debugging`*

---

## Why does a plugin built for any architecture show a compatibility warning in After Effects even though it's not running in Rosetta mode?

The conversation identifies this as an issue but does not provide a resolution. The user reports that ArchiCheck confirms the plugin is built for any architecture, yet After Effects displays a compatibility warning despite not running in Rosetta mode.

*Tags: `macos`, `apple-silicon`, `build`*

---

## What is the correct architecture string to use when building for x64 on Apple Silicon?

The correct string for x64 is x86_64, not x64. If you put an unrecognized string, it builds for your native architecture instead.

*Tags: `build`, `apple-silicon`, `macos`*

---

## Why might a plugin fail to load even though it builds successfully?

The plugin's PiPL resource needs to include ARM64 architecture support. If ARM64 is not added to the PiPL, the plugin won't load properly even if the build succeeds.

*Tags: `pipl`, `apple-silicon`, `build`*

---

## Why are After Effects plugins failing to load on Apple Silicon Macs in versions 25.2 and 25.3?

The investigation suggests code signing requirements have become stricter on Apple Silicon Macs. While entitlements haven't changed, Adobe may be using a newer Xcode version that enforces stricter code signing validation. Additionally, there may be a new hardening behavior in Apple's code-signing that creates a problematic interaction between After Effects' own signatures and plugin signatures.

*Tags: `macos`, `apple-silicon`, `code-signing`, `deployment`*

---

## How do you debug ExtendScript in After Effects?

On Windows, you can use the standard debugging tools. On macOS, the situation was previously difficult because you needed to use the Intel version which was very slow, but Adobe updated the debugger about a month ago to be Apple Silicon native, which is a significant quality of life improvement.

*Tags: `scripting`, `debugging`, `macos`, `apple-silicon`*

---

## Is DirectX rendering already implemented in After Effects production, and has anyone made a plugin using it?

DirectX rendering was not widely available for plugins as of the conversation date. Adobe replaced the UI from OpenGL to DX12 in 2021, and the latest SDK has an official flag for it, but it's likely not yet ready to expose to plugins. It may be added for Windows on ARM device support, where DirectX is the first-party option. Adobe has not been supporting Vulkan, possibly because the version of AFX that passes DX12 handles to plugins hasn't been released yet.

*Tags: `gpu`, `directx`, `windows`, `apple-silicon`, `sdk`*

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

## How do you embed MoltenVK as a static library in an After Effects plugin bundle to avoid conflicts with system installations?

When embedding MoltenVK as a static library in an After Effects plugin (xcframework), you may encounter Vulkan instance creation issues. The errors typically relate to missing Vulkan layers (VK_LAYER_KHRONOS_validation) and extensions (VK_KHR_portability_enumeration). This often indicates the MoltenVK_icd is not being correctly located or loaded. Ensure the Vulkan loader is properly configured for static linking. Reference the MoltenVK documentation for macOS development: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos. The scaleUp plugin demonstrates a working approach with embedded MoltenVK and Vulkan as static libraries.

*Tags: `vulkan`, `macos`, `moltenvk`, `static-linking`, `plugin-deployment`, `apple-silicon`*

---

## Why does ImGui fail to initialize OpenGL on macOS with Apple Silicon (M1)?

OpenGL initialization failures on M1 Macs are often related to Apple Silicon compatibility issues. ImGui can run over Metal instead of OpenGL, though converting from OpenGL matrices to Metal requires additional work. The issue may be related to GLSL version differences between macOS and Windows, or Metal API requirements on Apple Silicon.

*Tags: `macos`, `apple-silicon`, `opengl`, `metal`, `debugging`*

---

## Is there an example of integrating Vulkan with After Effects on Apple Silicon?

Wunk is updating the Vulkanator sample project to demonstrate VulkanAPI (via MoltenVK) integration with After Effects on Apple Silicon, showing how to leverage Vulkan for GPU-accelerated rendering on macOS ARM64 systems.

*Tags: `vulkan`, `gpu`, `apple-silicon`, `metal`, `open-source`*

---

## How should architecture settings be configured in Xcode build settings for ARM and Intel compatibility?

In Xcode build settings, there is an architecture option that allows you to set both ARM and Intel architectures, ARM only, or Intel only. These settings are separate from other build configuration options and should be reviewed carefully when troubleshooting cross-platform compatibility issues.

*Tags: `apple-silicon`, `macos`, `build`, `cross-platform`*

---

## How do you ensure ARM64 architecture support is properly configured in an After Effects plugin build?

You need to add ARM64 to your PiPL (Property List) file. Without this entry, the plugin may not build or load correctly for ARM64 architecture even if the C++ code and Xcode settings are correct.

*Tags: `pipl`, `macos`, `apple-silicon`, `build`*

---

## Why are plugins failing to load on Apple Silicon Macs in After Effects 25.2 and 25.3?

According to investigation by Maxon and Adobe, the issue appears to be related to code signing. Apple may have introduced stricter code-signing requirements or hardening behavior in newer Xcode versions. There seems to be a potential interaction between After Effects' own code signatures and plugin signatures, though the exact cause is still under investigation. Even plugins that are properly signed and notarized are experiencing load failures on Apple Silicon Macs.

*Tags: `macos`, `apple-silicon`, `code-signing`, `deployment`, `debugging`*

---

## Is there a reference Adobe forum post discussing plugin loading failures on macOS?

Yes, there is an Adobe Community forum post discussing plugin load failures in After Effects 25.2 and 25.3 on macOS: https://community.adobe.com/t5/after-effects-beta-discussions/my-plugins-fails-to-load-in-25-2-and-25-3-on-macos/m-p/15192946. This post documents the issue where plugins don't appear to be signed correctly, which is now a requirement on Apple Silicon Macs.

*Tags: `macos`, `apple-silicon`, `deployment`, `reference`, `debugging`*

---

## How do you debug ExtendScript for After Effects on macOS?

ExtendScript debugging on macOS has traditionally been difficult, requiring the Intel version which is slow. However, Adobe recently updated the debugger (about a month ago) to be Apple Silicon native, which is a significant quality of life improvement for macOS developers.

*Tags: `scripting`, `debugging`, `macos`, `apple-silicon`*

---

## Is DirectX rendering currently implemented in After Effects production, and how can it be enabled for plugin development?

DirectX rendering support was added to the After Effects SDK with an official flag, though it appears to be in early stages. Adobe replaced the UI from OpenGL to DX12 in 2021. To enable DirectX compute for debugging and testing, use the 'forcedxcompute' setting in the After Effects console panel under settings, but note that a full computer restart (not just AE restart) is required for the setting to take effect. The feature is likely being developed primarily for Windows on ARM device support, as those devices have first-party support for DX12. Adobe has not yet officially supported Vulkan exposure to plugins.

*Tags: `directx`, `gpu`, `windows`, `debugging`, `apple-silicon`*

---

## What historical CPU architecture values were stored in the Gestalt API for After Effects plugins?

The Gestalt API values held gestalt values for CPU and FPU on 68x and PowerPC architectures. While the Gestalt API was deprecated on macOS since 10.8, Apple added values for Intel and Silicon ARM CPUs (values 10 and 20). However, Adobe no longer passes these values through to plugins.

*Tags: `macos`, `apple-silicon`, `deprecated`, `reference`, `architecture`*

---

## How do you build an After Effects plugin for Apple Silicon M1 Macs?

To build for M1, you need to: (1) Set the appropriate build configuration for Mac ARM64 architecture, and (2) Add CodeMacARM64 {"EffectMain"} to your PiPL (Plugin Property List) file to ensure the plugin is properly registered for native ARM64 execution rather than Rosetta emulation.

```cpp
CodeMacARM64 {"EffectMain"}
```

*Tags: `apple-silicon`, `macos`, `build`, `pipl`*

---
