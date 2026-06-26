# Codebase Reuse

> 1 Q&A · source: AE plugin dev community Discord

### How can I make a plugin support 8-bit, 16-bit, and 32-bit color depths without duplicating code for each bit depth?

Use C++ templates to create a single implementation that can be instantiated for different data types and range constants. This allows you to share the same core logic while supporting multiple bit depths. You can also manually convert higher bit-depth input to 8-bit within the plugin itself if needed, rather than relying on After Effects' automatic conversion which may introduce unwanted color noise.

*Tags: `bit-depth`, `c++`, `codebase-reuse`, `color-processing`, `sdk`, `templates`*

---
