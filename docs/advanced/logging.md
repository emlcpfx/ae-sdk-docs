# Logging

> 4 Q&As · source: AE plugin dev community Discord

### What's a good approach for logging in AE plugins?

Use spdlog library which supports file rotation (configurable file size and number of files to keep). For performance-sensitive plugins, use conditional logging: check for the existence of a specific file (e.g., 'plugin_log.txt') during GlobalSetup, and only enable logging when that file is present. This lets users enable logging on demand for debugging.

*Tags: `debugging`, `logging`, `performance`, `spdlog`*

---

### How do you debug/log variables from a C++ AE plugin on Windows?

Use OutputDebugString to send debug output that can be viewed with tools like DebugView or Visual Studio's Output window.

```cpp
std::string log1 = "variable: " + std::to_string(variable) + "\n"; OutputDebugString(log1.c_str());
```

*Tags: `cpp`, `debugging`, `logging`, `outputdebugstring`, `windows`*

---

### How can you add conditional logging to an After Effects plugin without slowing down renders?

Implement a file-based toggle for logging by checking for a specific file (e.g., 'pixel_sorter_log.txt') in global setup at startup. This allows users to enable detailed logging only when needed for debugging, without impacting render performance during normal operation.

*Tags: `debugging`, `logging`, `macos`, `performance`, `plugin-development`*

---

### What logging library is recommended for After Effects plugins with file rotation capabilities?

spdlog is recommended for plugin logging. It provides file rotation functionality where you can define the maximum size of individual log files and how many log files to retain, making it suitable for long-running plugin operations.

*Tags: `debugging`, `logging`, `open-source`, `tool`*

---
