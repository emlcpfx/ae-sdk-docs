# Non Param Vary

> 2 Q&As · source: AE plugin dev community Discord

### Why do keyframed parameters on adjustment layers in Premiere always return the value at the first keyframe regardless of current time?

This is caused by the PF_OutFlag_NON_PARAM_VARY flag. When this flag is set, keyframes on adjustment layers in Premiere always return the first keyframe value during checkout param, regardless of current time. This affects sliders, angles, checkboxes, and 2D points. The bug occurs in Premiere v23 and v24. Removing the NON_PARAM_VARY flag fixes the issue. In AE, this flag can be replaced using MIX_GUID during the pre_render thread. Note that apart from this bug, the flag does nothing useful in Premiere.

*Tags: `adjustment-layer`, `bug`, `checkout-param`, `keyframes`, `non-param-vary`, `premiere`*

---

### Why does memory usage spike to 6+ GB when I add PF_OutFlag_NON_PARAM_VARY to my plugin?

This is normal AE behavior - it's hoarding cached frames. PF_OutFlag_NON_PARAM_VARY tells AE the output varies independently of parameters, so AE caches each unique frame. Verify by purging AE's memory (Edit > Purge > All Memory) - if RAM consumption drops to expected levels, everything is working correctly. AE will release cached memory when it's needed for new frame renders. PF_OutFlag_WIDE_TIME_INPUT is only needed if you use a layer selector and sample it at different times. Implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup.

*Tags: `caching`, `frame-setdown`, `frame-setup`, `memory`, `non-param-vary`, `purge`*

---
