# Boilerplate

> 2 Q&As · source: AE plugin dev community Discord

### What is the minimal code needed for SmartFX PreRender and SmartRender?

In PreRender: checkout the layer with channel_mask set to PF_ChannelMask_ARGB, then union the result rects. In SmartRender: checkout layer pixels and output, check the bit depth via extra->input->bitdepth (8, 16, or 32) and process accordingly. If you don't need to process anything, use PF_COPY(inputP, outputP, NULL, NULL) which is bit-depth independent. Always check in params with ERR2 regardless of error state. Don't forget to set the correct flags in global setup.

```cpp
// PreRender
PF_RenderRequest req = extra->input->output_request;
PF_CheckoutResult in_result;
req.channel_mask = PF_ChannelMask_ARGB;
ERR(extra->cb->checkout_layer(in_data->effect_ref, EFFECT_INPUT, EFFECT_INPUT, &req,
    in_data->current_time, in_data->time_step, in_data->time_scale, &in_result));
UnionLRect(&in_result.result_rect, &extra->output->result_rect);
UnionLRect(&in_result.max_result_rect, &extra->output->max_result_rect);

// SmartRender
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, EFFECT_INPUT, &input_worldP));
ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));
switch (extra->input->bitdepth) {
    case 8: /* process 8 bit */ break;
    case 16: /* process 16 bit */ break;
    case 32: /* process 32 bit */ break;
}
ERR2(PF_CHECKIN_PARAM(in_data, &paramWhatever));
```

*Tags: `bit-depth`, `boilerplate`, `minimal`, `pre-render`, `smart-render`, `smartfx`*

---

### Are there any good C++ wrapper libraries for the After Effects Filter GUI API?

No well-known general-purpose wrapper exists. Most wrappers that have been built end up being very complex themselves if you try to handle all use cases -- you end up with a non-standard system that is almost as complex as the original, just with more C++ features. Copy-pasting C chunks from the SDK examples works well in practice. That said, even a C wrapper with automatic memory management and state stored in structs would be an improvement over the current state machine. There is also a Rust binding project (https://github.com/virtualritz/after-effects/) that significantly reduces boilerplate compared to C/C++.

*Tags: `boilerplate`, `cpp`, `gui-api`, `rust`, `sdk`, `wrapper`*

---
