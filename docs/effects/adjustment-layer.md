# Adjustment Layer

> 2 Q&As · source: AE plugin dev community Discord

### Why do keyframed parameters on adjustment layers in Premiere always return the value at the first keyframe regardless of current time?

This is caused by the PF_OutFlag_NON_PARAM_VARY flag. When this flag is set, keyframes on adjustment layers in Premiere always return the first keyframe value during checkout param, regardless of current time. This affects sliders, angles, checkboxes, and 2D points. The bug occurs in Premiere v23 and v24. Removing the NON_PARAM_VARY flag fixes the issue. In AE, this flag can be replaced using MIX_GUID during the pre_render thread. Note that apart from this bug, the flag does nothing useful in Premiere.

*Tags: `adjustment-layer`, `bug`, `checkout-param`, `keyframes`, `non-param-vary`, `premiere`*

---

### How can a plugin render output that extends beyond the bounds of its adjustment layer?

You need to modify the output_rect to be larger than the layer's bounds. By setting the output rectangle in your render function to extend beyond the adjustment layer's dimensions, you can render content (like text) that appears outside the layer's original boundaries while still being contained within the composition.

*Tags: `adjustment-layer`, `output-rect`, `render-loop`*

---
