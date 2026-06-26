# Text Layer

> 3 Q&As · source: AE plugin dev community Discord

### How can I get text layer path vertices in 3D world space when the text layer is parented to another layer?

Use AEGP_GetLayerToWorldXform() to convert from layer space to world space. However, note that PF_PathDataSuite does not return vertices in true layer space as documented. The vertices it returns are affected by 2D transforms (xy-translation, shear from parented transforms, Z Rotation) but not by z-translation, Orientation, or X/Y Rotation properties. To match the behavior of path.points() in expressions (which returns true layer space coordinates), you need to account for this discrepancy by converting PF_PathDataSuite output to match the layer space definition used in expressions before applying the layer-to-world transform.

*Tags: `3d-transforms`, `aegp`, `layer-space`, `path-data`, `text-layer`*

---

### What is a reliable method to pass data from an effect plugin to a text layer?

An AEGP plugin using an idle hook is the best approach. Each frame, push string results into a queue, and periodically process the queue in the idle hook. This requires two binaries (AEGP plugin and FX plugin) communicating through a C interface. Alternatively, you can encode data into dead pixels in your render and use sampleImage expressions to retrieve those pixels and set source text, though this has caveats with render format reliability.

*Tags: `aegp`, `idle-hook`, `plugin-communication`, `render-loop`, `text-layer`*

---

### Why does calling PF_ForceRerender() on a text layer result in 'layer does not have a source' error?

Text layers and shape layers do not have a source, unlike solids. Any source-related operation performed on a text layer will generate this error. As a workaround, instead of using PF_ForceRerender(), you can add a dummy parameter to the effect and call AEGP_SetStreamValue() to trigger rendering of all layers with the effect.

*Tags: `aegp`, `debugging`, `params`, `text-layer`*

---
