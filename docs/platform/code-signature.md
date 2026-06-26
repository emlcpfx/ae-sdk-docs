# Code Signature

> 3 Q&As · source: AE plugin dev community Discord

### How should a plugin installer handle upgrades to maintain code signature integrity on macOS?

The installer must completely uninstall all old files before installing the new version. If files from version 1 remain when version 2 is installed (even if those files are no longer needed), the code signature will be broken. Simply overwriting files is insufficient; you need a proper uninstall to remove all old files, then install the new plugin fresh. Adding any extraneous files to a signed package breaks the code signature.

*Tags: `build`, `code-signature`, `deployment`, `macos`*

---

### What is causing the recent code signature breakage in After Effects on macOS?

After Effects was built with a newer version of Xcode that implements stricter code signature checks compared to previous versions. The newer Xcode tampers differently with the build output, resulting in proper signature validation that wasn't enforced before. This is a macOS-specific issue related to how the application binary itself is built and signed.

*Tags: `apple-silicon`, `build`, `code-signature`, `macos`*

---

### Will notarization ensure that a plugin maintains a valid code signature after installation?

Notarization ensures the initial package is properly signed, but it does not prevent code signature breakage if the installer overwrites or leaves extraneous files during upgrade. The installer process itself must be correct to maintain signature integrity; notarization alone cannot compensate for poor installer logic that leaves old files or adds unexpected files to the package.

*Tags: `code-signature`, `deployment`, `macos`*

---
