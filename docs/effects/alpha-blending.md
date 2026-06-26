# Alpha Blending

> 2 Q&As · source: AE plugin dev community Discord

### Why does a semitransparent layer in gamma 2.2 look the same whether placed over a solid black background or nothing, but in gamma 1.0 it looks different?

This is due to how gamma affects alpha blending calculations. In gamma 2.2, the color space conversion affects how transparency composites with backgrounds differently than in linear gamma (1.0). In linear color space (gamma 1.0), alpha blending math is mathematically correct and shows the true difference between compositing over black versus over nothing. In gamma 2.2 (non-linear), the gamma encoding/decoding affects the perceived blending, which can cause semitransparent layers to appear similar across different backgrounds due to the non-linear color curve affecting the alpha mathematics.

*Tags: `alpha-blending`, `color-space`, `compositing`, `gamma`*

---

### Why does a semitransparent layer appear differently in gamma 1.0 versus gamma 2.2 when composited over a black background?

In gamma 2.2, semitransparent layers appear the same whether composited over a black background or over nothing because gamma 2.2 rendering applies consistent color space conversions that make the visual difference negligible. In gamma 1.0 (linear color space), the same semitransparent layer looks completely different when placed over a black background versus transparency because linear blending treats black (0,0,0) and transparent areas fundamentally differently in the blend calculation. This is due to how alpha compositing and color space gamma correction interact—gamma 2.2 compresses the value range in a way that reduces the perceived difference, while linear gamma 1.0 preserves the full mathematical difference in how semitransparent pixels blend with the background.

*Tags: `alpha-blending`, `color-space`, `compositing`, `rendering`*

---
