# Global Data

> 4 Q&As · source: AE plugin dev community Discord

### Why does Media Encoder not render plugins correctly or fail to load AEGP plugins?

Media Encoder does not load AEGP plugins. If your effect plugin relies on a companion AEGP plugin for rendering, it will fail in Media Encoder. Issues may also be related to static global variables or elements defined in GlobalData/GlobalSetup. The AEGP functionality would need to be incorporated directly into the effect plugin, or you'd need a standalone/command-line tool alternative.

*Tags: `aegp`, `global-data`, `media-encoder`, `render-issues`*

---

### How do you store a struct in global_data?

Create a pointer to the struct in your global data type, then in GlobalSetup allocate it with new: globaldata->structPointer = new StructName(). Use new/malloc (not smart pointers) since AE does special things with the global_data pointer. Optionally delete it in GlobalSetdown, but it's not strictly necessary since GlobalSetdown only gets called when AE closes and the OS will clean up.

*Tags: `global-data`, `global-setdown`, `global-setup`, `memory-management`*

---

### What are the best practices for using global variables in AE plugins?

Global variables are generally frowned upon but widely used. Considerations: (1) std::vectors allocate system memory directly rather than through AE's RAM management, which can cause AE to exceed desired memory usage. (2) Globals are shared by ALL instances of your effect. (3) Memory is only deallocated when AE shuts down, even after global_setdown, so you can't release it through AE's suites. The recommended approach is to allocate memory and store it in a pointer on your global_data handle. For std::vector, you can write a custom allocator that uses AE memory allocation. If the data is constant and read-only, globals are more acceptable.

*Tags: `allocator`, `global-data`, `global-variables`, `memory-management`, `std-vector`*

---

### Why is sequence data different between PF_Cmd_EVENT (UI thread) and SmartPreRender (render thread)?

Sequence data is separate on the render and UI threads by design. The render threads are free to render asynchronously without having their data change from another thread. Global data IS shared between render and UI threads, though it's not instance-specific. To share per-instance data: store data in global data keyed by instance identifiers (comp item ID + layer ID + effect index on layer), protected by a mutex. Comp item IDs don't change during a session. Layer IDs don't change on reorder but may change on copy/paste. Effect index changes when moved in the stack.

*Tags: `global-data`, `mutex`, `render-thread`, `sequence-data`, `threading`, `ui-thread`*

---
