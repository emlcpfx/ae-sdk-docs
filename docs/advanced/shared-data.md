# Shared Data

> 1 Q&A · source: AE plugin dev community Discord

### How do you share state across multiple effect plugins in the same AE session?

Options: (1) Use a companion AEGP plugin that communicates with all your effect plugins. (2) Use dlopen/symbol loading: define a global variable in one plugin and have other plugins dynamically load a function from it to get/set the shared state. (3) Use PlugPlug DLL for IPC communication. For state that should reset on next AE launch (not persist to prefs), the dlopen approach or AEGP companion are preferred.

*Tags: `aegp`, `dlopen`, `global-state`, `inter-plugin-communication`, `shared-data`*

---
