# Templates

> 2 Q&As · source: AE plugin dev community Discord

### How can I make a plugin for 8-bit, 16-bit, and 32-bit projects share the same internal codebase?

Use C++ templates to create 3 instances of your algorithm with different data types and range constants. This allows you to write the core logic once and instantiate it for PF_Pixel8, PF_Pixel16, and PF_PixelFloat with appropriate range constants for each bit depth.

*Tags: `bit-depth`, `code-reuse`, `pixel-format`, `smart-render`, `templates`*

---

### How can I make a plugin support 8-bit, 16-bit, and 32-bit color depths without duplicating code for each bit depth?

Use C++ templates to create a single implementation that can be instantiated for different data types and range constants. This allows you to share the same core logic while supporting multiple bit depths. You can also manually convert higher bit-depth input to 8-bit within the plugin itself if needed, rather than relying on After Effects' automatic conversion which may introduce unwanted color noise.

*Tags: `bit-depth`, `c++`, `codebase-reuse`, `color-processing`, `sdk`, `templates`*

---
