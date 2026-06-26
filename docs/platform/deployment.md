# Deployment & Signing

> 127 Q&As · source: AE plugin dev community Discord

### What is required to support M1 Macs when including external libraries like OpenCV in an After Effects plugin?

When building for M1 with external libraries, ensure the library is compiled with appropriate Mac support options. If using dynamic libraries (dylib), you need to embed them in the plugin file and specify rpath in Xcode. Static libraries (.a files) should be linked directly. The Missing Entry Point error typically occurs when the plugin is applied, not during loading, and may indicate a PiPL configuration issue rather than a linking problem. It's recommended to test on both M1 and x86 Mac architectures to isolate platform-specific issues, as the problem may be related to project configuration rather than the external library itself.

*Tags: `apple-silicon`, `build`, `deployment`, `macos`, `pipl`*

---

### Should external libraries in After Effects plugins be linked as static or dynamic libraries?

Both approaches are valid. Static libraries (.a files) are linked directly during compilation. Dynamic libraries (dylib) can be used but require embedding in the plugin file and specifying rpath in Xcode build settings to ensure proper loading at runtime.

*Tags: `build`, `deployment`*

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

### How can I embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications using MoltenVK?

You can embed MoltenVK in the plugin's framework folder as a dynamic library with the JSON configuration in resources/vulkan. However, this path approach doesn't work reliably for plugin bundles. An alternative is to use xcframework to embed it as a static library inside the plugin binary, similar to what the scaleUp plugin does. If Vulkan fails to create instances when using the static library approach, verify that the Vulkan layer configuration and initialization is correct for the statically-linked library within the plugin bundle context.

*Tags: `build`, `cross-platform`, `deployment`, `macos`, `vulkan`*

---

### How can MoltenVK be embedded inside an After Effects plugin bundle to avoid conflicts with other MoltenVK applications?

MoltenVK can be embedded as a static library using xcframework within the plugin binary, similar to how the scaleUp plugin does it. However, this approach requires careful configuration of the Vulkan loader and MoltenVK ICD to ensure proper function pointer resolution and layer/extension loading. The main challenge is ensuring the Vulkan loader correctly locates the MoltenVK ICD when statically linked, as opposed to using the standard /etc/vulkan or /usr/local installation paths. Testing should verify that the Vulkan loader is correctly configured for the static linking approach.

*Tags: `build`, `cross-platform`, `deployment`, `macos`, `vulkan`*

---

### What causes VK_ERROR_LAYER_NOT_PRESENT and VK_ERROR_EXTENSION_NOT_PRESENT errors when creating Vulkan instances with embedded MoltenVK in a plugin?

These errors typically indicate that the MoltenVK ICD is not being located correctly by the Vulkan loader, or there are environment configuration issues. When MoltenVK is statically embedded rather than installed system-wide, the Vulkan loader may not find the necessary validation layers and extensions. This is particularly challenging in sandboxed or non-standard installations where the LunarG Vulkan SDK is not in the default /usr/local path. The solution involves ensuring the Vulkan loader can properly locate the MoltenVK ICD configuration.

*Tags: `debugging`, `deployment`, `macos`, `vulkan`*

---

### Does the Stable Diffusion plugin include the SD model or fetch it externally?

The plugin does not ship with the model. Instead, it pulls the model from Hugging Face on the first use.

*Tags: `deployment`, `scripting`*

---

### Can a deployment target mismatch in Xcode cause an 'Invalid Filter' error on macOS?

Yes, setting a deployment target higher than the user's OS version will cause that error. The user had deployment target 12.3 in Xcode but the user was on Big Sur 11.17.9, which is lower than the deployment target.

*Tags: `debugging`, `deployment`, `macos`*

---

### Is there a way to get the file path of the current After Effects project?

There is no direct API to get the project file path in After Effects. The question notes that this is ambiguous until the project is saved. The broader use case of associating additional user files with a project and having them participate in the Gather process is not directly supported through standard APIs.

*Tags: `aegp`, `deployment`, `scripting`*

---

### Is there a way to associate additional user files with an After Effects project so they participate in the Gather process?

There is no built-in API to add arbitrary files to a project for inclusion in the Gather process. This would require custom workarounds outside the standard After Effects file format.

*Tags: `aegp`, `deployment`, `scripting`*

---

### What causes an 'entry point not found' error on Windows after a plugin is loaded?

Two main possibilities: a missing dependency (such as a Visual C++ redistributable package) or an error in global setup. The issue may be resolved by installing the Visual C++ redistributable package. In debug mode, you can set a breakpoint on global setup to verify if it's being called.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### Is it normal to see msvcp140d.dll when compiling with Visual Studio 2022 (v143)?

Yes, this is normal. The msvcp140d.dll appears in compilations with Visual Studio 2022 (v143). However, if the entry point is not found, it typically indicates a dependency issue rather than a problem with this DLL itself.

*Tags: `build`, `deployment`, `windows`*

---

### What macOS and Xcode version compatibility issues occur when updating macOS?

When updating macOS (e.g., from macOS 13.x to 14), you are often forced to update Xcode to a compatible version. For example, updating to macOS 14.1.1 required updating to Xcode 15. This tight coupling between macOS and Xcode versions is more restrictive than Windows/Visual Studio, which is more flexible about version compatibility.

*Tags: `build`, `deployment`, `macos`*

---

### What development strategy should be used to test plugin compatibility across macOS versions?

The best approach is to maintain separate development and testing machines: one dev Mac with the target macOS version and a separate test Mac updated to the latest version. This allows developers to confirm that products run on new OS versions and debug any version-specific bugs that only occur on the latest release. However, this is an expensive option.

*Tags: `build`, `debugging`, `deployment`, `macos`*

---

### Does Media Encoder load AEGP plugins to render After Effects projects?

No, AEGP plugins are not supported by Media Encoder (AME). If your plugin needs an AEGP plugin to render, you will need to get your AEGP as a standalone/command line tool to work around this limitation.

*Tags: `aegp`, `deployment`, `premiere`*

---

### What causes memory issues when using the Compute Cache API on macOS during export?

On macOS, when caching large amounts of data (e.g., 100MB per frame) during export, the memory handles are not destroyed quickly enough, causing RAM to become overloaded and eventually crash on long exports. The issue is that the delete function may not be called frequently enough to free the cached memory handles during the export process.

*Tags: `compute-cache`, `deployment`, `macos`, `memory`*

---

### Have you used nuitka for Python compilation on macOS with numpy dependencies?

One developer tried nuitka once but had limited experience with it. They preferred using Cython for compilation instead to handle their specific needs.

*Tags: `deployment`, `macos`, `scripting`*

---

### Are there licensing restrictions around Blender plugins or their marketplace?

There are some licensing restrictions around Blender plugins, particularly if you use their marketplace.

*Tags: `deployment`, `licensing`*

---

### How do you compile .rc files on macOS for After Effects plugins?

On macOS, you can use the native macOS tool called Rez to compile .rc files. The virtualritz/after-effects project demonstrates this approach as part of its build system. The Rust PiPL compiler from that same project can also handle resource compilation as part of a cross-platform build setup.

*Tags: `build`, `deployment`, `macos`, `pipl`*

---

### Is using f32 (32-bit float) data type safe and well-supported in After Effects plugin development?

Yes, f32 support is safe and essential for After Effects plugins, particularly those integrating 3D renderers. This has been validated through practical experience with production plugins like AtomKraft for AE and other Rust-based AE integrations that have run in production for extended periods.

*Tags: `build`, `deployment`, `gpu`, `threading`*

---

### How can a plugin require AVX2 support or handle the AVX2 cutoff in After Effects?

After Effects has required AVX2 since version 2023. Given that Steam hardware survey shows 98.25% support for AVX and 96.30% support for AVX2 (with growing adoption rates), targeting AVX2 is practical for modern plugins. Rather than wrapping every dispatch with if(HasAVX()) checks, you can compile your plugin with AVX2 as a baseline requirement, allowing you to leverage AVX2 optimizations throughout your codebase without conditional branching. This approach can potentially double performance compared to targeting x86-64-v2 (up to SSSE3).

*Tags: `build`, `deployment`, `performance`, `windows`*

---

### How can a plugin support AVX2 requirements given After Effects 2023+ requires it, and should developers worry about older CPU compatibility?

After Effects has required AVX2 since version 2023. Based on Steam hardware survey data, 96.30% of systems support AVX2 (growing 1.63% per month) and 98.25% support AVX (growing 0.77% per month), though this may not directly overlap with video editing users. Most developers using After Effects in 2025, even older versions, are unlikely to encounter machines without AVX2 support. For selective CPU feature support, ISPC (Intel SPMD Program Compiler) can automatically compile multiple versions and switch between them at runtime.

*Tags: `build`, `cpu`, `cross-platform`, `deployment`, `windows`*

---

### How can you determine the After Effects version at runtime?

You can use a 'dirty solution' by locating the path of the running binary host using AEGP_GetPluginPaths with AEGP_GetPathTypes_APP as the path type. From this you can extract the AE folder name or version from the binary's metadata. Alternatively, you can try mapping AEGP_GetDriverSpecVersion to the AE version number.

```cpp
AEGP_GetPluginPaths(plugin_id, AEGP_GetPathTypes_APP, &path_outH)
```

*Tags: `aegp`, `debugging`, `deployment`*

---

### Does the AEGP version increment in parallel with the host After Effects version?

This was posed as a question about whether AEGP version numbering follows a parallel system to the host version, but no definitive answer was provided in the conversation.

*Tags: `aegp`, `deployment`*

---

### How do you debug a plugin that fails to load in After Effects and appears offline in Premiere?

Check if the error occurs before global_setup is called. If it happens before that point, it may be due to a library dependency that AE loaded in a new version that your plugin uses. Verify whether the issue occurs on both Windows and macOS or only on specific platforms.

*Tags: `debugging`, `deployment`, `macos`, `pipl`, `windows`*

---

### What could cause a simple C++ plugin with no external dependencies to fail loading?

Even simple plugins without explicit dependencies can fail due to library dependencies loaded by After Effects itself in newer versions. The issue may be platform-specific, as it can work on one platform (like Windows) while failing on another (like macOS).

*Tags: `cross-platform`, `debugging`, `deployment`, `macos`, `windows`*

---

### What is the minimal macOS SDK version required for After Effects plugins?

The minimal macOS SDK requirement was not specified in the conversation. The question was asked but not answered.

*Tags: `build`, `deployment`, `macos`*

---

### Where is the DebugDatabase.txt file located in the latest After Effects beta release?

The DebugDatabase.txt file is in the wrong location in the latest beta release. The After Effects team is working on fixing this issue.

*Tags: `debugging`, `deployment`*

---

### What is the nature of the issues reported by the Red Giant team with the pre-release version?

The issues are related to code signature validation. Something is getting corrupted somewhere and the validation process is causing the problem. Red Giant is in contact with Adobe and the issue is being investigated, but a full picture is still being determined.

*Tags: `build`, `debugging`, `deployment`*

---

### What macOS SDK was used to compile the plugins?

The conversation discusses whether plugins were compiled with a newer macOS SDK with an updated version, noting differences between Adobe's ExporterAIFF.plugin between versions 25.1 and 25.2, but no definitive answer is provided about which specific SDK version was used.

*Tags: `build`, `deployment`, `macos`*

---

### Why are some plugin files becoming 0 bytes after upgrading to After Effects 2025.2?

Multiple plugins were found to be 0 bytes (emptied) after upgrading to AE 2025.2, while released versions work properly. This appears to be a bug in the 2025.2 release that corrupts plugin files in the MediaCore folder during upgrade, though the root cause was not determined in the conversation.

*Tags: `build`, `deployment`, `macos`, `windows`*

---

### Why are After Effects plugins failing to load on Apple Silicon Macs in versions 25.2 and 25.3?

The investigation suggests code signing requirements have become stricter on Apple Silicon Macs. While entitlements haven't changed, Adobe may be using a newer Xcode version that enforces stricter code signing validation. Additionally, there may be a new hardening behavior in Apple's code-signing that creates a problematic interaction between After Effects' own signatures and plugin signatures.

*Tags: `apple-silicon`, `code-signing`, `deployment`, `macos`*

---

### How can you code sign and notarize After Effects plugins for macOS?

You need an Apple Developer account (costs $99/year). With this account, you can sign and notarize your plugins. The process involves using Xcode's code signing tools to sign the plugin executables and then submitting them to Apple's notarization service.

*Tags: `code-signing`, `deployment`, `macos`*

---

### How can installer leftovers from previous versions break code signing on macOS?

When an installer overwrites files instead of properly uninstalling, leftover files from the previous version remain in the package. Any extra files in a signed package will break the code signature. For example, if version 1 has file X and version 2 removes it, but the installer leaves file X behind, the code signature breaks. The solution is to completely uninstall and remove all files before installing the new version, rather than just overwriting files.

*Tags: `build`, `codesign`, `deployment`, `macos`*

---

### Is the After Effects code signing issue affecting binary plugins on Windows as well as macOS?

The binary code signing issue appears to be Mac-only. It's related to After Effects being built with the latest Xcode, which applies stricter code signature checks than before. There was a separate ZXP signing issue that affected both macOS and Windows platforms, but that's a different problem related to the ZXPSignCmd tool.

*Tags: `codesign`, `deployment`, `macos`, `windows`*

---

### Does notarization ensure that adding files to a macOS package won't break code signing?

No. Notarization does not prevent code signature breakage from modified package contents. Even after notarization, if you add or leave extra files in a signed package (like a dummy.txt file), the code signature will break. Proper uninstallation and clean installation are required to maintain valid code signatures.

*Tags: `codesign`, `deployment`, `macos`*

---

### Is PiPL deprecated and are there alternative ways to describe a plugin entrypoint?

PiPL is considered somewhat deprecated, and there are alternative approaches to describing plugin entrypoints, though the conversation does not specify the exact alternatives mentioned.

*Tags: `deployment`, `pipl`*

---

### What is the first version of After Effects that supports the new plugin format without PIPL?

Based on the conversation, support started with After Effects 2018, though the documentation still references it as a future release at the time of discussion.

*Tags: `build`, `deployment`, `pipl`*

---

### Is a PiPL resource still needed to register After Effects plugins, or can plugins be registered with just the URL info function?

PiPL is still needed even with the PF_Register_effect_ext2 function added in version 23. While you can overwrite some PiPL values in code, the resource file cannot be completely eliminated. After Effects still cannot find effects without a PiPL/rsrc file, though Adobe may update this in the future.

*Tags: `build`, `deployment`, `pipl`, `plugin-registration`*

---

### What are the advantages of writing host and plugin format-independent code across multiple applications?

Writing format-independent code for multiple hosts (AE, Premiere Pro, OFX, Nuke, Frei0r, etc.) frees up significant resources by allowing you to write the processing code once and only occasionally update the wrapper projects, rather than maintaining separate implementations for each host.

*Tags: `code-architecture`, `cross-platform`, `deployment`, `premiere`*

---

### Is there a reliable way to get the current After Effects or Premiere version number (e.g., 2025)?

The in_data->version major and minor fields do not appear to be reliably updated between versions, returning the same values on both 2024 and 2025. This approach may work on Mac AE but needs verification on Premiere and Windows.

*Tags: `debugging`, `deployment`, `macos`, `premiere`, `windows`*

---

### How should I debug the 'plugin wasn't installed correctly' error on Windows 11?

Use dumpbin from Visual C++ to check plugin dependencies. Run 'dumpbin /dependents plugin.aex' in a terminal to see what dependencies are listed. The issue may be related to missing Microsoft Visual C++ runtime package that needs to be installed on the user's system.

```cpp
dumpbin /dependents plugin.aex
```

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### When building a Rust SDK plugin with 200-300 crates, are all dependencies included in the final binary?

The dependencies should be included in the final binary when building with the Rust SDK, but installation errors can occur if required system libraries like Microsoft Visual C++ runtime are missing on the target system.

*Tags: `build`, `deployment`, `windows`*

---

### How do you create a render-only version of a plugin that doesn't require additional licenses for render farms?

One approach is to make all parameters invisible while still allowing the plugin to read them, creating a read-only version of the plugin that can process renders without displaying the GUI interface.

*Tags: `deployment`, `pipl`, `render-loop`, `ui`*

---

### How can you check if the After Effects renderer is running in headless mode?

Use the AppSuite4 API with the PF_IsRenderEngine function. Create a PF_Boolean variable and call suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine) to set it to true if running in render engine mode.

```cpp
PF_Boolean bIsRenderEngine = true;
suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine);
```

*Tags: `aegp`, `debugging`, `deployment`, `render-loop`*

---

### Would the aegp_idle hook approach work when rendering through MediaEncoder or without a GUI?

No, the aegp_idle hook approach would not work in MediaEncoder or without a GUI, since the idle hook requires an active UI thread to execute.

*Tags: `aegp`, `deployment`, `threading`, `ui`*

---

### Why does a plugin work in the Plugins folder on Mac but not in the Mediacore folder on Mac AE 26.0.0?

This is a known compatibility issue with AE 26.0.0 on Mac. Several developers have reported the same problem. Gaussian Splat and other plugin developers have issued updates to address AE 26 compatibility, suggesting a breaking change in how AE 26 handles plugins in the Mediacore folder on macOS.

*Tags: `build`, `deployment`, `macos`*

---

### Does the Mediacore plugin loading issue affect Premiere Pro 2026?

Yes, plugins in the Mediacore folder work in Premiere Pro 2023, 2024, and 2025, but do not show up or load in Premiere Pro 2026, indicating a similar compatibility issue across Adobe applications.

*Tags: `deployment`, `macos`, `premiere`*

---

### Does PF_OutFlag_I_AM_OBSOLETE flag work to hide a plugin in Premiere Pro while keeping it visible in After Effects?

The flag will hide the plugin in both AE and Premiere Pro, so it won't achieve the desired result of hiding only in PPro unless you conditionally set it after checking the host name. You can make it work by checking the host application first and only applying the flag when running in Premiere Pro.

*Tags: `cross-platform`, `deployment`, `pipl`, `premiere`*

---

### What are the causes of plugins not loading in After Effects 2026?

There is a known issue on Mac where plugins from the Mediacore folder are not recognized and only load from AE's plugin folder. Additionally, AE2026 has GPU compatibility detection issues where it may only detect onboard GPU and refuse installation if it deems the GPU incompatible.

*Tags: `deployment`, `gpu`, `macos`*

---

### Where should After Effects SDK developers share and discover code snippets and solutions?

Stack Overflow with the 'After Effects SDK' tag is recommended as a good system for searching and hosting code snippets. GitHub can also be used for sharing code examples, though the AESDK community is niche. Creating dedicated channels or communities helps organize and grow the knowledge base.

*Tags: `deployment`, `open-source`, `reference`, `scripting`, `tool`*

---

### Why is Lua a better choice than Python for an After Effects script-driven drawing plugin?

Lua is significantly easier to compile into a plugin compared to Python. With Python, the developer had to request permission to install libraries across the system, which is intrusive. Lua, on the other hand, compiles cleanly into the plugin without requiring system-wide library installations. Both languages require similar effort for C bindings without automation or code generation tools, but Lua's compilation model is much cleaner for plugin distribution.

*Tags: `deployment`, `open-source`, `scripting`*

---

### How can you install and activate Adobe After Effects CS6 on Windows 11?

CS6 may have compatibility issues with Windows 11. Try running the installer in compatibility mode, or consider using Windows 10 instead if Windows 11 proves problematic. Adobe's activation servers for CS6 may no longer be functional, which can prevent license key activation on new machines.

*Tags: `cross-platform`, `deployment`, `reference`, `windows`*

---

### What is the status of UXP support for Premiere Pro and where can I learn more about it?

UXP for Premiere Pro is now in public beta. Developers with existing CEP panels should review the announcement and communicate their migration requirements to the Premiere Pro team. More information is available at the Creative Cloud Developer Forums: https://forums.creativeclouddeveloper.com/t/uxp-now-available-in-premiere-pro-beta/8795

*Tags: `cep`, `deployment`, `migration`, `premiere`, `uxp`*

---

### How can a JavaScript plugin script open the default email client from ExecuteScript?

Use the mailto: URI scheme by opening a URL with the format 'mailto:email@address.com'. The browser will interpret the mailto: protocol and open the default mail application.

*Tags: `cross-platform`, `deployment`, `scripting`*

---

### Can you build After Effects plugins against older macOS SDKs using the latest Xcode?

James Whiffin reported that with the latest Xcode, there are compatibility issues when trying to build against older macOS SDKs (e.g., 10.10). This is a known limitation where newer Xcode versions may not support targeting legacy macOS versions.

*Tags: `build`, `cross-platform`, `deployment`, `macos`*

---

### How can I diagnose missing DLL dependencies in a Windows After Effects plugin?

Use Dependency Walker on Windows to check for missing DLLs and verify that all libraries and DLLs have the correct architecture (32-bit vs 64-bit) matching your plugin build. Also check system folders like System32 where driver-installed libraries like Vulkan may be located.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What should I verify when using external libraries like Vulkan in an After Effects plugin?

Ensure that external libraries are properly installed on the user's system (e.g., Vulkan drivers install libraries to System32), check for any transitive dependencies from licensing tools or other libraries, and verify that all linked DLLs match the architecture of your plugin build.

*Tags: `build`, `deployment`, `vulkan`, `windows`*

---

### How can you embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications?

When embedding MoltenVK in an AE plugin, the standard approach of placing it in the framework folder as a dynamic library with JSON in resources/vulkan doesn't work for plugin bundles. The scaleUp plugin demonstrates using xcframework to embed it as a static library inside the plugin binary, though this approach requires careful configuration as Vulkan instance creation may fail if not set up correctly. The key challenge is ensuring proper linking and resource resolution within the plugin bundle rather than relying on system-wide installation in /etc/vulkan.

*Tags: `deployment`, `macos`, `plugin`, `reference`, `vulkan`*

---

### What does After Effects error 25:3 mean and what causes it?

Error 25:3 in After Effects indicates that an effect cannot be applied because it cannot be initialized. This error is typically associated with a corrupted installation or missing dependencies. It has been observed to occur specifically when users are running After Effects over Windows Remote Desktop software, suggesting potential issues with plugin initialization in remote desktop environments.

*Tags: `debugging`, `deployment`, `plugin-error`, `windows`*

---

### What causes After Effects error 25:3 when using plugins over remote desktop software?

Error 25:3 ('This effect cannot be applied because it cannot be initialized') typically indicates a corrupted installation or missing dependency. However, when the error occurs specifically for users accessing After Effects through Windows Remote Desktop, it may be related to remote desktop-specific issues rather than actual corruption. The problem appears to be isolated to remote desktop environments rather than standard installations.

*Tags: `debugging`, `deployment`, `windows`*

---

### How do you load machine learning models in an After Effects plugin without shipping them with the tool?

Models can be pulled from Hugging Face on first use rather than being included with the plugin distribution. This approach reduces initial download size and keeps models up to date.

*Tags: `aegp`, `deployment`, `scripting`*

---

### What causes the 'Invalid Filter 25::3' error on macOS in After Effects?

The 'Invalid Filter 25::3' error on macOS can be caused by a mismatch between the Deployment Target set in Xcode and the macOS version running After Effects. If the Deployment Target is set higher than the OS version (e.g., setting Deployment Target to 12.3 when running on Big Sur 11.17.9), it will cause this error.

*Tags: `build`, `debugging`, `deployment`, `macos`*

---

### What is the minimum supported macOS version for After Effects plugin development?

High Sierra is a practical cutoff point for macOS support. Older versions like Sierra and earlier tend to have compatibility issues with many applications and libraries. Changing the macOS deployment target may not resolve underlying compatibility errors, and users may receive different error messages.

*Tags: `build`, `cross-platform`, `deployment`, `macos`*

---

### What causes 'entry point not found' errors on Windows when an After Effects plugin loads?

This error typically indicates a missing dependency, most commonly a missing Visual C++ redistributable package. Even if Visual Studio is installed on the development machine, the target machine may not have the required Visual C++ runtime libraries. On Windows 10, installing the Visual C++ 2022 redistributable (https://aka.ms/vs/17/release/vc_redist.x64.exe) often resolves the issue. Debugging can be done by setting a breakpoint on global setup in debug mode to verify if the plugin initialization is being reached.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What is the challenge of timing OS and development tool updates during plugin development?

Developers face a difficult choice between updating to new versions quickly (to catch bugs and unexpected behavior that only occur on the latest versions) and waiting to ensure stability and compatibility with their products. Rolling back an entire OS is impractical, so alternatives include waiting for the next patch of Xcode or maintaining separate machines for development and testing at different version levels.

*Tags: `build`, `debugging`, `deployment`, `macos`*

---

### Why might an AEGP not appear in After Effects and fail to hit any breakpoints during debugging?

This is a Mac-specific AEGP question about plugin visibility and debugging. The issue could be related to security settings or allocation changes on macOS, particularly around code signing, entitlements, or plugin sandbox restrictions. Possible causes include: incorrect plugin bundle structure, code signing issues, missing entitlements, or the plugin not being properly registered in After Effects' plugin directory.

*Tags: `aegp`, `debugging`, `deployment`, `macos`*

---

### Does Media Encoder load AEGP plugins?

No, Media Encoder does not load AEGP plugins. If your workflow relies on AEGP plugins for rendering, you cannot use Media Encoder as an alternative.

*Tags: `aegp`, `deployment`, `premiere`*

---

### Is Media Encoder compatible with AEGP plugins for rendering After Effects projects?

AEGP plugins are not supported by Adobe Media Encoder (AME). If your plugin requires AEGP functionality to render, you will need to convert the AEGP plugin into a standalone or command-line tool to use with Media Encoder.

*Tags: `aegp`, `deployment`, `premiere`, `tool`*

---

### What are alternatives to Nuitka for compiling Python code in After Effects plugins, especially for resolving dependency conflicts on macOS?

Cython is a viable alternative to Nuitka for compiling Python code in AE plugins. It can be used to compile specific portions of Python code and has fewer dependency conflicts compared to Nuitka, particularly when dealing with libraries like NumPy on macOS (which can have issues with libopenblas.0.dylib).

*Tags: `build`, `cross-platform`, `deployment`, `macos`, `scripting`*

---

### What is a tool for calculating PiPL outflags and outflags2 bitmasks?

Tobias Fleischer (reduxFX) created an online lookup/cheat sheet for calculating and evaluating bitmasks for the outflags and outflags2 fields in PiPL files. Available at https://reduxfx.com/aeoutflags.htm

*Tags: `deployment`, `pipl`, `reference`, `tool`*

---

### Are there tools available that can remove XMP packets from After Effects project files to reduce file size?

Yes, tools like AE Viewer are available that can remove the XMP packet from After Effects files, significantly reducing file sizes (e.g., from 50mb down to 50kb). This demonstrates that AE bloats project files with substantial XMP metadata that can be stripped if not needed.

*Tags: `deployment`, `reference`, `tool`*

---

### What is plugplug.DLL and how can it be used for inter-plugin communication?

plugplug.DLL is a DLL that allows After Effects plugins to communicate with each other. You can call the extern "C" functions exposed by plugplug.DLL from your C++ plugin to enable inter-plugin communication. This concept is related to loading plugins from within other plugins, as discussed in the Adobe community post: https://community.adobe.com/t5/after-effects-discussions/how-to-load-a-plugin-from-another-plugin/td-p/6738182

*Tags: `aegp`, `cross-platform`, `deployment`, `reference`, `windows`*

---

### Is it possible to include multiple PiPLs in a single plugin file, and what are the recommendations?

Yes, it is technically possible to include multiple PiPLs (both AEGPs and effects) in the same file, but it is not recommended. If you do use multiple PiPLs in the same file, AEGPs must come first. However, no other hosts (not even Premiere Pro) support multiple PiPLs pointing to multiple effects within the same .dll or code fragment. The recommendation from the SDK is to use one PiPL and one plugin per code fragment, especially to avoid shipping new builds of all plugins when updating just one.

*Tags: `aegp`, `deployment`, `pipl`, `premiere`*

---

### How can you determine the After Effects version at runtime in a plugin?

You can use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP as the path type to locate the running binary host, then extract the version from the AE folder name or binary metadata. Alternatively, you can try mapping AEGP_GetDriverSpecVersion to the AE version number, though it's unclear if AEGP version increments in parallel with the host version.

*Tags: `aegp`, `cross-platform`, `debugging`, `deployment`*

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

### What causes code signature breaks in After Effects plugins on macOS?

Code signature breaks can occur when the plugin installer doesn't properly clean up old files before installing new versions. If v1 has a file that is removed in v2, an installer that overwrites rather than fully uninstalls will leave the old file in place, breaking the code signature. Modifying any file in a signed package (even adding a dummy.txt) will break the signature. The solution is to ensure complete removal of all old files before installing the new plugin version, not just overwriting existing files.

*Tags: `build`, `deployment`, `macos`*

---

### What is the relationship between After Effects build tools (Xcode) and plugin code signing issues?

After Effects being built with the latest Xcode version has introduced stricter code signature checks that were not enforced in previous builds. Apple's new Xcode performs code tampering detection differently, causing signature validation failures for plugins that previously loaded without issue. This is a macOS-specific binary signing problem, separate from CEP or ZXP signing issues.

*Tags: `build`, `deployment`, `macos`*

---

### Are there known issues with ZXPSignCmd and plugin signing on Windows?

Yes, there have been reports of ZXPSignCmd segmentation faults and timestamp-related signing failures on Windows. These issues appear to be affecting plugin developers globally, and Adobe was working on releasing a patched version of the ZXPSignCmd tool to resolve them.

*Tags: `deployment`, `tool`, `windows`*

---

### Can multiple effects be registered within a single DLL for After Effects plugins?

Yes, according to Alex Bizeau from maxon, it is possible to call multiple register effect functions in one DLL. This approach could be used to create a bootstrapper similar to what was done for OFX, allowing a single DLL to load multiple effects and reduce code duplication across OFX, After Effects, and AVX plugins.

*Tags: `aegp`, `build`, `deployment`, `pipl`, `plugin-architecture`*

---

### When was the URL-based plugin registration method added to After Effects?

The URL-based plugin registration version was added two versions ago in After Effects 23, according to tlafo. This allows plugins to be registered without needing traditional PIPL resources.

*Tags: `aegp`, `deployment`, `pipl`, `reference`*

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

### How can you create a render-only version of an After Effects plugin that doesn't require a GUI license on render farms?

One approach is to make all parameters invisible while still allowing the plugin to read them, effectively creating a read-only version of the plugin. This way, the plugin can process parameters set during the authoring phase without requiring interactive UI on render farm machines.

*Tags: `deployment`, `params`, `render-loop`, `ui`*

---

### Why does a plugin work in the Plugins folder but not the Mediacore folder on Mac After Effects 26.0.0?

Multiple developers reported this issue with After Effects 26.0.0 on Mac where plugins load correctly from the Plugins folder but fail to load from the Mediacore folder. The Gaussian Splat team issued an update to address AE 26 compatibility, suggesting there are specific compatibility changes in AE 26 affecting Mediacore plugin loading on Mac. The issue also affects Premiere Pro 2026, where plugins in Mediacore don't appear at all, even though they work in earlier Premiere versions (2023-2025) and in AE 2026.

*Tags: `cross-platform`, `debugging`, `deployment`, `macos`, `mfr`*

---

### What is the most logical way to prevent an After Effects plugin from being detected in Premiere Pro?

One approach is to return an error in the global setup function, since Premiere Pro runs global setup for all plugins during application startup. This may cause the plugin to be hidden from the effects list, though this behavior was noted as not always working reliably.

*Tags: `aegp`, `deployment`, `pipl`, `premiere`*

---

### How can plugin developers avoid requiring customers to reinstall plugins when After Effects versions are updated?

Use the PF_OutFlag_I_AM_OBSOLETE flag when setting plugin output flags. This flag, when added to the plugin configuration, signals to After Effects that the plugin can work across version updates without requiring reinstallation. Plugin developers should suggest customers install plugins into the mediacore directory to leverage this compatibility feature and avoid the yearly reinstall cycle that occurs with AE version upgrades.

*Tags: `cross-platform`, `deployment`, `pipl`*

---

### How can you prevent the 'need to reinstall plugin' message when After Effects updates without hiding the plugin from the UI?

James Whiffin suggests installing plugins into mediacore to avoid yearly reinstall requirements when AE versions up. However, Tobias Fleischer notes that using PF_OutFlag_I_AM_OBSOLETE will hide the plugin in AE itself. To use this flag only for Premiere Pro while keeping it visible in After Effects, you should add a host name check before applying the flag.

*Tags: `cross-platform`, `deployment`, `pipl`, `premiere`*

---

### What happened to Transcriptive and how did Premiere's native transcription feature impact third-party plugins?

Transcriptive by Digital Anarchy was a popular plugin for years but Adobe introduced native transcription features in Premiere, making third-party solutions less necessary. Transcriptive's web services are ending in May 2026. Reference: https://digitalanarchy.com/blog/video-editing-plugins/transcriptive-end-of-life-web-services-will-be-ending-in-may-2026/

*Tags: `deployment`, `premiere`, `reference`*

---

### How did Adobe's variable font support in After Effects affect third-party VariFont plugins?

Tobias Fleischer (reduxFX) developed the VariFont plugin as the only way to use variable fonts in After Effects for an extended period. After years of waiting, Adobe finally implemented native variable font support in AE, making the third-party plugin less essential. Plugin developers anticipated this would eventually happen.

*Tags: `ae`, `deployment`, `reference`*

---

### Why are plugins not loading in After Effects 2026?

There is a known issue on Mac where plugins located in the Mediacore folder are not recognized by AE2026 and must be placed in After Effects' main plugin folder instead. This appears to be one of several compatibility issues affecting plugin loading in AE2026.

*Tags: `debugging`, `deployment`, `macos`, `pipl`*

---

### Where should After Effects plugin developers share and host code snippets?

Stack Overflow is recommended as a good platform for sharing code snippets and knowledge about After Effects SDK development. Using the tag 'After Effects SDK' makes the content searchable and discoverable. While GitHub could be used, Stack Overflow's forum structure is better suited for this niche community compared to chat systems which have limitations for code hosting.

*Tags: `deployment`, `open-source`, `reference`, `scripting`, `tool`*

---

### What are the advantages of using Lua over Python for developing After Effects plugins?

Lua is significantly better suited for plugin development compared to Python. The main advantage is that Lua is much easier to compile in and integrate as a library. Python integration was problematic because it required asking permission to install libraries all over the user's system, which is intrusive. Lua avoids this dependency management issue entirely. Both languages require similarly tedious C bindings/code generation, but Lua's compilation and integration model is cleaner for plugin distribution.

*Tags: `build`, `deployment`, `plugin`, `scripting`*

---

### Is there an open-source After Effects plugin example available?

dvb metareal shared their After Effects plugin at https://omino.com/pixelblog, currently at version 0.9. This serves as a reference implementation for AE plugin development.

*Tags: `deployment`, `open-source`, `reference`*

---

### Is it possible to install and activate Adobe After Effects CS6 on Windows 11?

CS6 should work on Windows 11, but you may need to run it in compatibility mode. If Windows 11 causes issues, consider testing on Windows 10 instead. Activation may be problematic due to Adobe's legacy license servers, so compatibility mode is a recommended workaround.

*Tags: `compatibility`, `cross-platform`, `deployment`, `windows`*

---

### What changes are coming to the Premiere SDK regarding capture and recording tools?

According to the SDK roadmap, Premiere will be removing tools for capture recording functionality in the future, signaling the deprecation of tape-based workflows.

*Tags: `deployment`, `premiere`, `sdk`*

---

### How long will the CEP engine remain available in Premiere Pro before it's deprecated?

The conversation mentions that UXP for Premiere Pro is now in public beta and encourages CEP panel developers to migrate, but no specific timeline for CEP deprecation was provided in the response.

*Tags: `cep`, `cross-platform`, `deployment`, `premiere`, `uxp`*

---

### What is the current status of UXP support for Premiere Pro?

UXP for Premiere Pro is now available in public beta. Developers with existing CEP panels are encouraged to migrate to UXP and provide feedback to the Premiere Pro team about their migration needs. More information is available at: https://forums.creativeclouddeveloper.com/t/uxp-now-available-in-premiere-pro-beta/8795

*Tags: `deployment`, `premiere`, `resource`, `ui`, `uxp`*

---

### How do you debug a plugin when it's loaded through After Effects Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible After Effects instance for rendering, separate from the main AE process. Breakpoints set in the main application won't be triggered during AME rendering. To debug in this scenario, you need to attach your debugger to the already-running subprocess. On Windows, use the Visual Studio debugger's attach-to-process feature: https://learn.microsoft.com/en-us/visualstudio/debugger/attach-to-running-processes-with-the-visual-studio-debugger?view=vs-2022. You can also observe these processes in Windows Task Manager.

*Tags: `debugging`, `deployment`, `media-encoder`, `windows`*

---

### How can I make an AEGP plugin invisible and prevent it from appearing in the After Effects Window toolbar?

To make an AEGP plugin invisible and remove it from the Window toolbar, do not use AEGP_InsertMenuCommand or AEGP_RegisterCommandHook. These functions are only needed to create menu entries. By omitting these calls, the plugin will work in the background without appearing in the UI.

*Tags: `aegp`, `deployment`, `ui`*

---

### What is the purpose of the PF_Cmd_SEQUENCE_FLATTEN selector in After Effects plugins?

PF_Cmd_SEQUENCE_FLATTEN is sent by After Effects when saving or duplicating sequences. It requires the plugin to flatten sequence data containing pointers or handles so the data can be correctly written to disk and saved with the project file. The flattened data must be properly byte-ordered for file storage. When this command is received, the plugin should free unflat data and set out_data->sequence_data to point to the new flattened data. As of SDK 6.0, if an effect's sequence data has been flattened, the effect may be deleted without receiving PF_Cmd_SEQUENCE_SETDOWN, and After Effects will dispose of the flat sequence data.

*Tags: `arb-data`, `deployment`, `sdk`, `sequence-data`*

---

### Why is my After Effects plugin not showing up in the Effects menu after building it?

The plugin's build output location must be in a directory where After Effects looks for plugins. AE loads plugins from the Plug-ins folder in its installation directory, searching up to 5 folders deep. Either build directly into a subfolder of AE's Plug-ins directory, or create a shortcut in that directory pointing to your build output. Using a shortcut is recommended as it allows testing across multiple AE versions without updating build preferences when installing new AE versions.

*Tags: `build`, `debugging`, `deployment`, `macos`, `windows`*

---

### Why does my After Effects plugin fail to load with Error 126 when launched directly, but works when loaded from Visual Studio?

Error 126 when loading an AE plugin typically indicates a dependency issue with missing DLLs. When running the plugin from Visual Studio's debugger, the debugger's environment may provide DLL paths that aren't available when After Effects is launched directly. To diagnose the issue, build a simple test plugin that loads correctly in both contexts and have it report its home directory path. If testing on the same machine, the problem is likely a difference in the process's working directory or DLL search paths between debugger and direct execution. Ensure all dependent DLLs (like those required by the cairo library) are either in the plugin directory, in a standard system path, or that your plugin's search paths are properly configured for non-debugger execution.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### How can I restrict an After Effects plugin to specific app versions while keeping it in a single installation folder?

Installing a plugin in the Adobe/Common/Plug-ins/7.0/MediaCore folder makes it available in all versions of After Effects and Premiere Pro. Unfortunately, there is no way to restrict it to specific app versions or applications from the MediaCore folder. To restrict a plugin to only specific versions (e.g., After Effects CC 2014-2019), you must install it in each application version's individual plug-ins folder instead of using the shared MediaCore location.

*Tags: `after effects`, `deployment`, `installation`, `plugin`, `premiere`*

---

### Can I disable an After Effects plugin when it loads in Premiere Pro by checking the application ID?

Previously, it was possible to disable a plugin in Premiere Pro by checking in_data->appl_id for 'PrMr' and returning an error code (-1 or similar) at the beginning of the entrypoint function. However, this method no longer works with Premiere Pro CC2018 and later versions. The recommended approach is to install the plugin in version-specific folders rather than relying on application ID checking.

*Tags: `aegp`, `after effects`, `deployment`, `plugin`, `premiere`*

---

### How do I embed an image resource in a Windows .aex plugin file?

Add your image file as an image resource or generic binary resource from the Visual Studio File menu. You will get a custom identifier for this resource. Then use standard Windows resource functions to load it. Example code: HRSRC myResource = ::FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA); HGLOBAL myResourceData = ::LoadResource(NULL, myResource); void* pMyBinaryData = ::LockResource(myResourceData);

```cpp
HRSRC myResource = ::FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA);
HGLOBAL myResourceData = ::LoadResource(NULL, myResource);
void* pMyBinaryData = ::LockResource(myResourceData);
```

*Tags: `build`, `deployment`, `params`, `ui`, `windows`*

---

### Can After Effects plugins be compiled in Release mode, and what are the runtime linking requirements?

Yes, After Effects plugins should be compiled in Release mode for both performance and deployment ease. There are four C++ runtime linking options: MT (multithreaded static), MD (multithreaded DLL), MTd (multithreaded debug static), and MDd (multithreaded debug DLL). Adobe recommends using MD or MDd because there is a limit to how many MT-compiled plugins After Effects can load simultaneously. In Release mode, use either MD or MT; in Debug mode, use either MDd or MTd. There is no noticeable performance difference between MD and MT variants.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What is the function of the PiPL.r file in After Effects plugin development?

The PiPL.r (Plug In Property List) file was originally created to allow After Effects to get information about plug-ins without loading them, which was useful when computers had limited RAM (128MB). Nowadays it's mostly a legacy requirement. The values in the PiPL must correlate to the values set in the plugin's global setup call, or an error message will be sent during the plugin's launch indicating a mismatch. To set the correct values: put a breakpoint in the global setup function to see the numerical values applied to outflags and outflags2 variables, then copy these values to the PiPL file. Note that a clean rebuild is required for changes to take effect, as the pipl_tool only generates a new .rc file when one doesn't already exist.

*Tags: `build`, `deployment`, `pipl`, `plugin-properties`*

---

### What is the difference between .plugin and .aex file formats for After Effects plugins?

On Windows, After Effects plugins are built as .aex files. On macOS, plugins are built as .plugin files, which are actually bundles (not single files). Both formats are correct for their respective platforms and will work when built correctly according to the SDK instructions.

*Tags: `build`, `deployment`, `macos`, `windows`*

---

### Why do project files crash after adding new parameters to an effect plugin?

Project files crash when parameters are added because After Effects uses DISK_ID values to associate saved parameter data with plugin parameters. If you change the order of parameter definitions in your enum or reassign DISK_IDs, After Effects cannot correctly map the old saved data to the new parameters. For example, if a slider parameter with DISK_ID=3 is replaced by a checkbox parameter in the new version, the saved slider data will be incorrectly applied to the checkbox, causing crashes. The solution is to ensure that existing parameters retain their exact same DISK_ID values from the old version, and only assign new DISK_IDs to newly added parameters. The DISK_ID order has nothing to do with the visual parameter order in the UI.

*Tags: `aegp`, `arb-data`, `deployment`, `params`*

---

### How should you handle sequence data structure changes when adding new parameters?

When modifying sequence data structure, if you extend or modify the structure, old project files loaded with the new plugin will attempt to match the old sequence data to the new structure, potentially causing conflicts. Best practice is to only add new parameters with new IDs at the end of the current parameter set without changing the order of existing parameters. Do not replace existing parameter definitions, only append new ones. This ensures backward compatibility with projects saved using older plugin versions.

*Tags: `arb-data`, `deployment`, `params`, `sequence-data`*

---

### What steps are necessary to convert an After Effects plugin from Windows to Mac?

If your C++ plugin code doesn't use Windows-specific native APIs or special library calls, porting to Mac should be straightforward—essentially just recompiling after adding all relevant project files. You'll need to use Xcode as your development environment on Mac instead of Visual Studio. Start from the Mac sample project (just as you did with the Windows sample project), but the plugin C/C++ code itself should require minimal to no changes if you've adhered to cross-platform APIs and the After Effects SDK.

*Tags: `build`, `cross-platform`, `deployment`, `macos`, `windows`*

---

### What language and tools should I use to develop an After Effects plugin for controlling layer properties and exporting data?

For your use case of creating a GUI, controlling layers and layer properties (content, effects, transform), and exporting data as XML, you don't need to develop a C plugin. Instead, use JavaScript/ExtendScript to create a script that can be encrypted as a .jsxbin file. Refer to the After Effects Scripting Guide PDF for detailed information on manipulating layers, effects, and properties. The AE Scripting forum at https://forums.adobe.com/community/aftereffects_general_discussion/ae_scripting is also a valuable resource for scripting questions.

*Tags: `deployment`, `reference`, `scripting`, `ui`*

---

### What file format does a compiled After Effects plug-in produce?

When you compile an After Effects plug-in using an SDK sample project as your base in Visual Studio or Xcode, it produces a .aex file. This file can be placed in After Effects' plug-ins folder and will appear in the effects menu when After Effects is launched.

*Tags: `build`, `deployment`*

---

### Can I set a higher priority for my custom AEIO to override Adobe's built-in handler for a format like FLV?

No, the After Effects SDK does not support overriding the priority of built-in AEIOs. After Effects will ignore custom AEIOs that associate to formats it already handles natively. The recommended workaround is to ask users to change the file extension to a different one (e.g., .flv to .custom_flv), and optionally create an import menu entry to automate this extension change for better user experience.

*Tags: `aegp`, `aeio`, `deployment`, `sdk`*

---

### What causes an 'invalid filter' error when loading a compiled After Effects plugin?

The invalid filter error is usually a dependency issue. Check your plug-in using Dependency Walker to verify all required components are present on your machine. This error also commonly occurs when you compile your plug-in in Debug mode and try to run it on a non-development machine where the debug DLLs don't exist. Compiling in Release mode resolves this issue.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What is the recommended way to store large numbers of parameters that need to be saved in a project file?

For storing many parameters (e.g., 1000+ values) that need to be persisted in the project file, use sequence data or arbitrary data rather than individual built-in parameter types. These are automatically saved and recalled with the project. Ensure you correctly handle copying and flattening/deflattening of values in the appropriate functions, especially if using pointers. The SDK examples demonstrate the correct implementation.

*Tags: `arb-data`, `deployment`, `params`, `sequence-data`*

---

### Is it possible to create a custom output device in After Effects beyond the built-in options?

Yes, it is possible to create a custom output device in After Effects. The recommended approach is to base your plugin on the EMP (External Monitor Preview) sample plugin that is included in the SDK. This sample demonstrates how to implement custom output device functionality.

*Tags: `aegp`, `deployment`, `output-rect`, `reference`*

---

### How should parameter stream IDs be managed when modifying plugin parameter order across versions?

Maintain separate enumerations: one for UI order (parameter definition order) and one for disk IDs (stream identifiers). When adding or reordering parameters in a new plugin version, keep the disk ID enumeration unchanged from previous versions. This allows After Effects to correctly map saved project data to parameters even if their UI positions change. For example, if a slider had disk_id=7 in v1, keep it at 7 in v2 even if it moves to a different position in the UI enumeration.

```cpp
// UI enumeration (definition order)
enum {
    BASE = 0,
    SLIDER_1,
    SLIDER_2,
    POINT,
    NUM_PARAMS
};

// Disk ID enumeration (never change these values)
enum {
    SLIDER_1_ID = 1,
    SLIDER_2_ID = 2,
    POINT_ID = 3
};
```

*Tags: `aegp`, `deployment`, `params`*

---

### Why does the After Effects SDK build succeed but produce no .dll or .aex file?

By default in the CS4 SDK, plugins build to c:\program files\adobe\after effect 9.0\plug-ins\sdk (note "after effect 9.0" instead of "CS4"). If you don't see output in your expected directory, check the Visual Studio Linker options to verify the output directory is set correctly, or search your computer for the .aex file as it may be building to an unexpected location.

*Tags: `after_effects_sdk`, `build`, `deployment`, `sdk`, `visual_studio`*

---

### How do I resolve error 126 when users try to load my After Effects plugin?

Error 126 typically indicates a dependency issue. Use Dependency Walker to analyze your plugin and identify missing or problematic DLL dependencies. If your plugin requires MSVCR100D.DLL (debug runtime), change your C/C++ code generation runtime library setting: use /MT to statically link the runtime library into your plugin (no external DLLs needed, larger file size), or /MD to use the release version of MSVCR100 (requires distributing the DLL). Test your plugin on a non-developer machine using Dependency Walker to ensure no problematic dependencies remain. Avoid requiring debug runtime libraries in production builds.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---

### What tool can I use to check for dependency issues in my After Effects plugin?

Dependency Walker is a useful tool for analyzing plugin dependencies on Windows. It can identify external DLL requirements and help diagnose loading errors. Run it on both 32-bit and 64-bit versions of your plugin to ensure compatibility across architectures.

*Tags: `debugging`, `deployment`, `tool`, `windows`*

---

### How do you compile the Panelator sample to a release version that runs on non-developer machines?

When compiling Panelator for release, change the runtime library from /MD to /MT to make the plugin independent of external MSVC runtime DLLs. Additionally, ensure all supporting code files in the 'supporting code' folder have consistent compiler settings (not mixed MD/MDd). If using Visual Studio 2008 instead of the SDK-intended VS2005, be aware that 2008 uses MSVCR90.DLL instead of MSVCR80.DLL, which can cause compatibility issues. Using /MT will statically link all libraries into the plugin, making it dependent only on KERNEL32.DLL. Use Dependency Walker to verify the plugin's dependencies before deployment.

*Tags: `build`, `debugging`, `deployment`, `windows`*

---
