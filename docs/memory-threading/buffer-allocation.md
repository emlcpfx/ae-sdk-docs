# Buffer Allocation

> 1 Q&A · source: AE plugin dev community Discord

### How do you create a proper copy of an input buffer in an After Effects plugin instead of a reference?

Use AEGP_WorldSuite3->AEGP_New() to allocate a new world buffer, then call AEGP_FillOutPFEffectWorld() to create a PF_EffectWorld wrapper for the allocated AEGP_WorldH. This prevents the common mistake of assigning the input layer directly (which only creates a reference) and corrupting the original input data when iterating operations.

*Tags: `aegp`, `buffer-allocation`, `memory`*

---
