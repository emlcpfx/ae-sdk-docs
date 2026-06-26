# Plugin Registration

> 1 Q&A · source: AE plugin dev community Discord

### Is a PiPL resource still needed to register After Effects plugins, or can plugins be registered with just the URL info function?

PiPL is still needed even with the PF_Register_effect_ext2 function added in version 23. While you can overwrite some PiPL values in code, the resource file cannot be completely eliminated. After Effects still cannot find effects without a PiPL/rsrc file, though Adobe may update this in the future.

*Tags: `build`, `deployment`, `pipl`, `plugin-registration`*

---
