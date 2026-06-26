# Blend Mode

> 2 Q&As · source: AE plugin dev community Discord

### How do you get the original foreground layer pixels when implementing a custom blend mode effect on an adjustment layer?

To get the foreground layer's original pixels, use the layer parameter checkout mechanism. First, set up a layer parameter with "self" as the default. Then use PF_CHECKOUT_PARAM() to checkout the layer at the desired time, and access the fetched pixels at checkout.u.ld. Alternatively, you can use AEGP_GetLayerSourceItem() to get the layer's source item, AEGP_NewFromItem() to create render options, AEGP_RenderAndCheckoutFrame() to get a receipt, and AEGP_GetReceiptWorld() to fetch the pixels. The layer parameter checkout method fetches pre-masks and pre-effects pixels, which is what you need for blend modes.

```cpp
PF_ParamDef checkout;
ERR(PF_CHECKOUT_PARAM( in_data,
index_of_layer_param,
in_data->current_time,
in_data->time_step,
in_data->time_scale,
&checkout));
// Fetched pixels are at checkout.u.ld
```

*Tags: `aegp`, `blend-mode`, `layer-checkout`, `params`, `pipl`*

---

### What is the difference between the old checkout mechanism and checkout_layer_pixels() when fetching layer pixels?

The old checkout mechanism (PF_CHECKOUT_PARAM) fetches the original layer pixels pre-masks and pre-effects, making it suitable for implementing custom blend modes. The newer checkout_layer_pixels() function fetches pixels post-masks and post-effects. For blend mode implementations where you need the original foreground pixels, use the old PF_CHECKOUT_PARAM mechanism instead.

*Tags: `blend-mode`, `layer-checkout`, `params`, `pipl`*

---
