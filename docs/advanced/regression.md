# Regression

> 3 Q&As · source: AE plugin dev community Discord

### What causes AEGP_StartUndoGroup(null) to crash in AE 2025?

Starting from AE 2025 beta, passing null/NULL to AEGP_StartUndoGroup causes a crash. Use an empty string "" instead. AEGP_StartUndoGroup("") works correctly and behaves as expected (no entry in the undo stack). This regression has occurred before in earlier AE betas but was corrected.

```cpp
// CRASHES in AE 2025:
suites.UtilitySuite5()->AEGP_StartUndoGroup(null);

// WORKS:
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `ae-2025`, `aegp`, `crash`, `regression`, `undo`*

---

### How does AE's Custom UI / arb params bug in AE 2025 manifest?

In AE 2025 (25.0.1), custom UI and arb params display glitches: one arb param's UI can corrupt other arb UIs, causing flashes or scrambled/duplicated content. The likely culprit is NewImageFromBuffer in DrawBot - every call may be setting pixels of all retained DRAWBOT_ImageRef within the same context. Adobe acknowledged fixing something in arb handling, but it may have introduced this unintended consequence. The issue affects multiple third-party plugins including Trapcode.

*Tags: `ae-2025`, `arb-data`, `bug`, `custom-ui`, `drawbot`, `regression`*

---

### Plugins are not loading via MediaCore folder on macOS Tahoe / AE 2026. What's happening?

Multiple reports have been received about plugins failing to load from the MediaCore folder on macOS Tahoe (or with the AE 2026 upgrade). Plugins only work when placed in the Plug-ins folder. This appears to be a newly emerging issue with the OS/AE update.

*Tags: `ae-2026`, `macos-tahoe`, `mediacore`, `plugin-loading`, `regression`*

---
