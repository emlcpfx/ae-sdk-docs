# 32Bpc

> 2 Q&As · source: AE plugin dev community Discord

### What are the benefits of SmartFX (SmartRender) over regular Render?

SmartFX: (1) Only requests pixels needed for the specific processing area via PreRender. (2) Required for 32bpc support. (3) Allows checking out frames with prior effects applied via checkout_layer_pixels. (4) Supports MFR (Multi-Frame Rendering) better. (5) Provides more control over buffer management. Even if your effect needs the full image, SmartFX is recommended for 32bpc support and future-proofing.

*Tags: `32bpc`, `pre-render`, `render`, `smart-render`, `smartfx`*

---

### What is the optimal way to make an AE effect plugin compatible with all bit depths (8, 16, and 32 bpc)?

The simplest approach is to write wrapper/converter functions to convert to 32bpc and back, then write your processing code once in 32bpc. These converter functions can also use the multi-threaded iteration suites. The alternative is branching with switch statements for each bit depth, but that results in three separate pieces of code that only differ in type names.

*Tags: `16bpc`, `32bpc`, `8bpc`, `bit-depth`, `conversion`, `pixel-format`*

---
