# Cs6

> 2 Q&As · source: AE plugin dev community Discord

### Do PF_LOCK_HANDLE / host_lock_handle functions actually do anything?

According to multiple Adobe engineers, PF_LOCK_HANDLE and host_lock_handle have been no-op dummy functions since AE CS6 (2011) when the codebase moved to 64-bit. Locking/unlocking handles is redundant in 64-bit address space. However, they still provide a safe way to dereference handles (effectively double pointers) and the SDK samples still call them. The DH macro can be used for direct dereferencing instead.

*Tags: `64-bit`, `cs6`, `handles`, `lock-unlock`, `memory-management`*

---

### Can After Effects CS6 be installed and run on Windows 11?

Yes, AE CS6 runs fine on Windows 11. If you have activation issues with the license key, you may need to run it in compatibility mode. Windows 10 is a fallback option if Windows 11 doesn't cooperate with activation.

*Tags: `compatibility`, `cs6`, `installation`, `legacy`, `windows-11`*

---
