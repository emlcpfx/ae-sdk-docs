# Code Reuse

> 2 Q&As · source: AE plugin dev community Discord

### How do you write cross-host plugin code that works for AE, Premiere, OFX, and other hosts?

Write host/format-independent processing code and create thin adapter/wrapper layers per host. This approach has been used successfully for AE/PPro/OFX/Nuke/Frei0r formats. The OFX SDK (partially based on AE SDK) is cleaner in some areas - for example, OFX supports thousands of plugins in a single binary (useful for suites like GMIC with ~2000 plugins), while AE requires separate PiPL/loader for each. When porting, the processing core stays the same and only the parameter declaration and pixel access patterns need per-host adaptation.

*Tags: `architecture`, `code-reuse`, `cross-platform`, `ofx`, `plugin-framework`*

---

### How can I make a plugin for 8-bit, 16-bit, and 32-bit projects share the same internal codebase?

Use C++ templates to create 3 instances of your algorithm with different data types and range constants. This allows you to write the core logic once and instantiate it for PF_Pixel8, PF_Pixel16, and PF_PixelFloat with appropriate range constants for each bit depth.

*Tags: `bit-depth`, `code-reuse`, `pixel-format`, `smart-render`, `templates`*

---
