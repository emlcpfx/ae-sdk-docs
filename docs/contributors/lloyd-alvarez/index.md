# Lloyd Alvarez

**5 contributions** to AE SDK community knowledge.

Top topics: `windows-arm`, `arm64`, `rust`, `sdk`, `cross-compile`, `visual-studio`, `architecture`, `wrapper`, `gui-api`, `cpp`

---

## Does the AE Windows ARM beta require native ARM64 plugins or can x64 plugins run in emulation?

Adobe's AE for ARM is not EC (Emulation Compatible), meaning x64 plugins will NOT run emulated on ARM, unlike Resolve and Nuke which currently work in EC mode. You must rebuild your Windows plugins as native ARM64. The latest VS2022 is required for ARM64 compilation, and all third-party libraries (static or dynamic) must also be available in ARM64 format. The C++ licensing library from aescripts is not yet available for Windows ARM64 but internal builds are working.

*Source: aescripts discord · 2026-01-29 · Tags: `windows-arm`, `arm64`, `emulation`, `native`, `visual-studio-2022` · [View in Q&A](../qa/windows-arm/)*

---

## Is there a way to create a universal .aex binary for both x64 and Windows ARM, similar to macOS universal binaries?

No, there is no universal binary format for Windows like macOS has. You need to compile separate binaries for each architecture. However, you can cross-compile for both x64 and ARM64 platforms on one system using the latest Visual Studio 2022 (after installing the respective tool sets). Note that AE on ARM is not EC (Emulation Compatible), so unlike Resolve and Nuke which run x64 code emulated on ARM, AE requires native ARM64 plugins. All third-party libraries you link to must also be available in ARM64 format.

*Source: aescripts discord · 2025-09-25 · Tags: `windows-arm`, `arm64`, `cross-compile`, `visual-studio`, `architecture` · [View in Q&A](../qa/windows-arm/)*

---

## Is there a Rust alternative for AE/Premiere plugin development?

Yes, there is a Rust bindings project at https://github.com/virtualritz/after-effects/ with documentation at https://docs.rs/after-effects/ and https://docs.rs/premiere/. It was originally created by Moritz Moeller and significantly refactored by Adrian Eddy (author of Gyroflow). The Gyroflow AE/Premiere plugin uses these bindings. The examples show significantly less boilerplate compared to C/C++ versions, are mostly 100% safe Rust, and include a PiPL helper crate. The crates also build on Linux for development purposes (though you need Windows/macOS to run the plugin).

*Source: aescripts discord · 2024-10-02 · Tags: `rust`, `bindings`, `gyroflow`, `alternative-language`, `pipl` · [View in Q&A](../qa/rust/)*

---

## Are developers allowed to redistribute the Adobe SDKs publicly?

Most likely no. You should check with Adobe directly, but the general expectation is that redistribution is not permitted.

*Source: aescripts discord · 2024-10-02 · Tags: `sdk`, `licensing`, `redistribution`, `adobe` · [View in Q&A](../qa/sdk/)*

---

## Are there any good C++ wrapper libraries for the After Effects Filter GUI API?

No well-known general-purpose wrapper exists. Most wrappers that have been built end up being very complex themselves if you try to handle all use cases -- you end up with a non-standard system that is almost as complex as the original, just with more C++ features. Copy-pasting C chunks from the SDK examples works well in practice. That said, even a C wrapper with automatic memory management and state stored in structs would be an improvement over the current state machine. There is also a Rust binding project (https://github.com/virtualritz/after-effects/) that significantly reduces boilerplate compared to C/C++.

*Source: aescripts discord · 2024-09-27 · Tags: `wrapper`, `gui-api`, `cpp`, `rust`, `boilerplate`, `sdk` · [View in Q&A](../qa/wrapper/)*

---
