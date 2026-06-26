# Mask Suite

> 1 Q&A · source: AE plugin dev community Discord

### Why does AEGP_GetLayerNumMasks return 0 even though the layer appears to have masks?

Track mattes and masks are two different things. A track matte is one layer operating with the alpha/luma of the layer above it. A mask is a vector shape drawn on a layer with the pen tool. If you're using a track matte, AEGP_GetLayerNumMasks correctly returns 0 because there are no actual masks on that layer. The mask suite only works with vector masks drawn with the pen tool, not track mattes.

*Tags: `aegp`, `layer`, `mask-suite`, `masks`, `track-matte`*

---
