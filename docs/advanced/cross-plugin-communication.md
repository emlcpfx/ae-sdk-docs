# Cross Plugin Communication

> 2 Q&As · source: AE plugin dev community Discord

### Is there a way for a CEP panel to communicate with a C++ plugin?

There is no direct communication mechanism between CEP and C++ plugins. CEP panels and C++ plugins operate in separate execution contexts and cannot directly exchange data or trigger each other's functions.

*Tags: `cep`, `cross-plugin-communication`, `plugin-architecture`*

---

### How can you pass a string from an AEGP plugin to an FX plugin?

The AEGP will be a separate plugin binary from the FX plugin. You need a C interface to push the string into a queue between the two plugins.

*Tags: `aegp`, `c-interface`, `cross-plugin-communication`, `plugin-architecture`*

---
