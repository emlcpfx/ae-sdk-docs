# Communication

> 2 Q&As · source: AE plugin dev community Discord

### How can C++ plugins communicate with CEP panels to exchange parameter values?

There are several approaches: (1) CEP can change parameter values via scripting, and C++ can open a CEP by executing a script that calls it; (2) C++ can check if CEP is open using TCP protocol or a hidden checkbox that CEP activates via scripting; (3) Use a hidden panel running as a server that receives requests from the plugin and adjusts values in CEP; (4) For CEP-to-plugin communication, store values in JavaScript memory using global variables, or use temporary files if memory size is a concern; (5) In AE-only scenarios, set a hidden checkbox activated by script to catch the value.

*Tags: `cep`, `communication`, `params`, `scripting`, `ui`*

---

### What are the recommended AE plugin communication methods for sharing state across multiple plugins?

Alex Bizeau from Maxon recommended two approaches: (1) PlugPlug for easy IPC communication, and (2) AEGP suite with stream params for managing state. For simpler session-specific flags that don't need to persist, dynamic symbol loading with dlopen to access global variables is an even lighter solution than creating a companion AEGP plugin.

*Tags: `aegp`, `communication`, `ipc`, `plugins`, `stream-params`*

---
