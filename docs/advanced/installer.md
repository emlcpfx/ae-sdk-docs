# Installer

> 2 Q&As · source: AE plugin dev community Discord

### What caused plugins to fail to load in AE 25.2 on macOS?

The issue was related to code signature validation. Adobe built AE 25.2 with a newer Xcode that enforces stricter code signature checks. Multiple root causes were identified: (1) Unsigned plugins now fail to load on Apple Silicon. (2) Installers that don't properly clean up old files can break code signatures - if v1 had file X and v2 removes it, but the installer leaves it, the signature breaks. (3) Some plugins had 0-byte .plugin files after the AE upgrade. Solution: Ensure proper code signing, notarization, and clean installation (remove all old files before installing new version).

*Tags: `ae-25-2`, `apple-silicon`, `code-signing`, `installer`, `macos`, `notarization`*

---

### How do you check AE plugin compatibility with older versions of After Effects?

You can get installers for AE 2019 and older from the ProDesignTools website (https://prodesigntools.com/adobe-direct-download-links.html). For all newer versions, contact Adobe support for an offline installer. They typically respond quickly and provide installers within minutes.

*Tags: `compatibility`, `installer`, `older-versions`, `testing`*

---
