# Q&A: xcode

**17 entries** tagged with `xcode`.

---

## How do you build AE plugins for Apple Silicon (ARM64)?

You can build on an M1 Mac (even a cheap Mac Mini works as a build machine). It's not strictly required to build on M1, but it helps for testing. The key thing many people miss is that you must add ARM64 to your PiPL resource file - just building as Universal Binary in Xcode is not enough. Without the ARM64 entry in the PiPL, AE will show the 'not yet compatible' warning.

*Contributors: [**gabgren**](../contributors/gabgren/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/), [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2023-08-18 · Tags: `apple-silicon`, `arm64`, `pipl`, `xcode`, `mac`, `universal-binary`*

---

## Are there CMake-based build systems for AE plugins?

Yes, there are several examples on GitHub: Vulkanator (https://github.com/Wunkolo/Vulkanator) uses CMake, and after_effects_cmake (https://github.com/mobile-bungalow/after_effects_cmake) is based on Vulkanator's setup. There's also a Rust-based project (https://github.com/virtualritz/after-effects) with its own PiPL compiler written in Rust. On Mac, you can use the Rez tool for .rsrc files and potentially use Ninja for faster incremental compilation.

*Contributors: [**tlafo**](../contributors/tlafo/), [**fad**](../contributors/fad/), [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2023-07-06 · Tags: `cmake`, `build-system`, `xcode`, `ninja`, `cross-platform`*

---

## What causes the 'Invalid Filter 25::3' error on macOS?

This error is typically caused by the Deployment Target being set too high in Xcode. For example, setting it to 12.3 will cause this error on older macOS versions. Try lowering it to support older versions. For Intel builds, supporting back to macOS 10.10 is common.

*Contributors: [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2023-08-16 · Tags: `mac`, `xcode`, `deployment-target`, `invalid-filter`, `compatibility`*

---

## How do you run an older version of Xcode on a newer macOS?

Download older Xcode versions from https://developer.apple.com/download/all/. Run the binary directly from Terminal: '/Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode'. The .app bundle won't open normally on newer macOS, but running the binary directly bypasses this restriction. For quick access, add the /Contents/MacOS folder to Finder sidebar.

*Contributors: [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2023-11-28 · Tags: `xcode`, `mac`, `debugging`, `development-environment`*

---

## What causes slow AE loading when debugging from Xcode 15?

After upgrading to macOS 14/Xcode 15, AE loading becomes ~3x slower due to 'dlsym cannot find symbol xSDKExport' messages for MediaCore bundles. This is Xcode trying to load symbols for all loaded bundles. Workarounds: (1) Run Xcode 14 binary directly from Terminal. (2) Open AE standalone first, then attach the debugger after AE is fully loaded. (3) Check if 'load symbols' option is accidentally selected in your IDE settings.

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**tlafo**](../contributors/tlafo/), [**gabgren**](../contributors/gabgren/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2023-11-29 · Tags: `xcode`, `debugging`, `performance`, `macos`, `symbol-loading`*

---

## How do I build an After Effects plugin for Apple Silicon (M1/ARM64)?

You need to do two things: (1) Set the Xcode build architecture to include arm64 (ensure the architecture setting targets ARM), and (2) Add 'CodeMacARM64 {"EffectMain"}' to your PiPL resource file. Without the PiPL entry, AE won't recognize the plugin as ARM-native even if the binary is compiled for arm64.

*Contributors: [**Nate**](../contributors/nate/) · Source: adobe-plugin-devs · 2022-04-28 · Tags: `apple-silicon`, `m1`, `arm64`, `pipl`, `xcode`, `mac`, `build-configuration`*

---

## How to compile unicode characters (e.g., Chinese) in C++ plugin parameter names and category names for Mac AE plugins?

The name field is defined as A_char[32]. Convert your string to UTF-8 and feed the param macro the resulting string. Make sure the length of the resulting string is no more than 31 chars long (including multibyte characters), as the last char must be used for null termination.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-06-01 · Tags: `unicode`, `utf8`, `mac`, `xcode`, `parameter-name`, `localization`*

---

## Is it possible to hot-reload AE plugins during development without restarting AE?

Hot reload (called 'Edit and Continue' in Visual Studio) was common on 32-bit builds but became less reliable on 64-bit. Even with hot reloading, there are limitations: you can't change ParamSetup because AE calls it only once per session when the plugin is first loaded. Starting AE via the debugger (Visual Studio or Xcode) tends to launch faster than normal startup. Each time you rebuild C++ code, you generally need to restart AE, unlike ExtendScript or CEP where reloads are possible without full restart.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/), [**Arie Stavchansky**](../contributors/arie-stavchansky/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `hot-reload`, `development-workflow`, `debugging`, `visual-studio`, `xcode`*

---

## How do you debug a Premiere plugin from Xcode and load debug symbols?

Ensure that you have 'Generate Symbols' set to 'Yes' in your Xcode scheme settings. If symbols are set to 'No', breakpoints will not work and Xcode will not be able to load the necessary debugging information for your Premiere plugin.

*Tags: `debugging`, `premiere`, `xcode`, `build`*

---

## Has anyone ported the After Effects plugin build system to use CMake instead of Visual Studio and Xcode?

The question was asked but no one in the conversation provided a definitive answer or shared an existing CMake-based build system for AE plugins. This remains an open challenge in the community.

*Tags: `cmake`, `build`, `cross-platform`, `xcode`, `windows`*

---

## How can you work around SDK issues when using Xcode 14 with After Effects plugin development?

Run the Xcode binary directly from Terminal instead of launching Xcode.app. The command is '/Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode' (replace with your Xcode version). You can download specific Xcode versions from https://developer.apple.com/download/all/. For quick access, add the /Contents/MacOS folder to your Finder sidebar, then right-click the binary, hold Alt/Option, click 'Copy as Pathname', paste into Terminal and execute. This takes 3-4 seconds versus the normal Spotlight launch method.

*Tags: `macos`, `debugging`, `build`, `xcode`*

---

## Why does adding another PiPL resource with a different name and matchname only show one plugin in After Effects?

When multiple PiPL resources are added to the same Xcode project, After Effects may only recognize one of them. This is typically due to incorrect PiPL configuration, resource ID conflicts, or the build process not properly including all PiPL resources in the final plugin binary. Ensure each PiPL has a unique resource ID, verify the matchname and name fields are correctly differentiated, and check that all PiPL resources are included in the target's Copy Bundle Resources build phase.

*Tags: `pipl`, `build`, `xcode`, `plugin-structure`*

---

## How do you debug a Premiere plugin from Xcode when symbols aren't loading and breakpoints don't work?

Ensure that symbol generation is enabled in your build settings. Set 'Generate Symbols' to 'Yes' in your Xcode scheme. If this setting is accidentally set to 'No', symbols won't be generated and debuggers won't be able to load them, preventing breakpoints from working properly.

*Tags: `debugging`, `premiere`, `xcode`, `build`*

---

## How do you handle Unicode characters in After Effects plugin parameter names and categories on macOS?

The name field is defined as 'A_char[32]', so you need to convert your string to UTF-8 and feed it to the param macro. Make sure the length of the resulting UTF-8 string is no more than 31 characters long (including multibyte characters), as the last character must be reserved for null termination. One successful approach mentioned was using ConvertUTF8ToGBK() in a UTF-8 encoded .cpp file when working with XCode.

*Tags: `macos`, `params`, `unicode`, `build`, `xcode`*

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
