# Emulation

> 1 Q&A · source: AE plugin dev community Discord

### Does the AE Windows ARM beta require native ARM64 plugins or can x64 plugins run in emulation?

Adobe's AE for ARM is not EC (Emulation Compatible), meaning x64 plugins will NOT run emulated on ARM, unlike Resolve and Nuke which currently work in EC mode. You must rebuild your Windows plugins as native ARM64. The latest VS2022 is required for ARM64 compilation, and all third-party libraries (static or dynamic) must also be available in ARM64 format. The C++ licensing library from aescripts is not yet available for Windows ARM64 but internal builds are working.

*Tags: `arm64`, `emulation`, `native`, `visual-studio-2022`, `windows-arm`*

---
