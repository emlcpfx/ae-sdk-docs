# Arm64

> 4 Q&As · source: AE plugin dev community Discord

### How do you build AE plugins for Apple Silicon (ARM64)?

You can build on an M1 Mac (even a cheap Mac Mini works as a build machine). It's not strictly required to build on M1, but it helps for testing. The key thing many people miss is that you must add ARM64 to your PiPL resource file - just building as Universal Binary in Xcode is not enough. Without the ARM64 entry in the PiPL, AE will show the 'not yet compatible' warning.

*Tags: `apple-silicon`, `arm64`, `mac`, `pipl`, `universal-binary`, `xcode`*

---

### How do I build an After Effects plugin for Apple Silicon (M1/ARM64)?

You need to do two things: (1) Set the Xcode build architecture to include arm64 (ensure the architecture setting targets ARM), and (2) Add 'CodeMacARM64 {"EffectMain"}' to your PiPL resource file. Without the PiPL entry, AE won't recognize the plugin as ARM-native even if the binary is compiled for arm64.

*Tags: `apple-silicon`, `arm64`, `build-configuration`, `m1`, `mac`, `pipl`, `xcode`*

---

### Is there a way to create a universal .aex binary for both x64 and Windows ARM, similar to macOS universal binaries?

No, there is no universal binary format for Windows like macOS has. You need to compile separate binaries for each architecture. However, you can cross-compile for both x64 and ARM64 platforms on one system using the latest Visual Studio 2022 (after installing the respective tool sets). Note that AE on ARM is not EC (Emulation Compatible), so unlike Resolve and Nuke which run x64 code emulated on ARM, AE requires native ARM64 plugins. All third-party libraries you link to must also be available in ARM64 format.

*Tags: `architecture`, `arm64`, `cross-compile`, `visual-studio`, `windows-arm`*

---

### Does the AE Windows ARM beta require native ARM64 plugins or can x64 plugins run in emulation?

Adobe's AE for ARM is not EC (Emulation Compatible), meaning x64 plugins will NOT run emulated on ARM, unlike Resolve and Nuke which currently work in EC mode. You must rebuild your Windows plugins as native ARM64. The latest VS2022 is required for ARM64 compilation, and all third-party libraries (static or dynamic) must also be available in ARM64 format. The C++ licensing library from aescripts is not yet available for Windows ARM64 but internal builds are working.

*Tags: `arm64`, `emulation`, `native`, `visual-studio-2022`, `windows-arm`*

---
