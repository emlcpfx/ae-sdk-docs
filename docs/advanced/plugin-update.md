# Plugin Update

> 1 Q&A · source: AE plugin dev community Discord

### How much work is required to update a simple AE plugin for Multi-Frame Rendering (MFR)?

For simple plugins that don't use sequence data or caching, there is not much to do besides rebuilding. The complexity comes when you have GPU plugins, custom caching, or sequence data which require more significant refactoring for thread safety.

*Tags: `mfr`, `multi-frame-rendering`, `plugin-update`, `thread-safety`*

---
