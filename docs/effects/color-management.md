# Color Management

> 1 Q&A · source: AE plugin dev community Discord

### How does the 'Blend colors using 1.0 gamma' checkbox affect compositing?

The 'Blend colors using 1.0 gamma' checkbox only affects blending modes between layers, not the final composite of a composition onto its background color. This is why a semitransparent layer in gamma 1.0 looks different atop a black background vs. atop nothing (transparency) - the compositing onto the background doesn't respect the gamma setting. The Solid Composite effect does respect this setting, however.

*Tags: `blending`, `color-management`, `compositing`, `gamma`*

---
