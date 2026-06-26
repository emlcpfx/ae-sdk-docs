# Effect Plugin

> 2 Q&As · source: AE plugin dev community Discord

### Should I manually allocate memory in After Effects effect plugins, or follow the sample plugins' approach?

You should not manually allocate memory in effect plugins. Instead, follow what the sample plugins are doing. Manual allocation is problematic because: (1) After Effects won't know whether to deallocate using delete[] or free(), (2) AE won't be able to track memory usage for optimization and management, and (3) it can lead to memory leaks or corruption. Let AE manage memory allocation and deallocation through its own APIs.

*Tags: `best-practices`, `effect-plugin`, `memory`, `reference`*

---

### Can AEGP_RegisterIdleHook be used in an effect plugin to sync sequence data and UI?

AEGP_RegisterIdleHook is a general call that is not instance-specific, even when placed in an effect plugin. It executes once globally rather than once per effect instance. To access sequence data from individual effect instances, you must use EffectCallGeneric() to find and call each instance separately, allowing each instance to perform its own sequence data comparison. This is the only way to access sequence data from an idle hook in an effect plugin.

*Tags: `aegp`, `effect-plugin`, `params`, `sequence-data`*

---
