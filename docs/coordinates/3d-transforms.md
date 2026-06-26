# 3D Transforms

> 1 Q&A · source: AE plugin dev community Discord

### How can I get text layer path vertices in 3D world space when the text layer is parented to another layer?

Use AEGP_GetLayerToWorldXform() to convert from layer space to world space. However, note that PF_PathDataSuite does not return vertices in true layer space as documented. The vertices it returns are affected by 2D transforms (xy-translation, shear from parented transforms, Z Rotation) but not by z-translation, Orientation, or X/Y Rotation properties. To match the behavior of path.points() in expressions (which returns true layer space coordinates), you need to account for this discrepancy by converting PF_PathDataSuite output to match the layer space definition used in expressions before applying the layer-to-world transform.

*Tags: `3d-transforms`, `aegp`, `layer-space`, `path-data`, `text-layer`*

---
