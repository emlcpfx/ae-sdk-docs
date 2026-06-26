# Frame Checkout

> 2 Q&As · source: AE plugin dev community Discord

### How do you check out a frame with prior effects applied (not the raw layer)?

Use checkout_layer_pixels (from PF_SmartRenderCallbacks in SmartRender) instead of PF_CHECKOUT_PARAM. PF_CHECKOUT_PARAM always returns the layer before all effects were applied. checkout_layer_pixels returns the frame with all prior effects applied. You must first checkout the layer in PreRender, then checkout_layer_pixels in SmartRender.

*Tags: `checkout-layer`, `effect-stack`, `frame-checkout`, `pre-render`, `smart-render`*

---

### Is AEGP_RenderAndCheckoutLayerFrame_Async safe to use for rendering multiple frames?

There is a known bug with the async version of RenderAndCheckout related to memory releasing. The issue was discussed between several developers and the AE team. All developers reverted back to the synchronous version, doing their best to render one frame at a time on each idle cycle to avoid making the UI laggy. The frame receipt cannot be properly checked in when using the async version.

*Tags: `async-render`, `bug`, `frame-checkout`, `memory-leak`, `render-suite`*

---
