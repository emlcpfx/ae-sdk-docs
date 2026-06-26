# Plugin Design

> 3 Q&As · source: AE plugin dev community Discord

### How can you create independent keyframe streams for individual parameters in an arbitrary data effect instead of concatenating all parameters into one keyframe stream?

Adobe deliberately designs some effects with concatenated keyframes (all parameters share one keyframe stream) and others with individual controls (each parameter has independent keyframes). The Levels effect has two versions shipped with After Effects: the standard Levels (concatenated) and Levels (Individual Controls) (independent parameters). This is a design choice rather than a technical constraint. If you need independent animations and keyframing, you can either use the individual controls variant pattern or calculate effects on-the-fly during the Render event without storing interpolation data in ArbData, similar to how Levels works without relying on ArbData for histogram display.

*Tags: `animation`, `arb-data`, `params`, `plugin-design`, `ui`*

---

### Why does the Hue/Saturation effect concatenate all parameters into a single keyframe stream instead of allowing independent parameter animation like Levels (Individual Controls)?

The Hue/Saturation effect uses USER_CHANGED_PARAM to create keys on arbitrary parameters, which results in all slider changes being keyed together rather than independently. This design choice means you cannot animate individual sliders (like Lightness) separately if another slider (like Saturation) is already animated, as all changes are stored in a single ArbData keyframe stream. The Levels (Individual Controls) variant demonstrates the alternative approach where parameters are independent, suggesting that Hue/Saturation's design prioritizes a different workflow, though the constraint could be overcome by redesigning parameter architecture to use separate animation streams.

*Tags: `animation`, `arb-data`, `params`, `plugin-design`, `ui`*

---

### How do modular plugins like Plexus work, where multiple plugins are stacked to produce a final result?

Modular plugins typically don't render anything themselves. Instead, the individual 'module' effects allow users to input parameters, while a main/master effect reads the parameter values from all the module effects and performs the actual rendering based on those combined parameter values. This is simpler and more elegant than trying to maintain off-screen buffers or create custom channels.

*Tags: `architecture`, `params`, `plugin-design`, `reference`*

---
