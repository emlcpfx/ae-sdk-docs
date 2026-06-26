# 32 Bit

> 2 Q&As · source: AE plugin dev community Discord

### What causes artifacts in 16/32-bit mode when a Track Matte is applied?

The root causes are typically: (1) Incorrect bit depth detection causing memory corruption - using wrong pixel sizes for memory operations. (2) Improper stride/rowbytes handling for padded buffers - when a track matte is applied, the layer gets cropped to a different size, changing the relationship between width and rowbytes. Fix bit depth detection to use rowbytes calculation and implement proper stride handling throughout your processing algorithm. The effect works in 8-bit because the rowbytes happen to align, but in 16/32-bit the padding differences cause corruption.

*Tags: `16-bit`, `32-bit`, `artifacts`, `bit-depth`, `memory-corruption`, `rowbytes`, `track-matte`*

---

### How should I handle color spaces in Premiere plugins, and can I avoid dealing with multiple color modes?

You can use basic render in 8/16-bit without handling all color spaces -- it works but is not optimized, and you won't get the 32-bit icon next to the plugin. You can choose to support only BGRA and skip VUYA. In GlobalSetup, you select which colorspaces to support. However, 32-bit float is required for color grading workflows to access values outside the 0-1 range. The SDKNoise example in the AE SDK demonstrates both BGRA and VUYA colorspace handling for Premiere.

*Tags: `32-bit`, `bgra`, `color-grading`, `colorspace`, `global-setup`, `premiere`, `vuya`*

---
