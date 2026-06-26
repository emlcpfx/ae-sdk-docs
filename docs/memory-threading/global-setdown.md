# Global Setdown

> 1 Q&A · source: AE plugin dev community Discord

### How do you store a struct in global_data?

Create a pointer to the struct in your global data type, then in GlobalSetup allocate it with new: globaldata->structPointer = new StructName(). Use new/malloc (not smart pointers) since AE does special things with the global_data pointer. Optionally delete it in GlobalSetdown, but it's not strictly necessary since GlobalSetdown only gets called when AE closes and the OS will clean up.

*Tags: `global-data`, `global-setdown`, `global-setup`, `memory-management`*

---
