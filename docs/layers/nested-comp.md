# Nested_Comp

> 2 Q&As · source: AE plugin dev community Discord

### How do you get the position stream for a nested composition layer in an AEGP plugin?

When you nest a composition (compA) into another composition (compB), the nested composition becomes a layer in compB. To get the position stream for that layer, you need to: 1) Use AEGP_GetCompLayerByIndex() on compB to get the AEGP_LayerH for the layer containing the nested composition, or receive it directly when nesting. 2) Then use AEGP_GetNewLayerStream() with AEGP_LayerStream_POSITION on that layer handle to get the position stream. Note that compositions themselves don't have positions—only layers within compositions do. If you're trying to find which comp a nested composition is being used in, you must scan the project and check each layer's source, as there is no direct API to query nesting relationships.

*Tags: `aegp`, `layer-checkout`, `nested_comp`, `reference`*

---

### Why don't animations on a nested composition appear when querying the source composition in an AEGP plugin?

When you have a composition (compA) with animated text layers that is then nested as a layer in another composition (compB) with animation applied to the nested layer itself, you will only see the text layer animations if you query compA directly. The animation applied to the nested composition layer in compB is separate and stored with that layer in compB, not in compA. To see both animations combined, you need to query the layer in compB that contains the nested composition and retrieve its position stream using AEGP_GetNewLayerStream() with AEGP_LayerStream_POSITION.

*Tags: `aegp`, `layer-checkout`, `nested_comp`, `params`*

---
