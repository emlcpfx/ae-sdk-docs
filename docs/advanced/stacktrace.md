# Stacktrace

> 1 Q&A · source: AE plugin dev community Discord

### How do you get a stacktrace with symbols when using Rust bindings for After Effects on macOS?

Use VSCode with the CodeLLDB extension and configure a launch.json that points to the After Effects application and runs a build task. The configuration should specify the lldb debugger type, request type as 'launch', and include the path to the After Effects application bundle along with a pre-launch build task.

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

*Tags: `debugging`, `macos`, `rust`, `stacktrace`*

---
