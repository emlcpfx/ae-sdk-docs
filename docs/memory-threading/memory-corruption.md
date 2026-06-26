# Memory Corruption

> 1 Q&A · source: AE plugin dev community Discord

### What causes artifacts in 16/32-bit mode when a Track Matte is applied?

The root causes are typically: (1) Incorrect bit depth detection causing memory corruption - using wrong pixel sizes for memory operations. (2) Improper stride/rowbytes handling for padded buffers - when a track matte is applied, the layer gets cropped to a different size, changing the relationship between width and rowbytes. Fix bit depth detection to use rowbytes calculation and implement proper stride handling throughout your processing algorithm. The effect works in 8-bit because the rowbytes happen to align, but in 16/32-bit the padding differences cause corruption.

*Tags: `16-bit`, `32-bit`, `artifacts`, `bit-depth`, `memory-corruption`, `rowbytes`, `track-matte`*

---
