# Sdk Upgrade

> 1 Q&A · source: AE plugin dev community Discord

### After upgrading a plugin from CS5 SDK to the 2021 SDK with MFR support, AE crashes when saving the project. How to debug?

The crash is likely related to sequence data flattening. Check if PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set in global setup - if so, you must handle PF_Cmd_SEQUENCE_FLATTEN properly. When modifying sequence data, be careful about whether you're changing data in the same handle or allocating a new one. Debug steps: (1) Purge Xcode intermediates and clean rebuild. (2) Test SDK sample projects to verify they work. (3) Comment out parts of your plugin to isolate the issue. (4) Copy your code onto a fresh sample project.

*Tags: `crash`, `debugging`, `mfr`, `save`, `sdk-upgrade`, `sequence-flatten`*

---
