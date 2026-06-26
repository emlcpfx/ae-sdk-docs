# Migration

> 4 Q&As · source: AE plugin dev community Discord

### How do you handle backward compatibility when adding new parameters to existing plugins?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on new parameters. This flag tells AE to use the parameter's 'value' field (not 'dephault') when loading projects saved before this parameter existed. For more complex migration scenarios, add a hidden parameter storing a version number, and in sequence data setup, parse that version to reset values accordingly.

*Tags: `backward-compatibility`, `migration`, `params`, `pf-param-flag`, `versioning`*

---

### What is UXP for Premiere Pro and how does it relate to CEP panel migration?

UXP (Unified Extensibility Platform) for Premiere Pro entered public beta in December 2024. It is the successor to CEP for building panels and extensions. Initially it is a web-based development environment, but a hybrid UXP+C++ approach is planned. For C++ computational tasks without the hybrid mode, you would need a service with IPC. The Premiere UXP beta is currently limited in functionality but expected to expand. Plugin developers with existing CEP panels should begin evaluating migration. Resources include the Adobe Creative Cloud Developer forums and the Hyperbrew blog (hyperbrew.co/blog/premiere-pro-uxp-beta). Bolt UXP (hyperbrew.co/resources/bolt-uxp/) is a tool to help with UXP development.

*Tags: `beta`, `cep`, `hybrid-cpp`, `migration`, `panel`, `premiere`, `uxp`*

---

### How long will the CEP engine remain available in Premiere Pro before the transition to UXP is required?

This question was asked but not answered in the conversation. Erin F. shared information about UXP for Premiere Pro entering public beta and encouraged developers with CEP panels to communicate their migration needs to the Premiere Pro team, but did not provide a specific timeline for CEP deprecation.

*Tags: `cep`, `migration`, `premiere`, `uxp`*

---

### What is the status of UXP support for Premiere Pro and where can I learn more about it?

UXP for Premiere Pro is now in public beta. Developers with existing CEP panels should review the announcement and communicate their migration requirements to the Premiere Pro team. More information is available at the Creative Cloud Developer Forums: https://forums.creativeclouddeveloper.com/t/uxp-now-available-in-premiere-pro-beta/8795

*Tags: `cep`, `deployment`, `migration`, `premiere`, `uxp`*

---
