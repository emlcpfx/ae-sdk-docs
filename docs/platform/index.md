# Platform & Build

Building, debugging, and deploying AE plugins on Windows and macOS.

## Build Systems

- **CMake** — Recommended for cross-platform. See the community CMake templates.
- **Visual Studio** — Windows native. SDK includes VS project templates.
- **Xcode** — macOS native. SDK includes Xcode project templates.

## Windows Essentials

- Use `/MT` (static CRT) to avoid runtime DLL dependencies
- PiPL compilation requires the SDK's `PiPLtool.exe`
- Debug with **Visual Studio** attached to `AfterFX.exe`
- Use **DebugView** (Sysinternals) to see `OutputDebugString` messages

## macOS Essentials

- Build **Universal Binaries** (x86_64 + arm64) for Apple Silicon support
- **Code signing** and **notarization** required for distribution
- Debug with **Xcode** or **lldb** attached to After Effects

## Debugging Tips

1. Attach debugger to the running AE process
2. Set breakpoints in your `PF_Cmd_SMART_RENDER` handler
3. Use `OutputDebugString` (Win) or `NSLog` (Mac) for printf-style debugging
4. **DebugView** is essential on Windows — filter by your plugin name

---

*54 documents in this section. Use **Search** (top of page) to find what you need.*
