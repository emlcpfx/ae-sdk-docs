# Limitation

> 2 Q&As · source: AE plugin dev community Discord

### Can the AE SDK provide post-processed shape path vertices (after Offset Path, Zig Zag, Wiggle modifiers)?

AE doesn't provide post-processed paths. You can only read the raw parameter streams and then try to emulate the processed path yourself. There is no way to get the final computed path after shape modifiers like Offset, Zig Zag, or Wiggle have been applied.

*Tags: `aegp`, `limitation`, `path-vertices`, `shape-layer`, `shape-modifiers`*

---

### Can After Effects plugins currently access the Param source type for effects, masks, or sources through the dropdown list?

As of February 2023, there is no direct access to the Param source type effect/mask/source dropdown list in the After Effects plugin API. This limitation prevents plugins from programmatically controlling or reading which layer or effect is selected as a source parameter.

*Tags: `aegp`, `limitation`, `params`*

---
