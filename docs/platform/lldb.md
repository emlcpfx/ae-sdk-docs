# Lldb

> 2 Q&As · source: AE plugin dev community Discord

### How do you debug AE plugins built with Rust on macOS?

Use VSCode with the CodeLLDB extension. Create a launch.json that launches AE directly and specifies 'sourceLanguages': ['rust']. The sourceLanguages line is important - without it, launching AE from LLDB can cause crashes. Use a preLaunchTask to build the plugin before launching.

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build",
      "sourceLanguages": ["rust"]
    }
  ]
}
```

*Tags: `debugging`, `lldb`, `macos`, `rust`, `vscode`*

---

### How do you get a stacktrace with symbols when using the Rust bindings for After Effects on macOS?

Use VSCode with the CodeLLDB extension and configure a launch.json file to attach the debugger to the After Effects application. Set up a launch configuration that points to the After Effects executable and runs a build task before launching.

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build"
    }
  ]
}
```

*Tags: `debugging`, `lldb`, `macos`, `rust`*

---
