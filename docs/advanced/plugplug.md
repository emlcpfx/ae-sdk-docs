# Plugplug

> 1 Q&A · source: AE plugin dev community Discord

### What is plugplug.DLL and how can you use it from C++ plugins?

plugplug.DLL is the gateway to communicate with CEP (Common Extensibility Platform) from C++. It allows sending events between C++ plugins and CEP panels. You can reverse-engineer the DLL's exports using depends.exe and reference the Illustrator SDK sample as a guide (it's not 1:1 but close enough). A GitHub repo with a working implementation is available: https://github.com/Trentonom0r3/AE-SDK-CEP-UTILS

*Tags: `cep`, `dll`, `inter-process-communication`, `plugplug`*

---
