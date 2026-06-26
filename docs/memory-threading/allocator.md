# Allocator

> 1 Q&A · source: AE plugin dev community Discord

### What are the best practices for using global variables in AE plugins?

Global variables are generally frowned upon but widely used. Considerations: (1) std::vectors allocate system memory directly rather than through AE's RAM management, which can cause AE to exceed desired memory usage. (2) Globals are shared by ALL instances of your effect. (3) Memory is only deallocated when AE shuts down, even after global_setdown, so you can't release it through AE's suites. The recommended approach is to allocate memory and store it in a pointer on your global_data handle. For std::vector, you can write a custom allocator that uses AE memory allocation. If the data is constant and read-only, globals are more acceptable.

*Tags: `allocator`, `global-data`, `global-variables`, `memory-management`, `std-vector`*

---
