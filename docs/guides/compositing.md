# Compositing

> 4 Q&As · source: AE plugin dev community Discord

### How does the 'Blend colors using 1.0 gamma' checkbox affect compositing?

The 'Blend colors using 1.0 gamma' checkbox only affects blending modes between layers, not the final composite of a composition onto its background color. This is why a semitransparent layer in gamma 1.0 looks different atop a black background vs. atop nothing (transparency) - the compositing onto the background doesn't respect the gamma setting. The Solid Composite effect does respect this setting, however.

*Tags: `blending`, `color-management`, `compositing`, `gamma`*

---

### Why does a semitransparent layer in gamma 2.2 look the same whether placed over a solid black background or nothing, but in gamma 1.0 it looks different?

This is due to how gamma affects alpha blending calculations. In gamma 2.2, the color space conversion affects how transparency composites with backgrounds differently than in linear gamma (1.0). In linear color space (gamma 1.0), alpha blending math is mathematically correct and shows the true difference between compositing over black versus over nothing. In gamma 2.2 (non-linear), the gamma encoding/decoding affects the perceived blending, which can cause semitransparent layers to appear similar across different backgrounds due to the non-linear color curve affecting the alpha mathematics.

*Tags: `alpha-blending`, `color-space`, `compositing`, `gamma`*

---

### Why does a semitransparent layer appear differently in gamma 1.0 versus gamma 2.2 when composited over a black background?

In gamma 2.2, semitransparent layers appear the same whether composited over a black background or over nothing because gamma 2.2 rendering applies consistent color space conversions that make the visual difference negligible. In gamma 1.0 (linear color space), the same semitransparent layer looks completely different when placed over a black background versus transparency because linear blending treats black (0,0,0) and transparent areas fundamentally differently in the blend calculation. This is due to how alpha compositing and color space gamma correction interact—gamma 2.2 compresses the value range in a way that reduces the perceived difference, while linear gamma 1.0 preserves the full mathematical difference in how semitransparent pixels blend with the background.

*Tags: `alpha-blending`, `color-space`, `compositing`, `rendering`*

---

### Does the 'Blend colors using 1.0 gamma' checkbox affect how a composition composites onto its background color?

No. The 'Blend colors using 1.0 gamma' checkbox only affects blending modes between layers. It does not affect the final composite of a composition onto the composition's background color, because that compositing step does not respect layer blending modes. For example, if a layer's blend mode is set to 'Overlay', that blending mode is only visible when compositing against another layer below it—it does not apply when compositing the composition onto its background color.

*Tags: `blending`, `color-space`, `compositing`, `ui`*

---
