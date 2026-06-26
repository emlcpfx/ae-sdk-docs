# Api Changes

> 1 Q&A · source: AE plugin dev community Discord

### What parameter should be passed to AEGP_StartUndoGroup to avoid crashes in After Effects 2025 beta?

As of After Effects 2025 beta, passing an empty string "" to AEGP_StartUndoGroup is correct, while passing null causes a crash. In previous versions, null did not cause adverse effects, but this behavior changed in the beta. Using an empty string works as expected and does not add an entry to the Undo stack.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `api-changes`, `backwards-compatibility`, `undo`*

---
