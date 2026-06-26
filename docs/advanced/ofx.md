# Ofx

> 4 Q&As · source: AE plugin dev community Discord

### How do you write cross-host plugin code that works for AE, Premiere, OFX, and other hosts?

Write host/format-independent processing code and create thin adapter/wrapper layers per host. This approach has been used successfully for AE/PPro/OFX/Nuke/Frei0r formats. The OFX SDK (partially based on AE SDK) is cleaner in some areas - for example, OFX supports thousands of plugins in a single binary (useful for suites like GMIC with ~2000 plugins), while AE requires separate PiPL/loader for each. When porting, the processing core stays the same and only the parameter declaration and pixel access patterns need per-host adaptation.

*Tags: `architecture`, `code-reuse`, `cross-platform`, `ofx`, `plugin-framework`*

---

### What floating license server options exist for OFX plugins (e.g., for Flame)?

RLM (Reprise License Manager) is one option used by companies like Maxon for their plugins. However, the options for floating license servers tend to be pricey. The aescripts licensing system does not support floating licenses. For Flame-specific plugins with Linux floating license requirements, RLM is the most commonly referenced solution in the community.

*Tags: `flame`, `floating-license`, `licensing`, `linux`, `ofx`, `rlm`*

---

### Where can I find resources and community support for developing OFX plugins for Flame on Linux?

There are several resources: (1) The official OpenFX Slack channel (now under ASWF - Academy Software Foundation) is the recommended place, though access may be restricted to specific email domains like linuxfoundation.org or aswf.io. (2) There is an unofficial OFX Discord server with a dedicated Flame channel, though it's not very active. (3) OpenFX also has a mailing list you can join. (4) You can request a free developer license from Autodesk for Flame testing. For DaVinci Resolve OFX debugging, Maxon has published documentation that may also be helpful.

*Tags: `autodesk`, `community`, `flame`, `linux`, `ofx`, `openfx`, `resolve`*

---

### What are the advantages of writing host and plugin format-independent code for multiple platforms?

Writing host and plugin format-independent code frees up significant resources by allowing the processing code to be written once and only occasionally requiring wrapper project updates. This approach has proven effective across multiple platforms including After Effects, Premiere Pro, OFX, Nuke, and Frei0r. For example, when porting the GMIC plugin suite (containing ~2000 plugins) to OFX, all plugins fit in a single file, whereas for AE/Premiere Pro, 2000 tiny individual loader plugins had to be created. The OFX SDK has also benefited from this approach by eliminating detours and inconveniences present in the AE SDK, such as PiPL requirements.

*Tags: `aegp`, `cross-platform`, `ofx`, `plugin-architecture`, `premiere`*

---
