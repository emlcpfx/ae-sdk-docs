# Blending

> 2 Q&As · source: AE plugin dev community Discord

### How does the 'Blend colors using 1.0 gamma' checkbox affect compositing?

The 'Blend colors using 1.0 gamma' checkbox only affects blending modes between layers, not the final composite of a composition onto its background color. This is why a semitransparent layer in gamma 1.0 looks different atop a black background vs. atop nothing (transparency) - the compositing onto the background doesn't respect the gamma setting. The Solid Composite effect does respect this setting, however.

*Tags: `blending`, `color-management`, `compositing`, `gamma`*

---

### Does the 'Blend colors using 1.0 gamma' checkbox affect how a composition composites onto its background color?

No. The 'Blend colors using 1.0 gamma' checkbox only affects blending modes between layers. It does not affect the final composite of a composition onto the composition's background color, because that compositing step does not respect layer blending modes. For example, if a layer's blend mode is set to 'Overlay', that blending mode is only visible when compositing against another layer below it—it does not apply when compositing the composition onto its background color.

*Tags: `blending`, `color-space`, `compositing`, `ui`*

---
