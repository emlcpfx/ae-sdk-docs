# Notarization

> 3 Q&As · source: AE plugin dev community Discord

### What caused plugins to fail to load in AE 25.2 on macOS?

The issue was related to code signature validation. Adobe built AE 25.2 with a newer Xcode that enforces stricter code signature checks. Multiple root causes were identified: (1) Unsigned plugins now fail to load on Apple Silicon. (2) Installers that don't properly clean up old files can break code signatures - if v1 had file X and v2 removes it, but the installer leaves it, the signature breaks. (3) Some plugins had 0-byte .plugin files after the AE upgrade. Solution: Ensure proper code signing, notarization, and clean installation (remove all old files before installing new version).

*Tags: `ae-25-2`, `apple-silicon`, `code-signing`, `installer`, `macos`, `notarization`*

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
