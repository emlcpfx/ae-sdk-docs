# Best Practices

> 4 Q&As · source: AE plugin dev community Discord

### Should I manually allocate memory in After Effects effect plugins, or follow the sample plugins' approach?

You should not manually allocate memory in effect plugins. Instead, follow what the sample plugins are doing. Manual allocation is problematic because: (1) After Effects won't know whether to deallocate using delete[] or free(), (2) AE won't be able to track memory usage for optimization and management, and (3) it can lead to memory leaks or corruption. Let AE manage memory allocation and deallocation through its own APIs.

*Tags: `best-practices`, `effect-plugin`, `memory`, `reference`*

---

### How can error handling be improved when developing After Effects plugins?

Use std::expected for all function returns instead of traditional error codes. This allows for proper error handling chains using .then() chaining, making error handling much easier and more maintainable. Wrap this in a result struct containing both error codes and results.

*Tags: `aegp`, `best-practices`, `cpp`, `error-handling`*

---

### What best practices should be followed for managing After Effects handles in C++ plugins?

Wrap all AE handles into classes with proper construction and destruction in destructors, and use smart pointers to manage them. Additionally, implement a greater allocation class manager that clears allocations when quitting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug mode, use a cache validator to verify no extra memory remains allocated at endpoints.

*Tags: `aegp`, `best-practices`, `debugging`, `memory`, `smart-pointers`*

---

### Should developers use the After Effects GPU Suite for plugin development?

No, the AE GPU Suite should be avoided for plugin development. Instead, developers should implement custom GPU renderer backends that support Metal, DirectX, and OpenGL with proper GPU buffer interoperability, as demonstrated by modern plugin architectures like Red Giant Universe's approach.

*Tags: `best-practices`, `debugging`, `gpu`, `metal`, `opengl`*

---
