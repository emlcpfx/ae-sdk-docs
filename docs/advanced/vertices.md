# Vertices

> 1 Q&A · source: AE plugin dev community Discord

### How can I checkout paths or masks from a different layer in the AE SDK?

You can fetch any mask using AEGP_MaskSuite6 and read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape param, you need to traverse the layer's dynamic streams. However, there are limitations: mask vertices are delivered unmodified (not expanded by the 'expand' param), shapes won't arrive 'rounded', and parametric shapes like 'star' have no shape param to read.

*Tags: `aegp`, `dynamic-streams`, `masks`, `paths`, `shape-layer`, `vertices`*

---
