# Symbols

> 1 Q&A · source: AE plugin dev community Discord

### What could cause a plugin to indirectly trigger a crash in Cineware even when the plugin code itself is not executing?

The plugin may be defining a symbol that clashes with Cineware's loading process. This could happen through the plugin's dependencies or exported symbols. Loading Cineware onto a clip before opening the project can work around the issue, suggesting a symbol collision or initialization order problem between the plugin and Cineware.

*Tags: `debugging`, `symbols`, `windows`*

---
