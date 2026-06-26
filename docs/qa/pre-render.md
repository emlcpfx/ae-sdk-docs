# Q&A: pre-render

**6 entries** tagged with `pre-render`.

---

## How do you check out a frame with prior effects applied (not the raw layer)?

Use checkout_layer_pixels (from PF_SmartRenderCallbacks in SmartRender) instead of PF_CHECKOUT_PARAM. PF_CHECKOUT_PARAM always returns the layer before all effects were applied. checkout_layer_pixels returns the frame with all prior effects applied. You must first checkout the layer in PreRender, then checkout_layer_pixels in SmartRender.

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2024-02-02 · Tags: `smart-render`, `checkout-layer`, `pre-render`, `effect-stack`, `frame-checkout`*

---

## How do you use GuidMixInPtr to force re-rendering of cached frames?

Set the flag PF_OutFlag2_I_MIX_GUID_DEPENDENCIES, then in PreRender call extraP->cb->GuidMixInPtr with changed data. When the mixed-in data changes, AE invalidates the cache for that frame. Check for null pointers on extraP, cb, and GuidMixInPtr. Also check against 0xabababababababab (uninitialized memory pattern). You can mix in timestamps, parameter values, or any data that should trigger re-rendering.

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data, PF_PreRenderExtra* extraP) {
    if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION &&
        extraP && extraP->cb && extraP->cb->GuidMixInPtr &&
        (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab) {
        int r = time(0) + in_data->current_time;
        extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(r), reinterpret_cast<void*>(&r));
    }
    // ...
}
```

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: adobe-plugin-devs · 2024-03-28 · Tags: `guid-mixin`, `cache-invalidation`, `pre-render`, `force-rerender`*

---

## What are the benefits of SmartFX (SmartRender) over regular Render?

SmartFX: (1) Only requests pixels needed for the specific processing area via PreRender. (2) Required for 32bpc support. (3) Allows checking out frames with prior effects applied via checkout_layer_pixels. (4) Supports MFR (Multi-Frame Rendering) better. (5) Provides more control over buffer management. Even if your effect needs the full image, SmartFX is recommended for 32bpc support and future-proofing.

*Contributors: [**fad**](../contributors/fad/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2024-06-10 · Tags: `smart-render`, `smartfx`, `render`, `32bpc`, `pre-render`*

---

## Why might SmartRender never be called even though PreRender returns PF_Err_NONE?

Possible causes: (1) GuidMixInPtr in PreRender is telling AE 'no new render needed' if the mixed-in data hasn't changed. (2) Mismatched SDK versions - building with 2025.2 SDK source but 2023 SDK headers causes undefined behavior, especially in Release builds where optimizations may expose the mismatch. Always ensure your SDK headers and source files are from the same version.

*Contributors: [**tlafo**](../contributors/tlafo/), [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2025-04-09 · Tags: `smart-render`, `pre-render`, `guid-mixin`, `sdk-version`, `release-build`*

---

## How do I expand the output buffer beyond the layer size using SmartFX?

Put the expanded buffer rect in extra->output->result_rect during PreRender. AE will use that rect for the render call. Expand the in_result.max_result_rect and put the expanded result in extra->output->max_result_rect. When implementing SmartRender, account for the origin offsets between input and output worlds (input_worldP->origin_x - output_worldP->origin_x). The key consideration is what you expand relative to: the cropped buffer, the original layer size, or the max rect of the input.

```cpp
PF_LRect expanded_rect = in_result.result_rect;
expanded_rect.left -= expansion;
expanded_rect.top -= expansion;
expanded_rect.right += expansion;
expanded_rect.bottom += expansion;
extra->output->result_rect = expanded_rect;

// Also expand max_result_rect from in_result.max_result_rect
PF_LRect expanded_max = in_result.max_result_rect;
expanded_max.left -= expansion;
expanded_max.top -= expansion;
expanded_max.right += expansion;
expanded_max.bottom += expansion;
extra->output->max_result_rect = expanded_max;
```

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-07-01 · Tags: `smartfx`, `buffer-expansion`, `pre-render`, `smart-render`, `result-rect`, `origin-offset`*

---

## What is the minimal code needed for SmartFX PreRender and SmartRender?

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

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2010-01-01 · Tags: `smartfx`, `smart-render`, `pre-render`, `minimal`, `bit-depth`, `boilerplate`*

---
