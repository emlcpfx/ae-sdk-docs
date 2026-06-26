# Preset

> 1 Q&A · source: AE plugin dev community Discord

### Why do parameter groups not stay hidden when applying an effect preset in After Effects?

When applying an effect preset, parameter visibility flags are not preserved if only the group topic is hidden. You must explicitly hide every individual parameter within each group using AEGP_SetDynamicStreamFlag on each stream reference. Hiding only the topic without hiding each parameter will cause them to appear in the UI without their group containers. Sequence data does not affect this behavior.

*Tags: `aegp`, `params`, `preset`, `ui`*

---
