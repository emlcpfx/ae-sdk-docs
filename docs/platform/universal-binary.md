# Universal Binary

> 1 Q&A · source: AE plugin dev community Discord

### How do you build AE plugins for Apple Silicon (ARM64)?

You can build on an M1 Mac (even a cheap Mac Mini works as a build machine). It's not strictly required to build on M1, but it helps for testing. The key thing many people miss is that you must add ARM64 to your PiPL resource file - just building as Universal Binary in Xcode is not enough. Without the ARM64 entry in the PiPL, AE will show the 'not yet compatible' warning.

*Tags: `apple-silicon`, `arm64`, `mac`, `pipl`, `universal-binary`, `xcode`*

---
