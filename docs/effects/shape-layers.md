# Shape Layers

> 2 Q&As · source: AE plugin dev community Discord

### How can I access post-processed shape path vertices (with effects like offset, zig zag, or wiggle applied) using the After Effects SDK?

Unfortunately, After Effects does not provide direct access to post-processed paths through the SDK. You can only read the parameter streams directly and then manually emulate the processed path behavior by implementing the effects yourself.

*Tags: `aegp`, `params`, `sdk`, `shape-layers`*

---

### How can I access the path property of a shape layer using the AEGP API?

To access shape layer paths via AEGP API, you cannot use standard AEGP_GetNewLayerStream since shape properties are added dynamically and cannot be pre-indexed. Instead, use the Dynamic Stream Suite to recursively navigate the layer's streams. Start by getting the first stream of the shape layer, then iterate through streams by querying their names, types, and parent groups. You will eventually find the "content" group, which contains "shape" groups, and within those you can access the path data. Use suites like MaskOutlineSuite, PathQuerySuite, and PathDataSuite (which are AEGP-compatible despite their naming) to analyze the mask stream once located. This approach is necessary because the order and presence of shape properties are not guaranteed or pre-determined.

*Tags: `aegp`, `dynamic-streams`, `path-data`, `shape-layers`*

---
