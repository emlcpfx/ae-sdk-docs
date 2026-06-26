# C Bindings

> 1 Q&A · source: AE plugin dev community Discord

### Is Lua or Python a better choice for embedding a scripting language inside an After Effects plugin?

Lua is a significantly better fit for embedding inside an AE plugin. It is very easy to compile in and has a small footprint. Python is much clunkier to embed because it requires installing libraries on the user's system (asking permission to install packages), which is intrusive. While there may be cleaner ways to embed Python, Lua is far simpler to integrate. Both require similarly tedious C binding work without some kind of automation or code-generation tooling. dvb metareal shipped 'omino lua' as a script-driven drawing plugin using this approach.

*Tags: `c-bindings`, `embedding`, `lua`, `plugin-architecture`, `python`, `scripting`*

---
