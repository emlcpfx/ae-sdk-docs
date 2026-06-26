# Plugin Load

> 1 Q&A · source: AE plugin dev community Discord

### What could cause a plugin to fail loading during After Effects startup with a 'bad filter' main entry point error when drivers and Windows are up to date?

The issue likely stems from a missing or incompatible dependency rather than driver or OS problems. Since no Vulkan instance errors appear in logs, the failure occurs before the global data thread initialization, suggesting the problem happens during plugin initialization or early in the entry point execution.

*Tags: `debugging`, `deployment`, `plugin-load`, `windows`*

---
