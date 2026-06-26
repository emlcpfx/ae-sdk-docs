# Justin

**2 contributions** to AE SDK community knowledge.

Top topics: `xmp`, `metadata`, `extendscript`, `project-data`, `storage`, `premiere`, `uxp`, `cep`, `panel`, `migration`

---

## What is UXP for Premiere Pro and how does it relate to CEP panel migration?

UXP (Unified Extensibility Platform) for Premiere Pro entered public beta in December 2024. It is the successor to CEP for building panels and extensions. Initially it is a web-based development environment, but a hybrid UXP+C++ approach is planned. For C++ computational tasks without the hybrid mode, you would need a service with IPC. The Premiere UXP beta is currently limited in functionality but expected to expand. Plugin developers with existing CEP panels should begin evaluating migration. Resources include the Adobe Creative Cloud Developer forums and the Hyperbrew blog (hyperbrew.co/blog/premiere-pro-uxp-beta). Bolt UXP (hyperbrew.co/resources/bolt-uxp/) is a tool to help with UXP development.

*Source: adobe-plugin-devs · 2024-12-06 · Tags: `premiere`, `uxp`, `cep`, `panel`, `migration`, `hybrid-cpp`, `beta` · [View in Q&A](../qa/premiere/)*

---

## What is the XMP Packet and can it be used to store plugin data in AE projects?

XMP Packets can be read/written via ExtendScript to store custom metadata in AE projects. However, limitations include: the data is not part of AE's undo stack, practical size limit around 500KB (though AE itself can bloat XMP to 50MB+), and it's not undoable. AE Viewer tools exist that can strip XMP to reduce file sizes dramatically (e.g., 50MB to 50KB).

*Source: adobe-plugin-devs · 2024-04-09 · Tags: `xmp`, `metadata`, `extendscript`, `project-data`, `storage` · [View in Q&A](../qa/xmp/)*

---
