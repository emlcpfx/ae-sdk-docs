# Windows Arm

> 2 Q&As · source: AE plugin dev community Discord

### Is there a way to create a universal .aex binary for both x64 and Windows ARM, similar to macOS universal binaries?

No, there is no universal binary format for Windows like macOS has. You need to compile separate binaries for each architecture. However, you can cross-compile for both x64 and ARM64 platforms on one system using the latest Visual Studio 2022 (after installing the respective tool sets). Note that AE on ARM is not EC (Emulation Compatible), so unlike Resolve and Nuke which run x64 code emulated on ARM, AE requires native ARM64 plugins. All third-party libraries you link to must also be available in ARM64 format.

*Tags: `architecture`, `arm64`, `cross-compile`, `visual-studio`, `windows-arm`*

---

### Does the AE Windows ARM beta require native ARM64 plugins or can x64 plugins run in emulation?

Adobe's AE for ARM is not EC (Emulation Compatible), meaning x64 plugins will NOT run emulated on ARM, unlike Resolve and Nuke which currently work in EC mode. You must rebuild your Windows plugins as native ARM64. The latest VS2022 is required for ARM64 compilation, and all third-party libraries (static or dynamic) must also be available in ARM64 format. The C++ licensing library from aescripts is not yet available for Windows ARM64 but internal builds are working.

*Tags: `arm64`, `emulation`, `native`, `visual-studio-2022`, `windows-arm`*

---
