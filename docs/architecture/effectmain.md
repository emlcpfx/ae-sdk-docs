# Effectmain

> 1 Q&A · source: AE plugin dev community Discord

### How do you prevent an AE plugin (.aex) from being detected and loaded in Premiere Pro?

Two main approaches: (1) Place the .aex file only in the AE plugins folder and not in the MediaCore folder. (2) Return an error from the EffectMain callback when the host identifier in in_data is "PrMr". You can check the host in EffectMain (which has access to in_data) rather than PluginDataEntryFunction.

*Tags: `effectmain`, `host-detection`, `mediacore`, `plugin-loading`, `premiere-pro`*

---
