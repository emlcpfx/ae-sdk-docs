# Plugin Api

> 1 Q&A · source: AE plugin dev community Discord

### What does host_lock_handle do and how should memory be allocated in After Effects plugins?

host_lock_handle tells After Effects that a memory chunk is in use and should not be moved by the operating system. In After Effects plugins, you should use the memory suites API (host_create_handle + host_lock_handle) rather than malloc/new, to allow AE to make RAM allocation decisions at the application scope. The pattern host_create_handle + host_lock_handle is functionally equivalent to malloc. For handles you allocate, you must lock, unlock, and dispose of them. Some handles are allocated and locked for you by AE (like those from ExecuteScript), while others like global_data and sequence_data are managed by AE automatically when the plugin is called.

*Tags: `aegp`, `memory`, `plugin-api`*

---
