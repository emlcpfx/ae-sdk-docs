# Codesign

> 3 Q&As · source: AE plugin dev community Discord

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
