# Bug

> 3 Q&As · source: AE plugin dev community Discord

### How does AE's Custom UI / arb params bug in AE 2025 manifest?

In AE 2025 (25.0.1), custom UI and arb params display glitches: one arb param's UI can corrupt other arb UIs, causing flashes or scrambled/duplicated content. The likely culprit is NewImageFromBuffer in DrawBot - every call may be setting pixels of all retained DRAWBOT_ImageRef within the same context. Adobe acknowledged fixing something in arb handling, but it may have introduced this unintended consequence. The issue affects multiple third-party plugins including Trapcode.

*Tags: `ae-2025`, `arb-data`, `bug`, `custom-ui`, `drawbot`, `regression`*

---

### Why do keyframed parameters on adjustment layers in Premiere always return the value at the first keyframe regardless of current time?

This is caused by the PF_OutFlag_NON_PARAM_VARY flag. When this flag is set, keyframes on adjustment layers in Premiere always return the first keyframe value during checkout param, regardless of current time. This affects sliders, angles, checkboxes, and 2D points. The bug occurs in Premiere v23 and v24. Removing the NON_PARAM_VARY flag fixes the issue. In AE, this flag can be replaced using MIX_GUID during the pre_render thread. Note that apart from this bug, the flag does nothing useful in Premiere.

*Tags: `adjustment-layer`, `bug`, `checkout-param`, `keyframes`, `non-param-vary`, `premiere`*

---

### Is AEGP_RenderAndCheckoutLayerFrame_Async safe to use for rendering multiple frames?

There is a known bug with the async version of RenderAndCheckout related to memory releasing. The issue was discussed between several developers and the AE team. All developers reverted back to the synchronous version, doing their best to render one frame at a time on each idle cycle to avoid making the UI laggy. The frame receipt cannot be properly checked in when using the async version.

*Tags: `async-render`, `bug`, `frame-checkout`, `memory-leak`, `render-suite`*

---
