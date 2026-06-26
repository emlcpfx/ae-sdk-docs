# Host Detection

> 2 Q&As · source: AE plugin dev community Discord

### What causes the 'Not able to acquire AEFX Suite' error in Premiere?

This error can occur during playback in Premiere when code attempts to acquire an AE-specific suite that is not available in the Premiere host. Check if you have a shared function that conditionally calls AEFX suites based on host detection (e.g., if app_id != 'PrMr' then call AEFX suite). The host detection condition might be returning the wrong result, causing Premiere to try acquiring AE-only suites. It could also be a bug with a Premiere suite itself.

*Tags: `aefx-suite`, `error`, `host-detection`, `premiere`, `suite-acquisition`*

---

### How do you prevent an AE plugin (.aex) from being detected and loaded in Premiere Pro?

Two main approaches: (1) Place the .aex file only in the AE plugins folder and not in the MediaCore folder. (2) Return an error from the EffectMain callback when the host identifier in in_data is "PrMr". You can check the host in EffectMain (which has access to in_data) rather than PluginDataEntryFunction.

*Tags: `effectmain`, `host-detection`, `mediacore`, `plugin-loading`, `premiere-pro`*

---
