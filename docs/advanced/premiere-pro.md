# Premiere Pro

> 4 Q&As · source: AE plugin dev community Discord

### How do you view the current pixel bit depth in Premiere Pro?

Use the DogEars feature. You can look up the keyboard shortcut online, or edit the debug database to enable it. This feature is only available in Premiere Pro, not in After Effects.

*Tags: `bit-depth`, `debugging`, `dogears`, `premiere-pro`*

---

### Can the same .aex plugin work in both After Effects and Premiere Pro?

Yes, but it requires correct coding. Premiere does many things differently than AE and not all features are supported in both hosts. The AE SDK documentation has a dedicated section for Premiere Pro compatibility (https://ae-plugins.docsforadobe.dev/). Notably, Premiere does not support SmartRender, which is a significant difference. There are also long-standing bugs in Premiere's implementation of AE plugin APIs (present since at least 2006) that are regularly reported to but largely ignored by Adobe.

*Tags: `aex`, `compatibility`, `cross-host`, `premiere-pro`, `smartrender`*

---

### Is there a way to show a progress bar in Premiere Pro for video filter / AE effect rendering?

In the AE SDK, there is a function for reporting render progress back to the host, which results in the standard AE progress bar being displayed/updated. It's unclear whether Premiere supports this. There is also an unofficial way used by some of Adobe's native effects to display custom progress, but the details may not be publicly disclosed.

*Tags: `premiere-pro`, `progress-bar`, `rendering`, `ui-feedback`*

---

### How do you prevent an AE plugin (.aex) from being detected and loaded in Premiere Pro?

Two main approaches: (1) Place the .aex file only in the AE plugins folder and not in the MediaCore folder. (2) Return an error from the EffectMain callback when the host identifier in in_data is "PrMr". You can check the host in EffectMain (which has access to in_data) rather than PluginDataEntryFunction.

*Tags: `effectmain`, `host-detection`, `mediacore`, `plugin-loading`, `premiere-pro`*

---
