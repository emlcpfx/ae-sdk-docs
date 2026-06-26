# C Interface

> 1 Q&A · source: AE plugin dev community Discord

### How can you pass data from an AEGP plugin to an effects plugin in After Effects?

You need to create two separate binaries: an AEGP plugin and an FX plugin. Communicate between them using a C interface that pushes the string into a queue. This allows data to be shared across the two plugin types.

*Tags: `aegp`, `c-interface`, `plugin-architecture`, `plugin-communication`*

---
