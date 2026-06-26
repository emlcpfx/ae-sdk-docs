# 64Bit

> 1 Q&A · source: AE plugin dev community Discord

### Why do lock/unlock functions still exist in the After Effects SDK if they were not ported to 64-bit?

The lock/unlock functions were not ported when the AE codebase moved to 64-bit in CS6 (2011), but they still exist in the current SDK and sample code. However, locking/unlocking handles is somewhat redundant in 64-bit address space. They do provide a safe way to dereference handles (which are now effectively double pointers) without confusion, and the SDK provides a DH macro as an alternative for direct dereferencing.

*Tags: `64bit`, `aegp`, `debugging`, `memory`*

---
