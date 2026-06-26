# Cpp

> 6 Q&As · source: AE plugin dev community Discord

### How do you debug/log variables from a C++ AE plugin on Windows?

Use OutputDebugString to send debug output that can be viewed with tools like DebugView or Visual Studio's Output window.

```cpp
std::string log1 = "variable: " + std::to_string(variable) + "\n"; OutputDebugString(log1.c_str());
```

*Tags: `cpp`, `debugging`, `logging`, `outputdebugstring`, `windows`*

---

### Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Tags: `code-organization`, `cpp`, `dll`, `prerender`, `smartfx`, `smartrender`*

---

### Are there any good C++ wrapper libraries for the After Effects Filter GUI API?

No well-known general-purpose wrapper exists. Most wrappers that have been built end up being very complex themselves if you try to handle all use cases -- you end up with a non-standard system that is almost as complex as the original, just with more C++ features. Copy-pasting C chunks from the SDK examples works well in practice. That said, even a C wrapper with automatic memory management and state stored in structs would be an improvement over the current state machine. There is also a Rust binding project (https://github.com/virtualritz/after-effects/) that significantly reduces boilerplate compared to C/C++.

*Tags: `boilerplate`, `cpp`, `gui-api`, `rust`, `sdk`, `wrapper`*

---

### How can you manage memory for AESDK objects in C++ plugins?

Create C++ wrappers around all AESDK objects to get automatic memory management. This approach helps prevent memory leaks and ensures proper cleanup of After Effects SDK resources.

*Tags: `aegp`, `cpp`, `memory`, `plugin-development`*

---

### What is a good tool for managing resource cleanup and memory safety in C++ plugin code?

Scope guards are a useful pattern for memory safety in C++ code. They act like smart pointers for any resource, allowing you to use lambda functions to handle cleanup on scope exit. This is particularly valuable for handling exceptions and early returns without explicitly managing deletion at every exit point. Alex Bizeau from maxon recommended the scope_guard library as a reference implementation: https://github.com/ricab/scope_guard/blob/main/scope_guard.hpp. Scope guards work well with both smart pointers and even malloc/free patterns, making them a favorite tool for resource management.

*Tags: `cpp`, `memory`, `reference`, `resource-management`, `smart-pointer`, `tool`*

---

### How can error handling be improved when developing After Effects plugins?

Use std::expected for all function returns instead of traditional error codes. This allows for proper error handling chains using .then() chaining, making error handling much easier and more maintainable. Wrap this in a result struct containing both error codes and results.

*Tags: `aegp`, `best-practices`, `cpp`, `error-handling`*

---
