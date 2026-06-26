# Dynamic Streams

> 2 Q&As · source: AE plugin dev community Discord

### How can I checkout paths or masks from a different layer in the AE SDK?

You can fetch any mask using AEGP_MaskSuite6 and read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape param, you need to traverse the layer's dynamic streams. However, there are limitations: mask vertices are delivered unmodified (not expanded by the 'expand' param), shapes won't arrive 'rounded', and parametric shapes like 'star' have no shape param to read.

*Tags: `aegp`, `dynamic-streams`, `masks`, `paths`, `shape-layer`, `vertices`*

---

### How can I access the path property of a shape layer using the AEGP API?

To access shape layer paths via AEGP API, you cannot use standard AEGP_GetNewLayerStream since shape properties are added dynamically and cannot be pre-indexed. Instead, use the Dynamic Stream Suite to recursively navigate the layer's streams. Start by getting the first stream of the shape layer, then iterate through streams by querying their names, types, and parent groups. You will eventually find the "content" group, which contains "shape" groups, and within those you can access the path data. Use suites like MaskOutlineSuite, PathQuerySuite, and PathDataSuite (which are AEGP-compatible despite their naming) to analyze the mask stream once located. This approach is necessary because the order and presence of shape properties are not guaranteed or pre-determined.

*Tags: `aegp`, `dynamic-streams`, `path-data`, `shape-layers`*

---
