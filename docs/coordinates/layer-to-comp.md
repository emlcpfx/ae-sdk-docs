# Layer To Comp

> 1 Q&A · source: AE plugin dev community Discord

### What does AEGP_GetLayerToWorldXform actually represent?

AEGP_GetLayerToWorldXform translates XY coordinates from the layer origin to XYZ in composition space. 'World' means composition space. You can use the returned matrix for simple layer-to-comp coordinate conversion, or use it to construct a full 'render view' matrix from the camera's point of view.

*Tags: `aegp`, `coordinate-space`, `layer-to-comp`, `matrix`, `transform`*

---
