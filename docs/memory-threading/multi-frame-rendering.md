# Multi Frame Rendering

> 2 Q&As · source: AE plugin dev community Discord

### How much work is required to update a simple AE plugin for Multi-Frame Rendering (MFR)?

For simple plugins that don't use sequence data or caching, there is not much to do besides rebuilding. The complexity comes when you have GPU plugins, custom caching, or sequence data which require more significant refactoring for thread safety.

*Tags: `mfr`, `multi-frame-rendering`, `plugin-update`, `thread-safety`*

---

### Should I use multithreading (iterate_generic) or MFR (Multi-Frame Rendering) in my plugin, or both?

Use both. MFR and multithreading can coexist because not all steps in the single-frame rendering pipeline are multi-threadable. MFR shines for sequential parts, while multithreaded parts (via iterate callbacks) are critical for performance. Mixing MFR with MT doesn't necessarily cause competition because MT-friendly code tends to be CPU cache friendly, while other parts may leave the CPU waiting for RAM. When using iteration callbacks, AE may prioritize between MFR and MT threads. The AE engineering team tested these scenarios extensively.

*Tags: `iterate-generic`, `mfr`, `multi-frame-rendering`, `multithreading`, `performance`*

---
