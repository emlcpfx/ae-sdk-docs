# Eric CPFX

**1 contributions** to AE SDK community knowledge.

Top topics: `track-matte`, `bit-depth`, `16-bit`, `32-bit`, `rowbytes`, `memory-corruption`, `artifacts`

---

## What causes artifacts in 16/32-bit mode when a Track Matte is applied?

The root causes are typically: (1) Incorrect bit depth detection causing memory corruption - using wrong pixel sizes for memory operations. (2) Improper stride/rowbytes handling for padded buffers - when a track matte is applied, the layer gets cropped to a different size, changing the relationship between width and rowbytes. Fix bit depth detection to use rowbytes calculation and implement proper stride handling throughout your processing algorithm. The effect works in 8-bit because the rowbytes happen to align, but in 16/32-bit the padding differences cause corruption.

*Source: adobe-plugin-devs · 2025-10-15 · Tags: `track-matte`, `bit-depth`, `16-bit`, `32-bit`, `rowbytes`, `memory-corruption`, `artifacts` · [View in Q&A](../qa/track-matte/)*

---
