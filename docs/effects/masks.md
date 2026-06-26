# Masks

> 5 Q&As · source: AE plugin dev community Discord

### How can I checkout paths or masks from a different layer in the AE SDK?

You can fetch any mask using AEGP_MaskSuite6 and read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape param, you need to traverse the layer's dynamic streams. However, there are limitations: mask vertices are delivered unmodified (not expanded by the 'expand' param), shapes won't arrive 'rounded', and parametric shapes like 'star' have no shape param to read.

*Tags: `aegp`, `dynamic-streams`, `masks`, `paths`, `shape-layer`, `vertices`*

---

### Why does AEGP_GetLayerNumMasks return 0 even though the layer appears to have masks?

Track mattes and masks are two different things. A track matte is one layer operating with the alpha/luma of the layer above it. A mask is a vector shape drawn on a layer with the pen tool. If you're using a track matte, AEGP_GetLayerNumMasks correctly returns 0 because there are no actual masks on that layer. The mask suite only works with vector masks drawn with the pen tool, not track mattes.

*Tags: `aegp`, `layer`, `mask-suite`, `masks`, `track-matte`*

---

### Why does the render not refresh automatically when drawing/deleting an open path/mask with PF_Path suites?

You need to set the flag PF_OutFlag2_DEPENDS_ON_UNREFERENCED_MASKS. This tells After Effects to invalidate your output when a mask is modified that doesn't appear to be referenced by your effect. Closed masks automatically refresh because they alter the input layer pixels before they reach your plugin (AE must call the plugin again since input changed). Open masks don't alter anything normally, so AE thinks there's no reason to re-render.

*Tags: `masks`, `open-path`, `outflags`, `pf-path`, `render-invalidation`*

---

### How can you apply masks to GPU-rendered output in After Effects when the GPU API only provides masked/cropped buffers?

Eric encountered this limitation while developing the Wrap It 3D camera projection plugin. After Effects' GPU rendering API (checkout_layer_pixels) only returns post-mask-applied, cropped buffers with no GPU equivalent of PF_CHECKOUT_PARAM to access the raw unmasked source. Attempted workarounds included using PF_OutFlag2_REVEALS_ZERO_ALPHA with expanded PreRender requests, but this didn't provide the full unmasked layer to GPU. A manual CPU→GPU upload of the unmasked source defeats GPU acceleration benefits. The core limitation is that AE's native GPU world has no mechanism to provide raw unmasked source data on GPU. As a workaround, users may need to use track mattes in GPU mode, or consider building a custom GPU context (OpenGL, Vulkan, or WebGPU) outside of After Effects' native GPU pipeline to maintain full control over the source data.

*Tags: `aegp`, `gpu`, `layer-checkout`, `masks`, `render-loop`*

---

### How can I checkout masks and paths from a different layer using the After Effects SDK?

You can fetch any mask using AEGP_MaskSuite6, then read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape parameter, you need to traverse the layer's dynamic streams. However, be aware that mask and shape vertex data is delivered unmodified—masks won't include the "expand" parameter modification, and shapes won't show rounding if applied by the user. Additionally, some shapes are parametric (like star shapes) and have no shape parameter to read.

*Tags: `aegp`, `layer-checkout`, `masks`, `sdk`, `sequence-data`*

---
