# Path Data

> 2 Q&As · source: AE plugin dev community Discord

### How can I get text layer path vertices in 3D world space when the text layer is parented to another layer?

Use AEGP_GetLayerToWorldXform() to convert from layer space to world space. However, note that PF_PathDataSuite does not return vertices in true layer space as documented. The vertices it returns are affected by 2D transforms (xy-translation, shear from parented transforms, Z Rotation) but not by z-translation, Orientation, or X/Y Rotation properties. To match the behavior of path.points() in expressions (which returns true layer space coordinates), you need to account for this discrepancy by converting PF_PathDataSuite output to match the layer space definition used in expressions before applying the layer-to-world transform.

*Tags: `3d-transforms`, `aegp`, `layer-space`, `path-data`, `text-layer`*

---

### How can I access the path property of a shape layer using the AEGP API?

To access shape layer paths via AEGP API, you cannot use standard AEGP_GetNewLayerStream since shape properties are added dynamically and cannot be pre-indexed. Instead, use the Dynamic Stream Suite to recursively navigate the layer's streams. Start by getting the first stream of the shape layer, then iterate through streams by querying their names, types, and parent groups. You will eventually find the "content" group, which contains "shape" groups, and within those you can access the path data. Use suites like MaskOutlineSuite, PathQuerySuite, and PathDataSuite (which are AEGP-compatible despite their naming) to analyze the mask stream once located. This approach is necessary because the order and presence of shape properties are not guaranteed or pre-determined.

*Tags: `aegp`, `dynamic-streams`, `path-data`, `shape-layers`*

---
