# Registration

> 1 Q&A · source: AE plugin dev community Discord

### Is PiPL still required for After Effects plugins, or can plugins be registered without it?

PiPL is still required for After Effects plugins. While the function PF_Register_effect_ext2 was added two versions ago (in AE 23) to allow registration with URL info, this does not eliminate the need for PiPL/rsrc files. Plugins without PiPL cannot be found by After Effects. The new entry point functionality has advantages, but PiPL is still mandatory, though some of its values can be overwritten in code. However, OFX (which is partly based on the AE SDK) successfully eliminated PiPL entirely, making it more flexible for multi-plugin deployment.

*Tags: `aegp`, `pipl`, `plugin-architecture`, `registration`*

---
