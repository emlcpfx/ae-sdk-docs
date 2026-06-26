# Effect Stack

> 1 Q&A · source: AE plugin dev community Discord

### How do you check out a frame with prior effects applied (not the raw layer)?

Use checkout_layer_pixels (from PF_SmartRenderCallbacks in SmartRender) instead of PF_CHECKOUT_PARAM. PF_CHECKOUT_PARAM always returns the layer before all effects were applied. checkout_layer_pixels returns the frame with all prior effects applied. You must first checkout the layer in PreRender, then checkout_layer_pixels in SmartRender.

*Tags: `checkout-layer`, `effect-stack`, `frame-checkout`, `pre-render`, `smart-render`*

---
