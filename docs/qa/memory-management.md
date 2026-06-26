# Q&A: memory-management

**5 entries** tagged with `memory-management`.

---

## How do you store a struct in global_data?

Create a pointer to the struct in your global data type, then in GlobalSetup allocate it with new: globaldata->structPointer = new StructName(). Use new/malloc (not smart pointers) since AE does special things with the global_data pointer. Optionally delete it in GlobalSetdown, but it's not strictly necessary since GlobalSetdown only gets called when AE closes and the OS will clean up.

*Contributors: [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2024-09-03 · Tags: `global-data`, `memory-management`, `global-setup`, `global-setdown`*

---

## Do PF_LOCK_HANDLE / host_lock_handle functions actually do anything?

According to multiple Adobe engineers, PF_LOCK_HANDLE and host_lock_handle have been no-op dummy functions since AE CS6 (2011) when the codebase moved to 64-bit. Locking/unlocking handles is redundant in 64-bit address space. However, they still provide a safe way to dereference handles (effectively double pointers) and the SDK samples still call them. The DH macro can be used for direct dereferencing instead.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/), [**rowbyte**](../contributors/rowbyte/) · Source: adobe-plugin-devs · 2025-04-03 · Tags: `memory-management`, `handles`, `lock-unlock`, `64-bit`, `cs6`*

---

## Is 5000+ arb dispose calls on AE shutdown normal?

Yes, this is expected behavior. AE's allocators generally don't dispose until available memory is saturated, thread destruction, or app exit. You can test this by setting AE's available memory artificially low (like 2GB) - you'll see many more destruction calls during normal operation. On exit, AE cleans up everything at once.

*Contributors: [**rowbyte**](../contributors/rowbyte/) · Source: adobe-plugin-devs · 2024-12-22 · Tags: `arb-data`, `memory-management`, `dispose`, `shutdown`*

---

## What are the best practices for using global variables in AE plugins?

Global variables are generally frowned upon but widely used. Considerations: (1) std::vectors allocate system memory directly rather than through AE's RAM management, which can cause AE to exceed desired memory usage. (2) Globals are shared by ALL instances of your effect. (3) Memory is only deallocated when AE shuts down, even after global_setdown, so you can't release it through AE's suites. The recommended approach is to allocate memory and store it in a pointer on your global_data handle. For std::vector, you can write a custom allocator that uses AE memory allocation. If the data is constant and read-only, globals are more acceptable.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `global-variables`, `memory-management`, `global-data`, `std-vector`, `allocator`*

---

## Why might an After Effects plugin continue increasing memory usage instead of freeing memory after each frame?

When a plugin fails to use After Effects' memory suites correctly, it may not properly deallocate memory after processing frames. This causes memory usage to accumulate across multiple frames during preview or rendering. The key is to ensure you are using the AE SDK memory suites as documented to properly manage allocation and deallocation of temporary buffers used during frame processing.

*Tags: `memory`, `memory-management`, `ae-plugin`*

---
