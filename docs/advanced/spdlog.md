# Spdlog

> 1 Q&A · source: AE plugin dev community Discord

### What's a good approach for logging in AE plugins?

Use spdlog library which supports file rotation (configurable file size and number of files to keep). For performance-sensitive plugins, use conditional logging: check for the existence of a specific file (e.g., 'plugin_log.txt') during GlobalSetup, and only enable logging when that file is present. This lets users enable logging on demand for debugging.

*Tags: `debugging`, `logging`, `performance`, `spdlog`*

---
