# Deployment Target

> 1 Q&A · source: AE plugin dev community Discord

### What causes the 'Invalid Filter 25::3' error on macOS?

This error is typically caused by the Deployment Target being set too high in Xcode. For example, setting it to 12.3 will cause this error on older macOS versions. Try lowering it to support older versions. For Intel builds, supporting back to macOS 10.10 is common.

*Tags: `compatibility`, `deployment-target`, `invalid-filter`, `mac`, `xcode`*

---
