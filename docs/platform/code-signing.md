# Code Signing

> 8 Q&As · source: AE plugin dev community Discord

### What caused plugins to fail to load in AE 25.2 on macOS?

The issue was related to code signature validation. Adobe built AE 25.2 with a newer Xcode that enforces stricter code signature checks. Multiple root causes were identified: (1) Unsigned plugins now fail to load on Apple Silicon. (2) Installers that don't properly clean up old files can break code signatures - if v1 had file X and v2 removes it, but the installer leaves it, the signature breaks. (3) Some plugins had 0-byte .plugin files after the AE upgrade. Solution: Ensure proper code signing, notarization, and clean installation (remove all old files before installing new version).

*Tags: `ae-25-2`, `apple-silicon`, `code-signing`, `installer`, `macos`, `notarization`*

---

### Should .aex plugins be code signed on Windows?

.aex plugins are regular DLLs (just renamed), so they can be codesigned. Microsoft currently allows unsigned DLLs to be loaded from a signed process, but they have hinted this might change in the future (was supposed to happen with Win11). On macOS, a plugin is a bundle (renamed folder), and you don't need to sign the actual plugin binary inside it as long as the bundle itself is signed.

*Tags: `aex`, `code-signing`, `dll`, `macos`, `security`, `windows`*

---

### Do Mac plugins need to be notarized in addition to codesigning for distribution via the aescripts app?

Yes, codesigning alone is not enough. The plugins also need to be notarized by Apple for distribution.

*Tags: `aescripts`, `code-signing`, `distribution`, `macos`, `notarization`*

---

### After signing, notarizing, and stapling a Mac plugin, why does it become untrusted after uploading and re-downloading?

This is a known issue where the notarization appears successful (notarytool says Accepted) but spctl reports 'rejected / source=Unnotarized Developer ID' after download. The downloaded file may get an Apple quarantine xattr. This can happen with AEGP plugins that are .plugin bundles. The issue may be related to how the bundle is zipped/unzipped, or how the stapling interacts with the bundle structure. Check the xattr flags after download with 'xattr -l <path>' to see if quarantine is applied.

```cpp
codesign --options runtime --force --deep --timestamp -strict --sign "Developer ID Application: Name (TEAMID)" <path/to/.plugin/file>
xcrun notarytool submit <path/to/zipped/.plugin/file> --keychain-profile "Profile" --wait
xcrun stapler staple <path/to/.plugin/file>
```

*Tags: `aegp`, `code-signing`, `macos`, `notarization`, `quarantine`, `spctl`*

---

### Is there a skeleton/starter project for building an AE plugin with Vulkan GPU rendering?

Yes — AE_Skeleton_Vulkan by Eric CPFX (emlcpfx) is a complete CMake-based skeleton that incorporates the Vulkan SDK into an AE plugin. It also includes build scripts and a sample script for signing, notarizing, and stapling on Mac. Repository: https://github.com/emlcpfx/AE_Skeleton_Vulkan

*Tags: `build`, `cmake`, `code-signing`, `gpu`, `macos`, `skeleton`, `vulkan`*

---

### Why are After Effects plugins failing to load on Apple Silicon Macs in versions 25.2 and 25.3?

The investigation suggests code signing requirements have become stricter on Apple Silicon Macs. While entitlements haven't changed, Adobe may be using a newer Xcode version that enforces stricter code signing validation. Additionally, there may be a new hardening behavior in Apple's code-signing that creates a problematic interaction between After Effects' own signatures and plugin signatures.

*Tags: `apple-silicon`, `code-signing`, `deployment`, `macos`*

---

### How can you code sign and notarize After Effects plugins for macOS?

You need an Apple Developer account (costs $99/year). With this account, you can sign and notarize your plugins. The process involves using Xcode's code signing tools to sign the plugin executables and then submitting them to Apple's notarization service.

*Tags: `code-signing`, `deployment`, `macos`*

---

### Why are plugins failing to load on Apple Silicon Macs in After Effects 25.2 and 25.3?

According to investigation by Maxon and Adobe, the issue appears to be related to code signing. Apple may have introduced stricter code-signing requirements or hardening behavior in newer Xcode versions. There seems to be a potential interaction between After Effects' own code signatures and plugin signatures, though the exact cause is still under investigation. Even plugins that are properly signed and notarized are experiencing load failures on Apple Silicon Macs.

*Tags: `apple-silicon`, `code-signing`, `debugging`, `deployment`, `macos`*

---
