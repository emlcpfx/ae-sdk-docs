# Plugins

> 2 Q&As · source: AE plugin dev community Discord

### How can you set a flag for an entire After Effects session across all plugins without using preferences or a companion AEGP plugin?

You can use dynamic symbol loading with dlopen to access a global variable from one plugin in other plugins. Define a global variable in one of your plugins and dynamically load a function from that plugin using dlopen. Other plugins can then call this function to read the state of the variable. This approach avoids the need for preferences (which persist across sessions) or a separate companion AEGP plugin.

*Tags: `aegp`, `cross-platform`, `ipc`, `plugins`, `session-state`*

---

### What are the recommended AE plugin communication methods for sharing state across multiple plugins?

Alex Bizeau from Maxon recommended two approaches: (1) PlugPlug for easy IPC communication, and (2) AEGP suite with stream params for managing state. For simpler session-specific flags that don't need to persist, dynamic symbol loading with dlopen to access global variables is an even lighter solution than creating a companion AEGP plugin.

*Tags: `aegp`, `communication`, `ipc`, `plugins`, `stream-params`*

---
