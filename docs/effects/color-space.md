# Color Space

> 4 Q&As · source: AE plugin dev community Discord

### Why does a semitransparent layer in gamma 2.2 look the same whether placed over a solid black background or nothing, but in gamma 1.0 it looks different?

This is due to how gamma affects alpha blending calculations. In gamma 2.2, the color space conversion affects how transparency composites with backgrounds differently than in linear gamma (1.0). In linear color space (gamma 1.0), alpha blending math is mathematically correct and shows the true difference between compositing over black versus over nothing. In gamma 2.2 (non-linear), the gamma encoding/decoding affects the perceived blending, which can cause semitransparent layers to appear similar across different backgrounds due to the non-linear color curve affecting the alpha mathematics.

*Tags: `alpha-blending`, `color-space`, `compositing`, `gamma`*

---

### How do you handle different color spaces when developing plugins for both After Effects and Premiere?

In After Effects, plugins typically work in ARGB colorspace. Premiere uses BGRA_8u with a special suite for 8-bit processing. For 32-bit float, you need to write your own conversion function. You can use the Iterate8Suite1 (or SuiteV2 since 2022) for ARGB, but Premiere requires a special BGRA suite. You don't have to support all color spaces—you can optimize for BGRA only and skip YUVA if needed. Basic rendering in 8/16 bit works without optimization, though you won't get the 32-bit icon next to the plugin. Check the SDK noise example for reference implementations of BGRA and YUVA colorspace handling.

*Tags: `argb`, `bgra`, `color-space`, `optimization`, `premiere`, `render-loop`*

---

### Why does a semitransparent layer appear differently in gamma 1.0 versus gamma 2.2 when composited over a black background?

In gamma 2.2, semitransparent layers appear the same whether composited over a black background or over nothing because gamma 2.2 rendering applies consistent color space conversions that make the visual difference negligible. In gamma 1.0 (linear color space), the same semitransparent layer looks completely different when placed over a black background versus transparency because linear blending treats black (0,0,0) and transparent areas fundamentally differently in the blend calculation. This is due to how alpha compositing and color space gamma correction interact—gamma 2.2 compresses the value range in a way that reduces the perceived difference, while linear gamma 1.0 preserves the full mathematical difference in how semitransparent pixels blend with the background.

*Tags: `alpha-blending`, `color-space`, `compositing`, `rendering`*

---

### Does the 'Blend colors using 1.0 gamma' checkbox affect how a composition composites onto its background color?

No. The 'Blend colors using 1.0 gamma' checkbox only affects blending modes between layers. It does not affect the final composite of a composition onto the composition's background color, because that compositing step does not respect layer blending modes. For example, if a layer's blend mode is set to 'Overlay', that blending mode is only visible when compositing against another layer below it—it does not apply when compositing the composition onto its background color.

*Tags: `blending`, `color-space`, `compositing`, `ui`*

---
