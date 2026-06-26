# Shape Layer

> 4 Q&As · source: AE plugin dev community Discord

### How can I checkout paths or masks from a different layer in the AE SDK?

You can fetch any mask using AEGP_MaskSuite6 and read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape param, you need to traverse the layer's dynamic streams. However, there are limitations: mask vertices are delivered unmodified (not expanded by the 'expand' param), shapes won't arrive 'rounded', and parametric shapes like 'star' have no shape param to read.

*Tags: `aegp`, `dynamic-streams`, `masks`, `paths`, `shape-layer`, `vertices`*

---

### Can the AE SDK provide post-processed shape path vertices (after Offset Path, Zig Zag, Wiggle modifiers)?

AE doesn't provide post-processed paths. You can only read the raw parameter streams and then try to emulate the processed path yourself. There is no way to get the final computed path after shape modifiers like Offset, Zig Zag, or Wiggle have been applied.

*Tags: `aegp`, `limitation`, `path-vertices`, `shape-layer`, `shape-modifiers`*

---

### How do you retrieve shape layer content parameters (like Polystar, ZigZag, Repeater) in AEGP?

Shape layer contents are not accessed the same way as effects. For path data, use PF_PathDataSuite1 to retrieve path vertices. For rendered pixels, you cannot directly read the source since shape layers don't have one. As an alternative, you can get the shape and render it yourself, or duplicate the composition, delete everything except the shape layer, and use AEGP_GetReceiptWorld() to render that new composition. The specific method was discussed in detail in an Adobe forum thread.

*Tags: `aegp`, `params`, `reference`, `shape-layer`*

---

### What is the correct AEGP API method to access shape layer contents versus effects?

To get layer effects, use AEGP_GetLayerEffectByIndex. However, shape layer contents (like Polystar, ZigZag, Repeater) are not accessed through the stream suite or effect methods. Instead, use PF_PathDataSuite1 for path geometry, or employ workarounds such as rendering the shape yourself or using AEGP_GetReceiptWorld() on a duplicate composition containing only the shape layer.

*Tags: `aegp`, `debugging`, `params`, `shape-layer`*

---
