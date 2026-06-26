# Resource Management

> 2 Q&As · source: AE plugin dev community Discord

### If a resource fails to allocate, does it still need to be disposed?

No, if a resource fails to allocate, it does not need to be disposed of. Only successfully allocated resources require cleanup and disposal.

*Tags: `debugging`, `memory`, `resource-management`*

---

### What is a good tool for managing resource cleanup and memory safety in C++ plugin code?

Scope guards are a useful pattern for memory safety in C++ code. They act like smart pointers for any resource, allowing you to use lambda functions to handle cleanup on scope exit. This is particularly valuable for handling exceptions and early returns without explicitly managing deletion at every exit point. Alex Bizeau from maxon recommended the scope_guard library as a reference implementation: https://github.com/ricab/scope_guard/blob/main/scope_guard.hpp. Scope guards work well with both smart pointers and even malloc/free patterns, making them a favorite tool for resource management.

*Tags: `cpp`, `memory`, `reference`, `resource-management`, `smart-pointer`, `tool`*

---
