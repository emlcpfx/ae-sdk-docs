# Prior Effects

> 1 Q&A · source: AE plugin dev community Discord

### Is checking out the current layer with prior effects applied possible in Premiere?

In AE, you can use checkout_layer_pixels in SmartRender to get the layer with prior effects applied. However, this does not work in Premiere. In Premiere, there is no straightforward way to check out the current layer with prior effects applied - PF_CHECKOUT_PARAM always returns the pre-effects layer. A potential hacky workaround might involve calling Premiere's JS scripting API to render, but this is not a clean solution.

*Tags: `checkout-layer`, `limitations`, `premiere`, `prior-effects`, `smart-render`*

---
