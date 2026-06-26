# Plugin Loading

> 2 Q&As · source: AE plugin dev community Discord

### How do you prevent an AE plugin (.aex) from being detected and loaded in Premiere Pro?

Two main approaches: (1) Place the .aex file only in the AE plugins folder and not in the MediaCore folder. (2) Return an error from the EffectMain callback when the host identifier in in_data is "PrMr". You can check the host in EffectMain (which has access to in_data) rather than PluginDataEntryFunction.

*Tags: `effectmain`, `host-detection`, `mediacore`, `plugin-loading`, `premiere-pro`*

---

### Plugins are not loading via MediaCore folder on macOS Tahoe / AE 2026. What's happening?

Multiple reports have been received about plugins failing to load from the MediaCore folder on macOS Tahoe (or with the AE 2026 upgrade). Plugins only work when placed in the Plug-ins folder. This appears to be a newly emerging issue with the OS/AE update.

*Tags: `ae-2026`, `macos-tahoe`, `mediacore`, `plugin-loading`, `regression`*

---
