# Uxp

> 12 Q&As · source: AE plugin dev community Discord

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

### Where can I find information about Premiere UXP Beta?

Alex Bizeau from Maxon published a blog post with information on Premiere UXP Beta. The blog post was shared as a resource for developers interested in learning more about the Premiere UXP Beta.

*Tags: `premiere`, `reference`, `resource`, `uxp`*

---

### What is the best documentation to get started with UXP for Premiere Pro?

According to the discussion, there is a comprehensive article that contains all the links for UXP documentation, forums, and other resources for Premiere Pro development. UXP in Premiere Pro is currently limited but will be expanded soon.

*Tags: `premiere`, `reference`, `ui`, `uxp`*

---

### Can C++ be integrated as a backend for UXP plugins in Premiere Pro, or is IPC required for computational tasks?

There is a new hybrid UXP approach that combines HTML and C++ backends, though the exact implementation details were not fully explored in the conversation. This hybrid solution is currently in beta and represents a promising development for allowing computational tasks without requiring a separate service with IPC.

*Tags: `build`, `premiere`, `uxp`*

---

### What is Bolt UXP and how does it relate to Premiere Pro plugin development?

Bolt UXP is a resource mentioned in the discussion as relevant to Premiere Pro UXP development. Justin shared this as a reference for developers looking to build UXP-based plugins.

*Tags: `open-source`, `premiere`, `tool`, `uxp`*

---

### How long will the CEP engine remain available in Premiere Pro before it's deprecated?

The conversation mentions that UXP for Premiere Pro is now in public beta and encourages CEP panel developers to migrate, but no specific timeline for CEP deprecation was provided in the response.

*Tags: `cep`, `cross-platform`, `deployment`, `premiere`, `uxp`*

---

### What is the current status of UXP support for Premiere Pro?

UXP for Premiere Pro is now available in public beta. Developers with existing CEP panels are encouraged to migrate to UXP and provide feedback to the Premiere Pro team about their migration needs. More information is available at: https://forums.creativeclouddeveloper.com/t/uxp-now-available-in-premiere-pro-beta/8795

*Tags: `deployment`, `premiere`, `resource`, `ui`, `uxp`*

---

### What is the best documentation to start learning UXP for Premiere Pro?

UXP in Premiere Pro is currently very limited but will change soon. There is a comprehensive article that contains all the relevant links to documentation, forums, and other resources to get started with UXP development for Premiere Pro.

*Tags: `documentation`, `premiere`, `reference`, `ui`, `uxp`*

---

### Can you integrate a C++ backend with UXP for Premiere Pro, or do you need to use a separate service with IPC for computational tasks?

There is a new hybrid UXP approach that combines HTML and C++ functionality, though details on how it works are still emerging. This hybrid approach is currently in beta and promises to allow C++ backend integration rather than requiring a separate IPC service for computational tasks.

*Tags: `c++`, `premiere`, `ui`, `uxp`*

---

### What is Bolt UXP?

Bolt UXP is a framework or resource for UXP development in Premiere Pro. It was shared as a relevant tool for developers working with UXP plugins.

*Tags: `premiere`, `reference`, `tool`, `ui`, `uxp`*

---
