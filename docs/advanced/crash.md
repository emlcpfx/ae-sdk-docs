# Crash

> 4 Q&As · source: AE plugin dev community Discord

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

### After upgrading a plugin from CS5 SDK to the 2021 SDK with MFR support, AE crashes when saving the project. How to debug?

The crash is likely related to sequence data flattening. Check if PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set in global setup - if so, you must handle PF_Cmd_SEQUENCE_FLATTEN properly. When modifying sequence data, be careful about whether you're changing data in the same handle or allocating a new one. Debug steps: (1) Purge Xcode intermediates and clean rebuild. (2) Test SDK sample projects to verify they work. (3) Comment out parts of your plugin to isolate the issue. (4) Copy your code onto a fresh sample project.

*Tags: `crash`, `debugging`, `mfr`, `save`, `sdk-upgrade`, `sequence-flatten`*

---

### How can I debug a crash in After Effects on Windows that occurs during project loading when my plugin is installed, but not on Mac?

Use a debugger with breakpoints on EffectMain to trace when the crash occurs relative to plugin initialization. The crash appears to happen after GlobalSetup and ParamsSetup complete successfully (around 85% project load) with no plugin code executing at that point, suggesting a symbol clash or state corruption rather than direct plugin code failure. Try importing the project into a new empty project as a workaround, or preload the conflicting library (Cineware in this case) before opening the project.

*Tags: `crash`, `debugging`, `macos`, `windows`*

---

### What change in After Effects 2025 beta affects AEGP_StartUndoGroup with null parameter?

Starting from After Effects 2025 beta, passing null to AEGP_StartUndoGroup will cause a crash. In previous versions, this did not cause adverse effects, but it is no longer safe to do so.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup(null);
```

*Tags: `aegp`, `api-change`, `crash`, `debugging`, `macos`, `windows`*

---
