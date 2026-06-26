# Stream Params

> 1 Q&A · source: AE plugin dev community Discord

### What are the recommended AE plugin communication methods for sharing state across multiple plugins?

Alex Bizeau from Maxon recommended two approaches: (1) PlugPlug for easy IPC communication, and (2) AEGP suite with stream params for managing state. For simpler session-specific flags that don't need to persist, dynamic symbol loading with dlopen to access global variables is an even lighter solution than creating a companion AEGP plugin.

*Tags: `aegp`, `communication`, `ipc`, `plugins`, `stream-params`*

---
