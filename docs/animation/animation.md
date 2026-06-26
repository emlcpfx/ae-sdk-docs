# Animation

> 3 Q&As · source: AE plugin dev community Discord

### How can I create a progress bar or continuously animated element in the ECW (Effect Control Window)?

ECW UI elements get idle calls only when the cursor is within their borders. For updates regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows() during that event. It's a brute-force approach but gets the job done. For softer redraws when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `animation`, `custom-ui`, `ecw`, `idle-hook`, `progress-bar`, `refresh`*

---

### How can you create independent keyframe streams for individual parameters in an arbitrary data effect instead of concatenating all parameters into one keyframe stream?

Adobe deliberately designs some effects with concatenated keyframes (all parameters share one keyframe stream) and others with individual controls (each parameter has independent keyframes). The Levels effect has two versions shipped with After Effects: the standard Levels (concatenated) and Levels (Individual Controls) (independent parameters). This is a design choice rather than a technical constraint. If you need independent animations and keyframing, you can either use the individual controls variant pattern or calculate effects on-the-fly during the Render event without storing interpolation data in ArbData, similar to how Levels works without relying on ArbData for histogram display.

*Tags: `animation`, `arb-data`, `params`, `plugin-design`, `ui`*

---

### Why does the Hue/Saturation effect concatenate all parameters into a single keyframe stream instead of allowing independent parameter animation like Levels (Individual Controls)?

The Hue/Saturation effect uses USER_CHANGED_PARAM to create keys on arbitrary parameters, which results in all slider changes being keyed together rather than independently. This design choice means you cannot animate individual sliders (like Lightness) separately if another slider (like Saturation) is already animated, as all changes are stored in a single ArbData keyframe stream. The Levels (Individual Controls) variant demonstrates the alternative approach where parameters are independent, suggesting that Hue/Saturation's design prioritizes a different workflow, though the constraint could be overcome by redesigning parameter architecture to use separate animation streams.

*Tags: `animation`, `arb-data`, `params`, `plugin-design`, `ui`*

---
