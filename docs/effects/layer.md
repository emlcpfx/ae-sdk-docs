# Layer

> 2 Q&As · source: AE plugin dev community Discord

### Why does AEGP_GetLayerNumMasks return 0 even though the layer appears to have masks?

Track mattes and masks are two different things. A track matte is one layer operating with the alpha/luma of the layer above it. A mask is a vector shape drawn on a layer with the pen tool. If you're using a track matte, AEGP_GetLayerNumMasks correctly returns 0 because there are no actual masks on that layer. The mask suite only works with vector masks drawn with the pen tool, not track mattes.

*Tags: `aegp`, `layer`, `mask-suite`, `masks`, `track-matte`*

---

### Is there a direct C++ API method to replace a layer's source in After Effects?

There is no direct C++ AEGP API method for replacing a layer's source. The functionality exists in the Java API as layer.source.replace(), but in C++ it must be accessed indirectly using DoCommand with command ID 2299. However, this indirect method requires making the composition the frontmost window, which is difficult to achieve from an AEGP plugin. The feature simulates alt+dragging a new source onto a selected layer in the composition window.

*Tags: `aegp`, `api`, `layer`, `sdk`*

---
