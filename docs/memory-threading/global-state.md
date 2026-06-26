# Global State

> 2 Q&As · source: AE plugin dev community Discord

### How do you share state across multiple effect plugins in the same AE session?

Options: (1) Use a companion AEGP plugin that communicates with all your effect plugins. (2) Use dlopen/symbol loading: define a global variable in one plugin and have other plugins dynamically load a function from it to get/set the shared state. (3) Use PlugPlug DLL for IPC communication. For state that should reset on next AE launch (not persist to prefs), the dlopen approach or AEGP companion are preferred.

*Tags: `aegp`, `dlopen`, `global-state`, `inter-plugin-communication`, `shared-data`*

---

### What is the recommended way to store global data like arrays or vectors in After Effects plugins?

While global variables are technically allowed and commonly used, the best practice for After Effects plugins is to allocate memory and store it in a pointer on your global_data handle. This approach is more RAM-friendly and respects AE's memory allocation and prioritization. Avoid using std::vector directly since it uses system malloc/new rather than AE's memory management. If you must use std::vector, write a custom allocator that uses AE's memory allocation functions. Global variables should only be used for constant data that doesn't change once set, and be aware that such memory persists until AE shuts down and cannot be properly released after global_setdown.

*Tags: `aegp`, `global-state`, `memory`*

---
