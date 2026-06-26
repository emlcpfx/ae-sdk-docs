# 64 Bit

> 3 Q&As · source: AE plugin dev community Discord

### Do PF_LOCK_HANDLE / host_lock_handle functions actually do anything?

According to multiple Adobe engineers, PF_LOCK_HANDLE and host_lock_handle have been no-op dummy functions since AE CS6 (2011) when the codebase moved to 64-bit. Locking/unlocking handles is redundant in 64-bit address space. However, they still provide a safe way to dereference handles (effectively double pointers) and the SDK samples still call them. The DH macro can be used for direct dereferencing instead.

*Tags: `64-bit`, `cs6`, `handles`, `lock-unlock`, `memory-management`*

---

### Why do lock/unlock functions still exist in the After Effects SDK if they weren't ported to 64-bit?

According to Tobias Fleischer, the lock/unlock functions were not ported when the AE codebase moved to 64-bit in CS6 (2011), yet they still exist in the current SDK and sample code calls them. While rowbyte notes these functions are somewhat redundant in a 64-bit address space, they still provide a safe way to dereference handles (which are effectively double pointers in 64-bit) without confusion. The SDK provides a DH macro as an alternative for direct dereferencing.

*Tags: `64-bit`, `aegp`, `memory`, `reference`, `sdk`*

---

### Can Visual C++ Express be used to compile 64-bit After Effects plugins?

Yes, Visual C++ Express can be used to compile 64-bit plugins by downloading the Windows SDK and using it as the compiler. A patch is available that enables 64-bit compilation support in VC2008 Express with Windows 7 SDK.

*Tags: `64-bit`, `build`, `sdk`, `windows`*

---
