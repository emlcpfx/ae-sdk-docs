# Quarantine

> 1 Q&A · source: AE plugin dev community Discord

### After signing, notarizing, and stapling a Mac plugin, why does it become untrusted after uploading and re-downloading?

This is a known issue where the notarization appears successful (notarytool says Accepted) but spctl reports 'rejected / source=Unnotarized Developer ID' after download. The downloaded file may get an Apple quarantine xattr. This can happen with AEGP plugins that are .plugin bundles. The issue may be related to how the bundle is zipped/unzipped, or how the stapling interacts with the bundle structure. Check the xattr flags after download with 'xattr -l <path>' to see if quarantine is applied.

```cpp
codesign --options runtime --force --deep --timestamp -strict --sign "Developer ID Application: Name (TEAMID)" <path/to/.plugin/file>
xcrun notarytool submit <path/to/zipped/.plugin/file> --keychain-profile "Profile" --wait
xcrun stapler staple <path/to/.plugin/file>
```

*Tags: `aegp`, `code-signing`, `macos`, `notarization`, `quarantine`, `spctl`*

---
