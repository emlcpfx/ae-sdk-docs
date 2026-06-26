# SmartFX / Smart Render

> 135 Q&As · source: AE plugin dev community Discord

### Why is checkout_layer_pixels returning data == NULL for a layer parameter in SmartRender?

In GPU mode (CUDA, Metal, OpenCL), the SDK documentation states that you must assume the data pointer is NULL in PF_EffectWorld. The pixel data lives on the GPU, not in CPU memory. This is expected behavior when GPU rendering is active.

*Tags: `checkout-layer`, `effect-world`, `gpu`, `null-data`, `smart-render`*

---

### How do you check out a frame with prior effects applied (not the raw layer)?

Use checkout_layer_pixels (from PF_SmartRenderCallbacks in SmartRender) instead of PF_CHECKOUT_PARAM. PF_CHECKOUT_PARAM always returns the layer before all effects were applied. checkout_layer_pixels returns the frame with all prior effects applied. You must first checkout the layer in PreRender, then checkout_layer_pixels in SmartRender.

*Tags: `checkout-layer`, `effect-stack`, `frame-checkout`, `pre-render`, `smart-render`*

---

### What are the benefits of SmartFX (SmartRender) over regular Render?

SmartFX: (1) Only requests pixels needed for the specific processing area via PreRender. (2) Required for 32bpc support. (3) Allows checking out frames with prior effects applied via checkout_layer_pixels. (4) Supports MFR (Multi-Frame Rendering) better. (5) Provides more control over buffer management. Even if your effect needs the full image, SmartFX is recommended for 32bpc support and future-proofing.

*Tags: `32bpc`, `pre-render`, `render`, `smart-render`, `smartfx`*

---

### How do you read sequence data in SmartRender since AE CC2021/2022?

Since CC2021/2022, in_data->seq_data is not correctly updated in render threads. You must use PF_EffectSequenceDataSuite1 with PF_GetConstSequenceData instead. The const handle must be locked with PF_LOCK_HANDLE before use. Note: According to Adobe engineers, PF_LOCK_HANDLE hasn't done anything since CS6 (it's a dummy function), but the code pattern of casting via the handle is still necessary for correct access.

```cpp
PF_ConstHandle const_seq;
AEFX_SuiteScoper<PF_EffectSequenceDataSuite1> seqdata_suite =
    AEFX_SuiteScoper<PF_EffectSequenceDataSuite1>(
        in_data,
        kPFEffectSequenceDataSuite,
        kPFEffectSequenceDataSuiteVersion1,
        out_data);
seqdata_suite->PF_GetConstSequenceData(in_data->effect_ref, &const_seq);
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `mfr`, `pf-get-const-sequence-data`, `sequence-data`, `smart-render`, `threading`*

---

### Why might SmartRender never be called even though PreRender returns PF_Err_NONE?

Possible causes: (1) GuidMixInPtr in PreRender is telling AE 'no new render needed' if the mixed-in data hasn't changed. (2) Mismatched SDK versions - building with 2025.2 SDK source but 2023 SDK headers causes undefined behavior, especially in Release builds where optimizations may expose the mismatch. Always ensure your SDK headers and source files are from the same version.

*Tags: `guid-mixin`, `pre-render`, `release-build`, `sdk-version`, `smart-render`*

---

### Is checking out the current layer with prior effects applied possible in Premiere?

In AE, you can use checkout_layer_pixels in SmartRender to get the layer with prior effects applied. However, this does not work in Premiere. In Premiere, there is no straightforward way to check out the current layer with prior effects applied - PF_CHECKOUT_PARAM always returns the pre-effects layer. A potential hacky workaround might involve calling Premiere's JS scripting API to render, but this is not a clean solution.

*Tags: `checkout-layer`, `limitations`, `premiere`, `prior-effects`, `smart-render`*

---

### Does Premiere support SmartFX for requesting only a portion of the input buffer?

Premiere does not support SmartFX. You cannot officially request only a specific region of the buffer like you can in AE. As a workaround, you can manually iterate through the data using row bytes to delimit the part you need. Be careful with negative row byte values in Premiere.

*Tags: `buffer`, `premiere`, `region-of-interest`, `row-bytes`, `smartfx`, `transition`*

---

### How can I make a plugin for 8-bit, 16-bit, and 32-bit projects share the same internal codebase?

Use C++ templates to create 3 instances of your algorithm with different data types and range constants. This allows you to write the core logic once and instantiate it for PF_Pixel8, PF_Pixel16, and PF_PixelFloat with appropriate range constants for each bit depth.

*Tags: `bit-depth`, `code-reuse`, `pixel-format`, `smart-render`, `templates`*

---

### How do I expand the output buffer beyond the layer size using SmartFX?

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

*Tags: `buffer-expansion`, `origin-offset`, `pre-render`, `result-rect`, `smart-render`, `smartfx`*

---

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

### Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Tags: `code-organization`, `cpp`, `dll`, `prerender`, `smartfx`, `smartrender`*

---

### How do you generate unique checkout IDs for checkout_layer/checkout_layer_pixels in SmartFX with multiple layers and instances?

This is a known challenge when dealing with multiple layers, multiple instances, and nested loops in SmartFX PreRender/SmartRender. The checkout ID must be unique across all checkouts. No definitive solution was provided in the discussion, but the issue typically arises with many cloned plugin instances resulting in 'checkout id is not unique' errors.

*Tags: `checkout-layer`, `multiple-instances`, `prerender`, `smartfx`, `smartrender`, `unique-id`*

---

### How do you cache frames in an After Effects effect plugin and use the cached result as input for processing the next frame?

This requires implementing frame caching within your plugin's render loop. You need to store the output of frame N, then retrieve it to use as input for frame N+1. This typically involves maintaining a persistent buffer or cache structure across multiple render calls. The exact implementation depends on whether you're using SmartFX or legacy plugins, but generally involves allocating memory to store intermediate results and managing that cache through the plugin's lifecycle to ensure the previous frame's output is available when processing subsequent frames.

*Tags: `caching`, `memory`, `output-rect`, `render-loop`, `smartfx`*

---

### How do you expand the output buffer with smartFX to prevent outlines from being cut off by the layer bounding box?

In smartFX, you need to modify the output rectangle to be larger than the input layer bounds. This is typically done by adjusting the output_rect parameter in your render function to expand beyond the original layer boundaries, allowing your outline effect to render fully without clipping.

*Tags: `output-rect`, `render-loop`, `smartfx`*

---

### What flags or methods are used to create a caching effect where rendering waits for frame caching to complete before updating?

This is a technique used in higher-end plugins like Particular where moving the Current Time Indicator (CTI) causes the plugin to wait for frame caching to complete before updating the render. This involves using smart render techniques and potentially compute cache mechanisms to defer rendering until cache state is ready.

*Tags: `caching`, `compute-cache`, `render-loop`, `smart-render`*

---

### How can you make After Effects re-render a frame multiple times until a calculation is complete?

You need to call render() several times in a loop until your calculation is done. This requires a flag that tells AE to re-render the frame again, and inside the render function you check with an if clause whether the render is actually finished or not.

*Tags: `debugging`, `render-loop`, `smart-render`*

---

### Can you store many PF_EffectWorld structures in a sequence data struct for caching purposes?

Yes, you can store thousands of PF_EffectWorld structures in your data struct. This can be used to cache frames by creating an array like PF_EffectWorld resultW[10000] and PF_EffectWorld matte[10000]. You can then implement logic in your main render function to cache all missing previous frames when a later frame is requested, using a loop to iterate and populate the cached frames before rendering the current frame.

```cpp
typedef struct {
    PF_EffectWorld  resultW[10000];
    PF_EffectWorld  matte[10000];
    bool            result_loaded = false;
    bool            isCaching = false;
    A_long          result_last_time;
    char            result_hash[260];
    int                firstFrame;
    bool            veryFirstLaunch;
    PF_InData        seqIn_data;
    PF_SampPB    seqSamp_pb;
} sequenceData;
```

*Tags: `caching`, `memory`, `sequence-data`, `smart-render`*

---

### What is the issue when trying to checkout multiple frames of a layer into an array during smart render?

When attempting to checkout multiple frames of a layer (such as thousands) using PF_CHECKOUT_PARAM in a loop, you will typically only succeed in filling about 12 matte worlds before encountering errors or bad callback parameters. This is a limitation of the checkout system - it is not designed to batch checkout multiple frames of the same layer into an array. The layer checkout system expects individual checkouts rather than mass array-based storage of the same layer across many frames.

```cpp
if (&seqdataP->matte[0].data != NULL && &checkout.u.ld.data != NULL) {
        for (int m = 0; m < 10000; m++) {
            if (int(in_data->time_scale*m) < int(in_data->total_time)) {
                    ERR(PF_CHECKOUT_PARAM(in_data,
                        VECTORMATTELAYER_DISK_ID,
                        int(in_data->time_scale * m),
                        in_data->time_step,
                        in_data->time_scale,
                        &checkout));
                    seqdataP->matte[m] = checkout.u.ld;
            }
        }
    }
```

*Tags: `layer-checkout`, `memory`, `sequence-data`, `smart-render`*

---

### Do you need to use AEGP for adding time values in the compute cache API?

The conversation indicates this is a question about the proper API approach for time value handling in the compute cache, but a definitive answer was not provided in the chat.

*Tags: `aegp`, `compute-cache`, `smartfx`*

---

### How can you access the previous frame result during the compute thread in After Effects?

The user asked this question but indicated that despite reading the documentation, a full example would be more helpful. No complete answer was provided in the conversation.

*Tags: `compute-cache`, `memory`, `render-loop`, `smartfx`*

---

### How do you invalidate cached frames to force After Effects to re-render them instead of using cached versions?

The user is asking about the mechanism used by effects like Warp Stabilizer and 3D Camera Tracker to invalidate frames. No answer was provided in this conversation.

*Tags: `caching`, `render-loop`, `smart-render`*

---

### How do you invalidate rendered frames in smart render when external data changes?

Set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag and use extra->cb->GuidMixInPtr in pre-render to scan data that should trigger invalidation. You can add multiple GuidMixInPtr calls for multiple data sources. Store data that affects rendering in sequence data or static variables, not in globalData, since After Effects expects globalData to remain unchanged outside of globalsetup/setdown.

```cpp
if (extra->cb->GuidMixInPtr && !err) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan_to_force_render), reinterpret_cast<void*>(*data_to_scan_to_force_render));
}
```

*Tags: `mfr`, `render-loop`, `sequence-data`, `smart-render`*

---

### Why does GuidMixInPtr not invalidate frames when using global variables or sequence data?

GuidMixInPtr appears to work only with AEGP-managed values or stack-allocated local variables. The workaround is to use an AEGP function or script to trigger frame invalidation, such as changing the composition background color via AEGP_ExecuteScript. This forces After Effects to check which frames need invalidation.

```cpp
if(extra->cb->GuidMixInPtr) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(bool), reinterpret_cast<void*>(&random_bool));
}
```

*Tags: `aegp`, `scripting`, `smart-render`*

---

### Why do frames only invalidate after manually triggering a change like modifying the comp background color?

The issue appears to be that After Effects does not check whether frames need invalidating frequently enough. Even though the boolean value updates properly (confirmed with breakpoints) and frames should invalidate, they only do so after giving AE a 'kick' by changing the composition background color or changing a parameter and changing it back.

*Tags: `debugging`, `render-loop`, `smart-render`*

---

### How can a plugin render frame N that depends on custom data from previous frames without checking out all 100 previous frames in memory during pre-render?

This is a known challenge with recursive operations in After Effects plugins. The issue is that smart render checks needed frames during pre-render, which forces checking out all intermediate frames. The checkout param solution doesn't account for previous effects. AEGP_CacheAndCheckoutFrame doesn't trigger the pre_render/smart render thread, so it doesn't solve the problem. A potential approach is to use layer checkout parameters during compute cache operations, but this requires careful design to avoid excessive memory usage and to ensure previous effects are properly evaluated.

*Tags: `aegp`, `compute-cache`, `layer-checkout`, `memory`, `smart-render`*

---

### Why is layer data NULL after checkout_layer_pixels when a layer is selected in the layer selection parameter?

This issue typically occurs when the layer parameter is not properly initialized or when the layer selection is invalid. Ensure that PF_ADD_LAYER uses appropriate default values and that the layer index being checked out actually contains valid data. The issue may also resolve itself if the layer parameter setup or plugin caching is refreshed. Verify that the layer being checked out is not disabled, hidden, or has zero dimensions, as these conditions can result in NULL data pointers.

```cpp
// Verify layer parameter setup
AEFX_CLR_STRUCT(def);
PF_ADD_LAYER(LAYER_STR, PF_LayerDefault_NONE, GPU_SKELETON_LAYER_Disk_ID);

// In SmartRender, add null checks before accessing data
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `debugging`, `layer-checkout`, `params`, `smart-render`*

---

### Why does checkout_layer_pixels return NULL data for a layer parameter after checkout_layer succeeds?

This appears to be a layer checkout issue where the layer parameter has valid layer data after PreRender checkout_layer calls, but checkout_layer_pixels in SmartRender returns a structure with NULL data. The issue may be related to how the layer parameter is defined or how Adobe's GPU template handles layer checkouts. Ensure the layer parameter is properly defined with PF_ADD_LAYER and that the same layer index is used consistently between PreRender and SmartRender callbacks.

```cpp
// ParamsSetup
AEFX_CLR_STRUCT(def);
PF_ADD_LAYER(LAYER_STR, PF_LayerDefault_NONE, GPU_SKELETON_LAYER_Disk_ID);

// SmartRender
ERR(extraP->cb->checkout_layer_pixels(in_data->effect_ref, GPU_SKELETON_LAYER, &env_worldP));
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `aegp`, `gpu`, `layer-checkout`, `params`, `smart-render`*

---

### How can I check out the current layer (INPUT) with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT on the INPUT layer at different times, it returns the effectworld before all effects are applied, even those that precede the current effect in the chain. To get the current layer at different times with all effects applied, you need to use a different approach than simple PF_CHECKOUT. The Echo effect demonstrates that this is possible, likely through using layer composition or render callbacks that account for the full effect stack.

*Tags: `layer-checkout`, `params`, `render-loop`, `smartfx`*

---

### Can you use AEGP render suite during the render call, or only during PreRender?

You cannot use AEGP render suite during the render call itself. However, you can use it during PreRender to get rendered frames at a given time with the previous effect reference. Alternatively, you can use smart render with frame preloading in pre-render, using the same method as the current frame.

*Tags: `aegp`, `render-loop`, `smart-render`, `smartfx`*

---

### Can arb data parameter values be accessed in paramssetup?

Arb data can be accessed in paramssetup using the standard access functions (similar to how it's accessed in other functions like smartrender), allowing you to read arbP member variables.

*Tags: `arb-data`, `params`, `smartfx`*

---

### When do I need to implement PF_Cmd_FRAME_SETUP to customize output size?

You only need to implement PF_Cmd_FRAME_SETUP for non-SmartFX effects if your effect expands the output buffer size, such as with glow or drop shadow effects. For effects that maintain the same output dimensions as the input, this command is not necessary.

*Tags: `mfr`, `output-rect`, `smartfx`*

---

### Does smart render provide benefits for effects that require the full image each time they are processed?

Smart render optimizes by only requesting pixels required for specific processing. For effects that require the full image each time, smart render's primary optimization of selective pixel fetching provides limited benefit since the effect must process the entire image anyway. However, smart render can still provide benefits in other scenarios such as avoiding redundant computation when the effect is cached or when only part of the output is needed by downstream effects.

*Tags: `caching`, `output-rect`, `render-loop`, `smart-render`*

---

### Does smart render provide any benefit for effects that require the full image each time it is processed?

Smart render helps by only getting pixels required for specific processing. For effects that require the full image each time it is processed, smart render would not provide additional benefit since the entire image must be obtained regardless.

*Tags: `output-rect`, `smart-render`, `smartfx`*

---

### What is required to support 32 bits per channel (32 bpc) in an After Effects effect?

You must use SmartFX if you want your effect to support 32 bpc.

*Tags: `params`, `smartfx`*

---

### Is SmartFX the only way to detect if an input frame actually changed between renders?

SmartFX provides built-in mechanisms to detect frame changes, but it's not necessarily the only way. You can also implement custom change detection by comparing frame data or checksums, monitoring layer properties (scale, position, effects), and using the render queue system to track what has actually changed. SmartFX simplifies this by providing smart rendering capabilities that automatically optimize based on detected changes.

*Tags: `render-loop`, `smart-render`, `smartfx`*

---

### Is SmartFX the only way to detect if an input frame actually changed?

No, SmartFX is not the only way. You can use PF_GetCurrentState to tell if an input has changed. Additionally, if an input frame changes you should certainly get a re-render. If you need to force another prerender for other changes, you can mix anything else into the guid mix in ptr.

*Tags: `render-loop`, `smart-render`, `smartfx`*

---

### How are transforms applied to input buffers in the dumb pipeline versus SmartFX?

In SmartFX, if the layer is continuously rasterised, you receive the input buffer with transforms already applied. If it's not continuously rasterised, the transforms are applied after your effect populates the output buffer. In the dumb pipeline, transforms are applied after the effect populates the output buffer.

*Tags: `output-rect`, `render-loop`, `smartfx`*

---

### How do you check out a frame of the current layer after prior effects are applied instead of before them?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects. However, this approach may not work in Premiere Pro. An alternative approach using Cmd_RENDER may be needed for cross-application compatibility.

*Tags: `layer-checkout`, `premiere`, `render-loop`, `smartfx`*

---

### How difficult would it be to match After Effects' lighting calculations in a custom 3D plugin?

This is a challenging task because it requires matching AE's specific lighting parameters and their meanings, such as spotlight falloff softness and the balance between ambient and direct lighting. Many plugins have implemented this, though the effort is significant.

*Tags: `gpu`, `metal`, `opengl`, `smartfx`*

---

### How do you ensure that an effect uses unaltered input pixels rather than the plugin's previously rendered output when a parameter changes?

After Effects automatically provides the correct input layer. In PF_Cmd_SMART_RENDER, use extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world) to get the right input. In PF_Cmd_RENDER, use params[0]->u.ld to access the correct input layer. You don't need to do anything special; AE handles giving you the unaltered input, not your plugin's previous output.

```cpp
// PF_Cmd_SMART_RENDER
extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world);

// PF_Cmd_RENDER
params[0]->u.ld // correct input layer
```

*Tags: `layer-checkout`, `params`, `smart-render`*

---

### How should sequence_data be accessed in SmartRender when it appears as nullptr?

Since After Effects v22, you must use the PF_EffectSequenceDataSuite to read sequence data in SmartRender. Create an AEFX_SuiteScoper for PF_EffectSequenceDataSuite1, call PF_GetConstSequenceData with the effect_ref to get a const handle, then lock that handle to access the sequence data. This provides read-only access to the sequence data.

```cpp
PF_ConstHandle const_seq;
AEFX_SuiteScoper<PF_EffectSequenceDataSuite1> seqdata_suite =
    AEFX_SuiteScoper<PF_EffectSequenceDataSuite1>(
        in_data,
        kPFEffectSequenceDataSuite,
        kPFEffectSequenceDataSuiteVersion1,
        out_data);
seqdata_suite->PF_GetConstSequenceData(in_data->effect_ref, &const_seq);
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `aegp`, `memory`, `sequence-data`, `smart-render`*

---

### Does using sequence data in SmartRender have a negative impact on performance?

No significant performance impact was observed. The read-only access pattern through the PF_EffectSequenceDataSuite in SmartRender does not cause noticeable performance degradation.

*Tags: `performance`, `sequence-data`, `smart-render`*

---

### Why does SmartRender never get called in Release builds even when PreRender returns PF_Err_NONE?

The issue was caused by a mismatch between the SDK code version used for building (2025.2) and the SDK headers (2023). The debug configuration worked fine because it didn't have this version mismatch. Ensure that both the SDK code and headers are from the same version when building.

*Tags: `build`, `debugging`, `release`, `smart-render`, `smartfx`*

---

### What is the exact behavior of the GuidMixInPtr callback from extra->cb in PreRender?

The GuidMixInPtr callback in PreRender can return a result that indicates whether a new render is needed or not. However, using or disabling this callback alone may not resolve SmartRender not being called if there are other underlying issues like SDK version mismatches.

```cpp
extra->cb->GuidMixInPtr
```

*Tags: `params`, `smart-render`, `smartfx`*

---

### Is there a method for forcing masks on a layer to be applied after the plugin processes?

No answer was provided in the conversation.

*Tags: `layer-checkout`, `render-loop`, `smartfx`*

---

### Is it possible to create a multi-layer effect that discovers material properties from per-layer dummy effects and performs global illumination rendering while properly communicating dependencies to After Effects' caching system?

Tobias Fleischer shared that he attempted a plugin-per-layer approach years ago but abandoned it due to render order troubles. Instead, he recommended using a single plugin with multiple layer parameters that pull in pixels from different layers, apply material/lighting options, and composite everything within that single plugin. This approach worked well despite AE's rigid parameter interface being somewhat clunky. The viability of the frankensteined multi-layer approach with proper dependency tracking depends on whether the SDK allows explicit communication of implicit dependencies to AE's caching system, though existing solutions tend to consolidate the logic into a single effect rather than relying on scattered per-layer effects.

*Tags: `caching`, `layer-checkout`, `output-rect`, `params`, `smart-render`*

---

### Is it possible to make a multi-layer 2D global illumination effect where a renderer effect discovers material metadata effects on other layers and communicates these dependencies to AE's caching system?

It is theoretically possible using AEGP_RenderAndCheckoutLayerFrame to enumerate layers and pass their GUIDs via GuidMixInPtr() to register dependencies. However, a more practical approach is to use hidden layer parameters (up to 999) with a script button that assigns layers from the composition to these parameters, allowing checkout_layer() to work efficiently with SmartPreRender. You can also mix in layer order information via GuidMixInPtr() to handle dynamic layer changes without manual updates.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `smart-render`*

---

### Can checkout_layer() be used on arbitrary composition layers during PF_Cmd_SMART_PRE_RENDER that aren't declared as effect parameters?

According to the SDK documentation, you can only grab layer parameters that are explicitly declared as layer parameters on the effect. To work with arbitrary layers, use hidden layer parameters that a script can populate, or use AEGP_RenderAndCheckoutLayerFrame which works with any layer IDs.

*Tags: `aegp`, `layer-checkout`, `smart-render`*

---

### Does AEGP_RenderAndCheckoutLayerFrame perform the expensive rendering immediately or defer it until AEGP_GetReceiptWorld is called?

AEGP_RenderAndCheckoutLayerFrame performs the expensive render operation immediately and returns an AEGP_FrameReceiptH. You can then call AEGP_GetReceiptGuid() on the receipt to get a GUID that can be mixed into GuidMixInPtr() for dependency tracking. There is both a synchronous and asynchronous version available; the async version could potentially be called during SmartPreRender and awaited during the render call/thread.

*Tags: `aegp`, `layer-checkout`, `smart-render`, `threading`*

---

### Is it possible to achieve progressive rendering in an After Effects plugin that continuously shows results to the user as more samples are computed?

The conversation acknowledges the desire for progressive rendering where samples accumulate and display over time rather than waiting for all samples to complete before showing results. While the technical feasibility is not explicitly detailed in the response, the user notes they have this working outside of After Effects and finds it more interactive. The challenge with current AE plugin behavior is that it waits for complete render before display, making it feel slow compared to progressive sampling approaches.

*Tags: `preview`, `render-loop`, `smart-render`, `ui`*

---

### How do 3D Camera Tracker and Warp Stabilizer output progress over frames with live updates without cache purging?

According to an After Effects developer, these built-in tools use internal APIs that are not available to regular plugin developers. The Warp Stabilizer effect 'cheats' and does not operate within the constraints of a normal AE Effect API plug-in. Adobe's internal team has access to secret suites that allow this functionality, which is not exposed to third-party developers. A feature request was made in 2023 to expose this capability to plugin developers, but it remains unavailable.

*Tags: `aegp`, `caching`, `debugging`, `mfr`, `smart-render`*

---

### How can you limit the output buffer to a specific region when Premiere doesn't support smartFX for transitions?

Since Premiere Pro does not support smartFX like After Effects does, you cannot use the smartFX output_rect mechanism to specify a smaller region of interest. When building transitions for Premiere, the plugin receives frames at the full sequence resolution even if the source footage is smaller. You need to work with the full buffer dimensions provided by Premiere's API rather than requesting a subset.

*Tags: `output-rect`, `premiere`, `smartfx`, `transition`*

---

### Why does Premiere pass NULL for input_world in Render calls when compiled with fast optimizations, while After Effects doesn't have this issue?

When a plugin is compiled with fast optimizations, Premiere may pass input_world = NULL in the Render call, whereas the same code without optimizations receives the data correctly. This issue does not occur in After Effects. The root cause appears to be related to compiler optimization levels affecting how Premiere handles input data passing to plugins.

*Tags: `debugging`, `premiere`, `render-loop`, `smartfx`*

---

### How should you store GPU and CPU data across multiple frames in After Effects plugins?

For GPU work, store data in global data or sequence data and access it later. For CPU work, cache the PF_EffectWorld in the global or sequence data. This approach may need validation against the new 3-way-checkout mechanism.

*Tags: `caching`, `gpu`, `memory`, `sequence-data`, `smartfx`*

---

### Why does a displacement map effect sometimes use the original frame instead of the previously cached frame?

Nate is experiencing inconsistent behavior with a displacement mapping effect that should feed the distorted output back as input for the next frame. The effect sometimes works correctly (distorting and caching each frame to use as the next input) but randomly reverts to using the original frame instead of the cached frame. This suggests a potential issue with the frame caching or smart render logic that determines which input frame gets used for displacement. The problem appears to be intermittent rather than systematic.

*Tags: `caching`, `debugging`, `render-loop`, `smart-render`*

---

### How can you debug why a cached frame is not being used even though it should be?

To diagnose caching issues, first determine whether the frame is always cached but not always used, or if it's not being cached consistently. If it's always cached but not always used, the problem lies in the conditional logic that checks cache validity. Start by disabling conditions in the if statement that verify cache availability (such as hash checks) to isolate which condition is returning false and preventing cache usage.

*Tags: `caching`, `debugging`, `memory`, `smart-render`*

---

### How do you expand the output buffer with smartFX to prevent outlines from being clipped by layer bounds?

When using smartFX in After Effects, if you're generating outlines around an alpha image and they're getting cut off by the layer bounding box, you need to expand the output buffer. This is typically done by modifying the output rectangle in your smartFX implementation to be larger than the layer's current bounds, ensuring that generated content (like outlines) isn't clipped.

*Tags: `mfr`, `output-rect`, `render-loop`, `smartfx`*

---

### What needs to be updated when using PF_COPY or buffer operations in a plugin?

When performing buffer operations like PF_COPY, the source and destination rectangles (src and dst rects) must be updated to reflect the actual regions being operated on.

*Tags: `output-rect`, `render-loop`, `smartfx`*

---

### What flags or methods are used in high-end plugins like Particular to cache frames and delay render updates until caching is complete?

The user is asking about caching mechanisms used in professional plugins like Particular that intelligently manage frame rendering by waiting for frame cache completion before updating the CTI (Current Time Indicator). This typically involves smart render and compute cache APIs provided by After Effects to optimize performance when scrubbing the timeline.

*Tags: `caching`, `compute-cache`, `render-loop`, `smart-render`*

---

### How can you implement an iterative render loop where a frame needs to be re-rendered multiple times until a calculation is complete?

According to gabgren, you can call render() multiple times within a single frame processing cycle. This requires a flag to tell After Effects to re-render the frame, and inside the render function, an if clause should check whether the render is actually done or not. This pattern was used in threads discussing multi-pass rendering approaches.

*Tags: `aegp`, `render-loop`, `smart-render`*

---

### What is the correct approach for checking out multiple frames of a layer parameter during Smart Render for frame-based caching?

When trying to checkout multiple frames of a layer parameter (like a matte layer) during Smart Render checkins/checkouts, the standard PF_CHECKOUT_PARAM loop has limitations and typically fails after ~12 successful checkouts due to callback parameter constraints. The recommended approach is to checkout layers during the Smart Render process frame-by-frame rather than attempting to batch checkout all frames in a single loop, or to implement alternative caching strategies that respect the plugin architecture's layer checkout limitations.

```cpp
if (&seqdataP->matte[0].data != NULL && &checkout.u.ld.data != NULL) {
        for (int m = 0; m < 10000; m++) {
            if (int(in_data->time_scale*m) < int(in_data->total_time)) {
                    ERR(PF_CHECKOUT_PARAM(in_data,
                        VECTORMATTELAYER_DISK_ID,
                        int(in_data->time_scale * m),
                        in_data->time_step,
                        in_data->time_scale,
                        &checkout));
                    seqdataP->matte[m] = checkout.u.ld;
            }
        }
    }
```

*Tags: `caching`, `layer-checkout`, `memory`, `sequence-data`, `smart-render`*

---

### Why does a plugin render get interrupted with PF_Interrupt_CANCEL during queue rendering?

PF_Interrupt_CANCEL during render queue operations can occur under specific conditions. One user reported this happening after 10-15 frames when CAPS LOCK was off and the composition had not been previously previewed to RAM cache. The interrupt appears as 'User abort' but occurs without user action. The behavior did not occur when CAPS LOCK was on or when the composition was fully previewed in the RAM cache before rendering. This suggests the interrupt may be related to input handling or cache state during rendering.

*Tags: `debugging`, `render-loop`, `smartfx`*

---

### How do you invalidate cached frames in After Effects so they will be re-rendered instead of using the cache?

This question was asked about invalidating rendered frames to force After Effects to re-render them, similar to how built-in effects like Warp Stabilizer and 3D Camera Tracker work. However, no answer was provided in the conversation.

*Tags: `aegp`, `caching`, `render-loop`, `smart-render`*

---

### How do you invalidate rendered frames when data changes in a plugin using smart render?

Set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag and use GuidMixInPtr in pre-render to scan data that should force re-rendering. In pre-render, call: if (extra->cb->GuidMixInPtr && !err) { extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan), reinterpret_cast<void*>(*data_to_scan)); }. You can add multiple GuidMixInPtr calls for different data sources. This works particularly well when data is stored in sequence data structures.

```cpp
if (extra->cb->GuidMixInPtr && !err) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan_to_force_render), reinterpret_cast<void*>(*data_to_scan_to_force_render));
}
```

*Tags: `params`, `render-loop`, `sequence-data`, `smart-render`*

---

### How should global variables be used in multi-threaded plugin development?

Define global variables at the top of the main .cpp file, initialized only once, to ensure the same value is accessed across both UI and render threads (which have run separately since CC 2014). Use atomic bools for thread-safe access, though mutexes may be needed in some cases. Avoid using static variables with MFR (Multi-Frame Rendering), but non-MFR plugins should be fine. Pass pointers to these globals carefully, as passing pointers (e.g., bool*) can cause crashes—use value types (e.g., bool) instead.

```cpp
static bool global_variable;
// In PF_cmd or prerender:
if(extra->cb->GuidMixInPtr) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(bool), reinterpret_cast<void*>(&global_variable));
}
```

*Tags: `memory`, `smart-render`, `threading`*

---

### How can you trigger frame invalidation from external background threads in After Effects plugins?

GuidMixInPtr can be problematic for triggering invalidation from background threads when using global or sequence data variables. A reliable workaround is to change a composition property (like background color) via AEGP calls or scripting, which forces After Effects to check if frames need invalidation. Another approach is to use a hidden parameter that gets modified through AEGP or scripting calls, though this has limitations when modal dialogs are open. The method must ultimately signal to After Effects through a mechanism it actively monitors.

*Tags: `aegp`, `params`, `sequence-data`, `smart-render`, `threading`*

---

### How can a plugin implement recursive operations where frame N depends on computed data from previous frames without checking out all intermediate frames during pre-render?

This is a known challenge when combining smart render with compute caches that require sequential frame dependencies. The user attempted several approaches: (1) using a compute cache with smart render, but this requires checking out all previous frames during pre-render when rendering frame 100, which is memory-inefficient; (2) using checkout params during compute cache, which doesn't account for previous effects; (3) using AEGP_CacheAndCheckoutFrame, which doesn't trigger pre_render/smart render callbacks on checked-out frames. A solution may involve manually managing frame dependencies outside the smart render system or using layer checkout strategically to load only necessary intermediate results rather than relying on automatic frame dependency detection.

*Tags: `aegp`, `compute-cache`, `layer-checkout`, `memory`, `sequence-data`, `smart-render`*

---

### How do you access computed data from previous frames in a compute cache when using smart render?

When accessing computed data from frame N-1 in a compute cache, if that frame hasn't been computed yet, you need access to the layer at the previous frame. This creates a cascading dependency—if N-2 is also not calculated, you need the layer at N-2 as well. With smart render, pre-loading all needed layers during pre-render can be expensive, especially if the user sets the timeline cursor at an arbitrary time like 10 seconds. The solution is to use classical checkout_param during compute cache, though this loads layers without their effects applied.

*Tags: `compute-cache`, `layer-checkout`, `render-loop`, `smart-render`*

---

### Why is layer pixel data NULL after checkout_layer_pixels for a selected layer parameter?

When checkout_layer_pixels returns a non-NULL world pointer but the data field within it is NULL, this typically indicates that the layer parameter has no layer selected or the layer is disabled/invisible. Ensure that in your layer parameter definition you're using an appropriate default (like PF_LayerDefault_NONE or a specific layer) and verify in PreRender that the layer checkout succeeded. The issue may also resolve itself after rebuilding or reloading the plugin. Check that the layer parameter is actually populated with a valid layer selection before SmartRender is called.

```cpp
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `debugging`, `layer-checkout`, `params`, `smart-render`*

---

### Why is checkout_layer_pixels returning NULL data for a layer parameter even though the layer is selected?

Tim Constantinov reported an issue where after calling checkout_layer_pixels on a layer selected in a layer parameter, the data field equals NULL. The problem occurs in SmartRender after checkout_layer_pixels is called on GPU_SKELETON_LAYER. The issue appears to be intermittent and may be related to the Adobe-supplied GPU template. The exact cause wasn't definitively identified in the conversation, but Tim noted this issue resolved itself spontaneously in the past.

```cpp
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `debugging`, `gpu`, `layer-checkout`, `params`, `smartfx`*

---

### How should a plugin handle GPU out-of-memory errors differently for interactive scrubbing versus render queue operations?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects handles it differently depending on context. During MFR (Multi-Frame Rendering) queue renders, AE may retry with a lower thread count. However, during interactive timeline scrubbing, AE doesn't warn the user and may display a black frame instead. The challenge is that PF_Err_OUT_OF_MEMORY warnings in out_data will fail the render, and there is currently no documented method to distinguish between a render queue request versus interactive scrubbing. Plugins must accept that they cannot reliably warn users about resolution/bitdepth being too high for available GPU memory in interactive contexts without failing the frame.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `smartfx`*

---

### How do you generate a key and create a pointer for computeIfNeeded or reading a function in After Effects plugins?

When you call computeIfNeeded or want to read a function, you create a pointer and must set in_data->pica_basicP in this pointer along with the other necessary values. The compute cache operates as something independent where you must send everything needed for the computation.

*Tags: `aegp`, `compute-cache`, `params`, `smart-render`*

---

### How do you handle dependencies on previous frames in smart render when you need data from multiple prior frames?

In smart render, if your compute needs the result of a previous frame (e.g., frame N-1) but also requires frame N-2, you cannot use the smart checkout layer. Instead, you must use the classical checkout/checkin parameter approach to access the frames you need.

*Tags: `layer-checkout`, `render-loop`, `sequence-data`, `smart-render`*

---

### How can I force After Effects to invalidate cached frames when I modify global data in my plugin?

Simply setting PF_OutFlag_FORCE_RERENDER in outflags while changing a GuidMixInPtr value in extra->cb may not always work reliably. One workaround is to "kick" AE by triggering an internal frame cache check, such as changing the composition background color and changing it back. However, if you properly use GuidMixInPtr for forcing rerenders, it should work without this workaround—if you're experiencing issues, it may be worth reviewing source code examples from developers who have successfully implemented this.

*Tags: `aegp`, `compute-cache`, `params`, `smartfx`*

---

### Can one effect render from another effect during smartrender in After Effects?

During smartrender, it is not straightforward to have one effect call another effect's render. Smartrender happens on the render thread, and AE will not call smartrender on other layers if a current smartrender is already in progress—it waits until the current smartrender completes. One possible approach mentioned is to use EffectCallGeneric with a custom passthrough code if the other plugin implements it, but this has significant limitations. AEGP calls cannot be made from the render thread directly; they must be delegated to the UI thread. The async render and checkout functions (AEGP_RenderAndCheckoutLayerFrame_Async) might work from any thread but are noted as buggy. Ultimately, the questioner settled on manually implementing effects internally rather than calling other plugins dynamically.

*Tags: `aegp`, `render-loop`, `smart-render`, `threading`*

---

### Why does After Effects throw an 'effect attempting to modify a locked project' error when adjusting effect parameters during smartrender?

During a smartrender command, the project becomes locked to prevent modifications. Attempting to use AEGP calls to adjust effect parameters while smartrender is in progress results in this error because AEGP operations that modify the project cannot execute on the render thread. Such modifications must be deferred to the UI thread after the smartrender completes.

*Tags: `aegp`, `params`, `smart-render`, `threading`*

---

### Does smart render provide benefits for effects that require the full image for processing?

Smart render helps by only requesting pixels required for specific processing. For effects that require the full image each time they are processed (when visible), smart render may not provide additional optimization benefits since the entire image must be fetched anyway. The advantage of smart render is primarily for effects that can work on partial regions of interest.

*Tags: `optimization`, `output-rect`, `render-loop`, `smart-render`*

---

### Does smart render provide benefits for effects that require the full image each time they process?

Smart render helps by only getting pixels required for the specific processing needed. For an effect that requires the full image each time it is processed (when visible), smart render may not provide additional benefits since the entire image must be fetched anyway.

*Tags: `render-loop`, `smart-render`, `smartfx`*

---

### Is SmartFX required for 32-bit per channel color depth support?

Yes, you must use SmartFX if you want your effect to support 32 bpc (32-bit per channel) color depth.

*Tags: `params`, `smartfx`*

---

### How can I detect if an input frame has actually changed in an After Effects plugin?

You can use PF_GetCurrentState to tell if an input has changed. Additionally, if an input frame changes you should certainly get a re-render. If you need to force another prerender, you can mix it into the guid mix in ptr.

*Tags: `layer-checkout`, `params`, `render-loop`, `smartfx`*

---

### How does transform application differ between SmartFX and standard pipelines in After Effects?

In SmartFX, if a layer is continuously rasterized, you receive the input buffer with transforms already applied. If not continuously rasterized, the transforms are applied after your effect populates the output buffer. This means you won't know if transforms changed in the standard pipeline unless using SmartFX.

*Tags: `mfr`, `output-rect`, `render-loop`, `smartfx`*

---

### How do you check out a frame of the current layer after prior effects are applied in After Effects?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects have been applied. This allows you to access the layer state after upstream effects like Exposure have been rendered, but before the current effect processes it. However, this approach may not work in Premiere Pro, requiring alternative solutions for cross-application compatibility.

*Tags: `aegp`, `layer-checkout`, `premiere`, `render-loop`, `smartfx`*

---

### How do you ensure an After Effects effect uses unaltered input pixels rather than output from a previous render when parameters change?

After Effects automatically provides the correct input layer on each render call. In PF_Cmd_SMART_RENDER, use extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world) to get the right input. In PF_Cmd_RENDER, use params[0]->u.ld to access the correct input layer. You don't need to do anything special—AE handles providing the unaltered input, not the output from your plugin's previous render.

```cpp
// PF_Cmd_SMART_RENDER
extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world);

// PF_Cmd_RENDER
params[0]->u.ld  // correct input layer
```

*Tags: `aegp`, `layer-checkout`, `render-loop`, `smart-render`*

---

### Why does SmartRender never get called in Release mode even when PreRender returns PF_Err_NONE?

SmartRender may not be called if there's a version mismatch between the SDK code used for building and the SDK headers being compiled against. In the reported case, the plugin was being built with 2025.2 SDK code but using 2023 SDK headers, which caused SmartRender to not be invoked in Release builds while Debug builds worked fine. Ensure that the SDK version used for compilation matches the SDK headers included in the project.

*Tags: `build`, `debugging`, `release`, `smart-render`*

---

### What does the GuidMixInPtr callback in PreRender do?

The extra->cb->GuidMixInPtr callback in PreRender can indicate whether a new render is needed. If this callback suggests no new render is needed, SmartRender may not be called, which is expected behavior.

```cpp
extra->cb->GuidMixInPtr
```

*Tags: `aegp`, `params`, `smart-render`*

---

### How can a plugin effect communicate implicit dependencies on other layers and effects to After Effects' caching system?

When building multi-layer effects that discover and depend on other layers' properties at runtime, you need to explicitly communicate these dependencies to AE's render dependency tracking system. This involves checking out the layers you depend on and notifying AE of those checkouts so the cache knows to invalidate when those layers change. The renderer effect should enumerate dependent layers, check out their pixels/masks/shapes/transforms, and use the appropriate AEGP calls to register these dependencies so AE's smart render and disk cache systems understand the effect needs to re-render when those layers are modified.

*Tags: `aegp`, `caching`, `compute-cache`, `layer-checkout`, `memory`, `smart-render`*

---

### Is it feasible to build a 2D global illumination plugin using material property metadata effects on individual layers with a separate renderer effect?

Yes, this architecture is theoretically viable with the AE SDK. The pattern involves creating dummy effects on individual layers that only expose material parameters (emissive, diffuse, glass properties, etc.) without rendering output, while a separate renderer effect on an adjustment layer or solid discovers and interprets these metadata effects at render time. The key challenge is properly communicating cross-layer dependencies to AE's caching system so changes to materials trigger re-renders of the expensive global illumination computation while still benefiting from disk caching when unrelated elements change.

*Tags: `aegp`, `caching`, `layer-checkout`, `params`, `rendering`, `smart-render`*

---

### What are the practical challenges and solutions when designing multi-layer effects in After Effects plugins?

According to Tobias Fleischer (reduxFX), a plugin-per-layer approach was abandoned in favor of a single plugin with multiple layer parameters due to render order troubles. The successful approach involved creating a single plugin that accepts 10 layer parameters to pull in pixels from multiple layers, applies lighting/material options, and composites everything together within that single plugin. While this worked well, AE's rigid parameter interface made it somewhat clunky to use. The key insight is that centralizing the multi-layer logic into a single plugin avoids render order dependency issues that arise from distributing logic across multiple per-layer plugins.

*Tags: `layer-checkout`, `params`, `render-loop`, `smartfx`, `ui`*

---

### Is it possible to create a multi-layer global illumination plugin that discovers material metadata effects on other layers and properly communicates dependencies to After Effects' caching system?

This is a complex architectural question posed by fad regarding whether a 'renderer' effect can enumerate other layers, discover per-layer material definition effects, check out layer pixels/masks/transforms, and have those dependencies properly tracked by AE's dependency tracking and disk cache system. While Tobias Fleischer's experience suggests multi-layer effects are viable, the specific mechanism for communicating hidden dependencies (non-explicit UI parameter dependencies) to AE's caching system to trigger rerenders when materials change remains an open design challenge requiring careful SDK integration and likely custom dependency management.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `memory`, `render-loop`, `smartfx`*

---

### Can I use checkout_layer() during PF_Cmd_SMART_PRE_RENDER to access arbitrary composition layers not exposed as effect parameters?

According to maxon developer Alex Bizeau, you can only grab layer params directly during SMART_PRE_RENDER. However, you can use AEGP_RenderAndCheckoutLayerFrame with any layer IDs, and both sync and async versions exist. For multiple layers, a workaround is to use hidden layer parameters (up to 999) assigned via a button script that updates the comp, so checkout_layer() can access them. You may also handle dynamic layer order/removal by looping through the comp in SmartPreRender and mixing layer order into GuidMixInPtr() for dependency tracking.

*Tags: `aegp`, `caching`, `layer-checkout`, `params`, `smart-render`*

---

### How can I make a multi-layer effect that depends on material properties defined by effects on other layers while maintaining smart render efficiency?

The most robust approach is to use hidden layer parameters (you can have up to 999) and a button parameter that triggers a script to assign layers from the composition to those parameters one by one. Alternatively, loop through the comp in SmartPreRender and mix layer order into GuidMixInPtr() for dependency tracking. This allows the renderer effect to check out all layers efficiently while maintaining AE's caching and smart render system. You only need to click an 'Update Comp' button when you add layers; layer order changes or removals can be handled automatically by the SmartPreRender logic.

*Tags: `caching`, `compute-cache`, `layer-checkout`, `params`, `smart-render`*

---

### Can I pass a layer receipt GUID to GuidMixInPtr() to register layer dependencies without rendering pixels during SmartPreRender?

You can retrieve a GUID from an AEGP_FrameReceiptH using AEGP_GetReceiptGuid and pass it to extra->cb->GuidMixInPtr(). However, AEGP_RenderAndCheckoutLayerFrame is the expensive call that renders the entire layer, so calling it during SmartPreRender defeats the performance benefit. The suggested approach is to use layer parameters instead, which allow AE to track dependencies without rendering during the SmartPreRender phase.

*Tags: `aegp`, `caching`, `compute-cache`, `smart-render`*

---

### Is it possible to achieve progressive rendering in an After Effects plugin where samples render incrementally and display results to the user in real-time?

gabgren mentioned porting functionality from outside AE where progressive sampling over time with continuous result display is possible, making the interaction feel more responsive. They noted that the current AE plugin workflow requires waiting for all samples to render before seeing results on screen, which feels slow. They're exploring whether AE plugins can support this type of progressive rendering approach.

*Tags: `mfr`, `performance`, `render-loop`, `smart-render`, `ui`*

---

### How do 3D Camera Tracker and Warp Stabilizer output progress and live update without purging cache?

According to an After Effects developer, these built-in effects use internal APIs that are not available to plugin developers. The effects "cheat" and do not operate within the constraints of a normal AE Effect API plug-in. Developers have attempted workarounds involving refresh kicks, but these have proven unreliable across macOS and Windows. A feature request was made in 2023 to expose this capability to third-party developers, but it remains unavailable through the public API.

*Tags: `aegp`, `caching`, `cross-platform`, `debugging`, `render-loop`, `smart-render`*

---

### How can you specify a custom output region in Premiere transitions when smartFX is not available?

Premiere does not support smartFX, which means you cannot use the smart render feature to request only a specific region of the buffer. When making transitions, Premiere will provide frames at the sequence resolution regardless of the actual footage size. This is a limitation of Premiere's plugin architecture compared to After Effects, where smartFX allows you to define output rectangles and optimize rendering to only the needed buffer region.

*Tags: `output-rect`, `premiere`, `render-loop`, `smartfx`*

---

### Why does Premiere pass NULL for input_world in Render when compiled with fast optimizations, but not in After Effects?

A developer reported that when compiling a plugin with fast optimizations enabled, Premiere passes input_world = NULL in the Render call, whereas without optimizations the data is present. The same code does not exhibit this problem in After Effects. This suggests the issue may be related to optimization levels affecting how Premiere handles or passes buffer data to plugins during rendering.

*Tags: `build`, `debugging`, `premiere`, `render-loop`, `smartfx`*

---

### How do you properly expand the buffer in SmartFX to avoid asymmetric expansion?

When expanding the buffer via SmartFX, put the expanded buffer rect in `extra->output->result_rect` rather than relying on `max_result_rect`. After expanding the input buffer (`in_result.max_result_rect`), put the expanded result in `extra->output->max_result_rect`. After Effects uses the rect from `extra->output->result_rect` for the render call. The key is deciding what the expansion is relative to: the input buffer as-is (cropped by mask or comp bounds), the original layer size, or the max rect of the input. The expanded input buffer should be centered in the intermediate buffer that matches the expanded output.

```cpp
PF_LRect expanded_rect = in_result.max_result_rect;
expanded_rect.left -= expansion;
expanded_rect.top -= expansion;
expanded_rect.right += expansion;
expanded_rect.bottom += expansion;
extra->output->result_rect = expanded_rect;
extra->output->max_result_rect = expanded_rect;
```

*Tags: `buffer-expansion`, `output-rect`, `smart-render`, `smartfx`*

---

### Is there a working sample demonstrating buffer expansion in SmartFX?

Yes, shachar carmi created a working sample based on the 'smarty pants' sample that demonstrates buffer resizing in SmartFX. The sample is available at https://drive.google.com/file/d/1w4Fc9rtGXjU-YT4CC-6q01ft2MOvOLtt/view?usp=sharing. Additionally, an updated version that implements Gaussian blur with buffer expansion as the blur size grows is available at https://drive.google.com/file/d/1JWtvacj-jt5I8XfqxCzz3k_8lGi_nLaI/view?usp=sharing. These samples show how to properly structure the PreRender and SmartRender functions to handle expanded buffers correctly.

*Tags: `open-source`, `reference`, `smart-render`, `smartfx`*

---

### Where can you find an open-source directional blur plugin with buffer expansion implementation?

The AE-Motion repository on GitHub contains a Directional Blur plugin modified to use buffer expansion: https://github.com/NotYetEasy/AE-Motion/tree/main/DirectionalBlur. The repository includes all the plugin source code and can be compiled to test buffer expansion functionality in a real blur plugin implementation.

*Tags: `github`, `open-source`, `reference`, `smartfx`*

---

### What is the minimal code structure needed to implement smart render in an After Effects plugin?

The minimal smart render implementation requires a PreRender function that checks out the input layer with the correct channel mask and updates result rectangles, and a SmartRender function that checks out input and output buffers, handles parameter checkout, and processes the image according to bit depth (8, 16, or 32 bit). The code must properly union the result rectangles and always check in parameters even on error conditions.

```cpp
static PF_Err PreRender(PF_InData *in_data, PF_OutData *out_data, PF_PreRenderExtra *extra) {
  PF_Err err = PF_Err_NONE;
  PF_RenderRequest req = extra->input->output_request;
  PF_CheckoutResult in_result;
  req.channel_mask = PF_ChannelMask_ARGB;
  ERR(extra->cb->checkout_layer(in_data->effect_ref, EFFECT_INPUT, EFFECT_INPUT, &req, in_data->current_time, in_data->time_step, in_data->time_scale, &in_result));
  UnionLRect(&in_result.result_rect, &extra->output->result_rect);
  UnionLRect(&in_result.max_result_rect, &extra->output->max_result_rect);
  return err;
}

static PF_Err SmartRender(PF_InData *in_data, PF_OutData *out_data, PF_SmartRenderExtra *extra) {
  PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
  PF_EffectWorld *input_worldP = NULL, *output_worldP = NULL;
  ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, SHIFT_INPUT, &input_worldP));
  ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));
  if(input_worldP && output_worldP) {
    PF_ParamDef paramWhatever;
    AEFX_CLR_STRUCT(paramWhatever);
    ERR(PF_CHECKOUT_PARAM(in_data, EFFECT_PARAM, in_data->current_time, in_data->time_step, in_data->time_scale, &paramWhatever));
    switch (extra->input->bitdepth) {
      case 8: /* process 8 bit */ break;
      case 16: /* process 16 bit */ break;
      case 32: /* process 32 bit */ break;
    }
    ERR2(PF_CHECKIN_PARAM(in_data, &paramWhatever));
  }
  return err;
}
```

*Tags: `aegp`, `params`, `reference`, `smart-render`*

---

### How do you copy input to output without processing the image in a smart render effect?

Use the PF_COPY command to copy memory from input to output buffer. This command is independent of color depth and handles 8-bit, 16-bit, and 32-bit images automatically. The syntax is: ERR(PF_COPY(inputP, outputP, NULL, NULL));

```cpp
ERR(PF_COPY(inputP, outputP, NULL, NULL));
```

*Tags: `reference`, `smart-render`*

---

### Can iterate_generic be used to parallelize multiple transform_world calls across threads?

No, iterate_generic should not be used to call transform_world from utility threads. The iterate_generic suite uses utility threads that are separate from the main render threads, and AE API interaction is only safe from main threads. transform_world is internally multithreaded but is not designed to be called from non-main utility threads, which can result in bugs or crashes. Additionally, acquiring suites repeatedly in the iteration callback causes significant overhead. Instead, consider writing a custom transform implementation optimized for multi-threaded execution.

```cpp
suites.Iterate8Suite1()->iterate_generic(
  numThreads,
  &xform_data,
  MT_Xform
);

PF_Err MT_Xform(void* refcon, A_long threadInd, A_long iterNum, A_long iterTotal) {
  // Avoid calling transform_world or other AE suite functions from here
  // Pre-acquire suites in main thread instead
}
```

*Tags: `aegp`, `performance`, `render-loop`, `smartfx`, `threading`*

---

### Why is there no performance improvement when calling suite functions repeatedly inside iterate_generic callbacks?

Repeatedly acquiring suites inside iteration callbacks creates significant overhead that negates parallel execution benefits. Suite acquisition should be done once from the main thread before the iteration, and then pointers to the callbacks should be passed to the iteration function. Pre-acquiring suites can improve performance by orders of magnitude compared to acquiring them for every iteration.

*Tags: `caching`, `optimization`, `performance`, `smartfx`, `threading`*

---

### How can I access decoded video frames in a specific order for face detection in an After Effects plugin?

You can request any project item's image at any time using AEGP_RenderAndCheckoutFrame and AEGP_RenderSuite4. However, After Effects renders frames on-demand and in random order by design. For sequential processing requirements, consider two approaches: (1) fast sequential pre-processing upfront at the cost of user experience, or (2) slower random-access processing for simplicity. Examples like AE's camera tracker and Lockdown perform sequential background processing while keeping the UI responsive. You may also need to use AEIO_InqNextFrameTime() to determine frame order and build a custom data structure to track the sequence.

*Tags: `aegp`, `caching`, `render-loop`, `sequence-data`, `smart-render`, `ui`*

---

### What is the primary rendering philosophy of After Effects for plugin developers?

After Effects uses an on-demand, random-access rendering scheme rather than sequential frame processing. This design prioritizes user experience by rendering only requested frames in any order. Plugins that need sequential processing must work against this architecture, but can implement background processing with progress indication (like the camera tracker) to maintain UI responsiveness while performing pre-processing tasks.

*Tags: `caching`, `render-loop`, `smart-render`, `ui`*

---

### How do I write the generate_key function for the compute cache API without access to in_data?

The generate_key function doesn't have an in_data pointer, but you only need SPBasicSuite to acquire suites, not the full in_data. You have two options: (1) store SPBasicSuite in a global variable (shown in some SDK samples), or (2) use the cleaner approach of passing SPBasicSuite through the AEGP_CCComputeOptionsRefconP struct that all compute cache callbacks receive. If you want to use suite handling macros, create a local copy of the in_data struct and assign the passed SPBasicSuite value to its pica_basicP member. Do not pass in_data itself as it's only valid for the duration of a single AE call.

```cpp
PF_Err AEFX_AcquireSuite( PF_InData *in_data, /* >> */
PF_OutData *out_data, /* >> */
const char *name, /* >> */
int32_t version, /* >> */
const char *error_stringPC0, /* >> */
void **suite) /* << */
{
  PF_Err err = PF_Err_NONE;
  SPBasicSuite *bsuite;
  bsuite = in_data->pica_basicP;
  if (bsuite) {
    (*bsuite->AcquireSuite)((char*)name, version, (const void**)suite);
    if (!*suite) {
      err = PF_Err_BAD_CALLBACK_PARAM;
    }
  }
  return err;
}
```

*Tags: `aegp`, `compute-cache`, `mfr`, `smartfx`*

---

### Will changing non-PUI_ONLY parameters trigger an automatic re-render in After Effects?

Yes, modifying any parameter that is not marked as PUI_ONLY should automatically trigger a re-render. If a re-render is not occurring after parameter changes via SetStreamValue, verify that the modified parameters are not PUI_ONLY flags.

*Tags: `params`, `render-loop`, `smart-render`*

---

### If SmartFX is implemented in a 32-bit After Effects plugin, do I still need to implement the Render() event?

If SmartFX is implemented, After Effects does not send RENDER or FRAME_SETUP calls anymore; instead it sends Smart Render and Pre-render calls. However, if you need Premiere Pro compatibility, you should keep both mechanisms implemented since Premiere Pro does not use SmartFX and still calls Render and Frame Setup events.

*Tags: `aegp`, `premiere`, `render-loop`, `smartfx`*

---

### Why does my C++ plugin show the wrong initial frame when loading, and how do I fix it?

The issue is likely caused by AE's dynamic resolution feature, which automatically reduces the composition's resolution to maintain interactive speeds. You can verify this by disabling it in the composition window settings (lightning icon). However, the real solution is to implement smartFX and report a GUID during SmartPreRender to tell AE when a frame differs from what it has cached. Additionally, you must account for the downsample factors passed in the in_data struct, which is common practice for AE plugin developers.

*Tags: `caching`, `debugging`, `render-loop`, `smartfx`*

---

### How does sequence data get synchronized between the UI thread and render thread in Smart Rendering?

Sequence data synchronization between UI and render threads happens at specific times only: when a new instance is created and during specific UI events. Synchronization occurs when PF_Cmd_USER_CHANGED_PARAM is triggered, and in CLICK and DRAG events if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented. The mechanism involves PF_Cmd_GET_FLATTENED_SEQUENCE_DATA and PF_Cmd_SEQUENCE_RESETUP for flattening and unflattening respectively. However, PreRendering and SmartRendering do not automatically trigger this flattening. For reliable data persistence, Adobe recommends using invisible arb parameters which sync on every change, or implementing a message-passing mechanism through global data structures accessible to both threads.

*Tags: `arb-data`, `render-loop`, `sequence-data`, `smart-render`, `threading`*

---

### What is the best way to persist custom objects from the UI to the rendering thread in After Effects plugins?

The recommended approach is to use invisible arb parameters, which create undo entries and provide immediate invalidation of cached frames on changes. Alternatively, if arb parameters don't fit your design, you can devise a message-passing mechanism by placing instructions in a global data structure common to both threads, with the render thread checking for messages. Direct use of sequence data flattening outside of USER_CHANGED_PARAM or UI click/drag events is not reliable and should be avoided, as it can lead to sync issues with SmartRendering and PreRendering.

*Tags: `arb-data`, `params`, `sequence-data`, `smart-render`, `threading`*

---

### What other techniques can help trigger re-renders when external layer properties change in AEGP plugins?

Use GuidMixInPtr to force in external factors, set 'non param vary' in global setup, or consider implementing SmartFX for more sophisticated dependency tracking. Note that GuidMixInPtr may require SmartFX implementation and additional tweaking.

*Tags: `aegp`, `mfr`, `params`, `smartfx`*

---

### How can you adjust the level of motion blur when using transform_world in a plugin?

According to shachar carmi, transform_world spreads 16 instances across provided matrices to create motion blur. The level of motion blur can be adjusted in three ways: (1) by controlling the length of the blur (shutter angle), (2) by changing the number of samples in the blur, or (3) by adjusting the mix between the motion-blurred result and non-motion-blurred render. Providing more matrices gives a more refined look with better rotations and curved paths, though it doesn't increase the total number of samples. With 2 matrices you get 16 linear positions between them; with 3 matrices you get 8 samples from mat1 to mat2 and another 8 from mat2 to mat3.

*Tags: `motion-blur`, `params`, `premiere`, `smartfx`, `transform_world`*

---

### Why does applying a buffer-expanding effect like blur after a SmartFX effect cause an offset in the output?

When using SmartFX to expand the output buffer to the full composition size, subsequent buffer-expanding effects (such as blur or drop shadow) can alter the output origins. To fix this, you need to explicitly set the output->origin_x and output->origin_y parameters in your SmartFX preRender function. This ensures that downstream effects don't inadvertently offset your effect's output.

```cpp
UnionLRect(&req.rect, &extra->output->result_rect);
UnionLRect(&req.rect, &extra->output->max_result_rect);
// Also set:
extra->output->origin_x = /* appropriate value */;
extra->output->origin_y = /* appropriate value */;
```

*Tags: `buffer`, `output-rect`, `sdk`, `smartfx`*

---

### Can an After Effects plugin be used as-is in Premiere Pro?

Yes, an AE plugin can be used in Premiere Pro if it meets two criteria: (1) it supports the old 'render' and 'frame setup' calls, not only SmartFX 'smart render' and 'pre-render' calls (since Premiere doesn't support SmartFX), though a plugin can support both in parallel; (2) it doesn't use any AEGP suites, which Premiere doesn't support. Additionally, Premiere Pro requires support for BGRA pixel format in addition to AE's ARGB format.

*Tags: `aegp`, `cross-platform`, `premiere`, `smartfx`*

---

### When should I use PreRender and SmartRender in After Effects plugins, and what are their benefits?

PreRender (PF_Cmd_SMART_PRE_RENDER) and SmartRender (PF_Cmd_SMART_RENDER) are supported only in After Effects and are mandatory if you wish to support 32bpc (32-bit per channel) rendering. They also offer rendering pipeline improvements. However, if your plugin needs to work in Premiere Pro as well, you must support the old rendering pipeline with FrameSetup() and regular Render() calls instead.

*Tags: `ae`, `mfr`, `reference`, `render-loop`, `smart-render`*

---

### Is there a sample plugin that demonstrates PreRender and SmartRender implementation?

The Shifter sample plugin included in the After Effects SDK demonstrates PreRender and SmartRender usage.

*Tags: `open-source`, `reference`, `sample`, `smart-render`*

---

### Why does my After Effects plugin display a white screen after parameter changes, requiring a purge to render correctly?

This is typically a buffer clearing issue. You need to fill the output buffer with empty pixels using PF_FillMatteSuite2()->fill(). Another common cause is accidentally writing data to the input buffer and then using it as an intermediate processing buffer, which causes After Effects to cache the tampered data. Always clear AEGP worlds before using them, as After Effects may reuse the same RAM block from a previous world call, which could contain junk data or previous frame data.

```cpp
PF_FillMatteSuite2()->fill()
```

*Tags: `aegp`, `caching`, `memory`, `output-rect`, `smart-render`*

---

### Should I use AEGP_World for floating-point processing in After Effects plugins?

It is not necessary to use AEGP_World for 32-bit floating-point processing. Even in non-smart plugins that only support 8 and 16-bit, you can create a 32-bit PF_EffectWorld internally for processing in floating point and avoid the complexity and potential issues associated with AEGP worlds.

*Tags: `aegp`, `memory`, `smart-render`*

---

### How can you keep UI and render threads in sync when automatically applying multiple effects in CC2015+?

When using Sequence Setup to automatically add effects to a layer, the UI and render copies of the project can become out of sync in CC2015+, causing verification failures and missing data effects. The solution is to use an AEGP (After Effects General Plug-in) with an idle hook instead of modifying the layer during Sequence Setup. The effect can set a flag when it's applied, and the AEGP's idle hook will then process the pending operation to add the additional effects. The AEGP can be bundled in the same binary as the effect plugins, but the AEGP's PiPL must be listed first in the bundle.

*Tags: `aegp`, `pipl`, `plugin-architecture`, `smart-render`, `threading`, `ui`*

---

### How do you get frame data like PF_world from sequence_data in After Effects plugins?

sequence_data is a PF_Handle that you should cast to whatever structure you're using. Since you know the structure you're working with, you can cast the handle directly to access the corresponding memory region. The key distinction is between smartFX and non-smartFX effects: non-smartFX effects get input images through params[0]->u.ld, while both types can use PF_CHECKOUT_PARAM() to specify source times. smartFX effects use checkout_layer_pixels() instead. For any source time checkout, you must set the PF_OutFlag_WIDE_TIME_INPUT flag. For accessing pixels outside layer params, use AEGP_GetReceiptWorld().

*Tags: `layer-checkout`, `params`, `sequence-data`, `smartfx`*

---

### How do you output the current frame number being rendered in an After Effects plugin using printf?

When accessing the current frame in a PF_Cmd_SMART_RENDER command, use the correct printf format specifier. The in_data->current_time field is an integer, so use %i instead of %s. Using %s (string format) will cause undefined behavior, including null values, blank output, and corrupted data like random ASCII characters appearing in the output.

```cpp
printf("\nRendering frame %i", in_data->current_time);
```

*Tags: `aegp`, `debugging`, `smart-render`*

---

### In what order does After Effects render frames when using the Render button?

After Effects does not necessarily render frames in sequential order when you hit the Render button. The application uses an internal algorithm to determine which frames should be rendered based on what is already cached in memory, which can result in non-sequential frame rendering order.

*Tags: `caching`, `render-loop`, `smart-render`*

---

### How do you get input layer data with effects applied during an arbitrary draw event in After Effects?

Use extra->cb->checkout_layer_pixels() instead of PF_CHECKOUT_PARAM(), as the latter only fetches the layer's source without previous effects. However, checkout_layer_pixels() requires special suites available only during smart render calls and cannot be called during UI events. A better approach is to cache histogram data in sequence data during render calls and use PF_GetCurrentState and PF_HasParamChanged during draw events to detect parameter changes, then force re-render with PF_OutFlag_FORCE_RERENDER.

```cpp
typedef struct {
  bool didRender;
  /* histogram data */
} Histogram;
typedef struct {
  Histogram* histograms;
} my_sequence_data, *my_sequence_dataP, **my_sequence_dataH;

/* During render, cache at current time */
sequence->histograms[in_data->current_time / in_data->time_step]

/* During draw event, detect changes */
if (PF_HasParamChanged(in_data, extra)) {
  event_extraP->evt_out_flags |= PF_OutFlag_FORCE_RERENDER;
}
```

*Tags: `aegp`, `caching`, `sequence-data`, `smart-render`, `ui`*

---

### How do you force After Effects to re-render when custom UI parameters change but the effect's cached result is stale?

Use PF_GetCurrentState and PF_HasParamChanged during a draw event to detect if the plug-in's parameter state has changed since the last draw call. If parameters have changed, set the evt_out_flags to include PF_OutFlag_FORCE_RERENDER to force a fresh render. Simply setting change_flags or PF_EO_HANDLED_EVENT alone is not sufficient to overcome AE's caching behavior.

*Tags: `caching`, `params`, `smart-render`, `ui`*

---

### How do I properly handle AEGP_EffectRefH and AEGP_StreamRefH disposal when SmartRender operations are interrupted?

Every Get() call must be balanced by a Dispose() call, even when render is interrupted. The key issue is that if an interrupt causes your error variable to have a value, all commands wrapped in the ERR() macro will be skipped, preventing proper disposal. Solution: either set an interrupt flag and nullify the err var (remembering to return the canceled code to AE), or use ERR2() for interrupt checks. Additionally, ensure PF_ABORT is called during iteration and proper disposal happens before returning on abort.

```cpp
if (err = PF_ABORT(in_data)){
  if(outputWorld) (suites.WorldSuite3()->AEGP_Dispose(outputWorld));
  if(effectWorld) (suites.WorldSuite3()->AEGP_Dispose(effectWorld));
  if(streamH) (suites.StreamSuite4()->AEGP_DisposeStream(streamH));
  if(effectH) (suites.EffectSuite2()->AEGP_DisposeEffect(effectH));
  return err;
}
```

*Tags: `aegp`, `debugging`, `memory`, `smart-render`*

---

### What causes incomplete rendering when stacking multiple plugin instances and reloading a project?

The issue is likely caused by confused world buffer handling, especially with temporary buffers that might get overwritten by stacked effects. Carefully trace world creation to ensure input, output, and temporary buffers are not confused. A specific trap: temporary world buffers obtained from AE may actually point to previous frame output buffers in memory. Solution: clean buffer pixels before using temporary buffers, or ensure proper buffer allocation and usage patterns.

*Tags: `caching`, `debugging`, `memory`, `smart-render`*

---

### How can I determine the bit depth and pixel format of input/output images in an After Effects plugin?

The easiest way to get the bit depth is using `extra->input->bitdepth` in the smart_render call, which returns 8, 16, or 32. Alternatively, use the PF_WorldSuite2 to call `PF_GetPixelFormat()` on the world. After Effects always uses ARGB pixel format. The pixel format enum returns values like PF_PixelFormat_ARGB128 or PF_PixelFormat_ARGB.

```cpp
PF_WorldSuite2 *wsP = NULL;
ERR(suites.Pica()->AcquireSuite(kPFWorldSuite, kPFWorldSuiteVersion2, (const void**)&wsP));
ERR(wsP->PF_GetPixelFormat(inputP, pixelFormat));
ERR(suites.Pica()->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2));
```

*Tags: `mfr`, `params`, `reference`, `sdk`, `smart-render`*

---

### How can I make my After Effects plugin support multiple bit depths instead of just 8-bit?

You cannot explicitly restrict AE to only process at specific bit depths through plugin settings. Instead, check the bitdepth at runtime using `extra->input->bitdepth`, and if the bitdepth is not supported, copy the input to output without processing and inform the user via a message overlay or alert that the effect requires a specific bit depth.

*Tags: `params`, `sdk`, `smart-render`, `ui`*

---

### What are the differences between PF_CHECKOUT_PARAM and the checkout_layer/checkout_layer_pixels methods for accessing layer parameters in After Effects plugins?

Both methods are functionally correct but differ in important ways. The checkout_layer/checkout_layer_pixels method (used with PreRender and Render callbacks) is more resource-efficient as it allows smartFX to plan ahead and use RAM and buffers more efficiently for the entire rendering process. However, smartFX is not supported by all hosts, so if portability is important, this should be considered. A key difference: when checking out the same layer you're on, PF_CHECKOUT_PARAM gives a pre-effects image, while checkout_layer_pixels gives a post-effects image (including previous effects in the stack), even when sampling times other than the current render frame. For float images, the smartFX checkout_layer/checkout_layer_pixels combination is required.

```cpp
// Method 1: PF_CHECKOUT_PARAM
PF_ParamDef def;
AEFX_CLR_STRUCT(def);
ERR(PF_CHECKOUT_PARAM(in_data, layerID, in_data->current_time, in_data->time_step, in_data->time_scale, &(def)));
PF_EffectWorld myLayer = def.u.ld;
ERR2(PF_CHECKIN_PARAM(in_data, &(def)));
PF_COPY(&myLayer, output, NULL, NULL);

// Method 2: checkout_layer with PreRender callback
extraP->cb->checkout_layer(in_dataP->effect_ref, layerID, layerID, &req, in_dataP->current_time, in_dataP->time_step, in_dataP->time_scale, &ck_result);

// Method 2: checkout_layer_pixels with Render callback
PF_EffectWorld* myLayer;
ERR(extraP->cb->checkout_layer_pixels(in_data->effect_ref, layerID, &myLayer));
PF_COPY(myLayer, output, NULL, NULL);
extraP->cb->checkin_layer_pixels(in_data->effect_ref, layerID);
```

*Tags: `caching`, `layer-checkout`, `memory`, `params`, `smartfx`*

---

### How can an effect force full resolution rendering when the user has downsampled the composition?

Use the RenderAndCheckoutFrame AEGP function to render at full resolution. However, be careful to render the composition your effect is applied IN (not TO) to avoid infinite loops. You can check the downsample factors using AEGP_GetDownsampleFactor and conditionally render full resolution only when downsampling is detected. Make sure to use AEGP_GetEffectLayer instead of AEGP_GetActiveLayer to get your effect's layer, as 'active' doesn't necessarily refer to your layer.

```cpp
if (1 != in_data->downsample_x.num || 1 != in_data->downsample_x.den || 1 != in_data->downsample_y.num || 1 != in_data->downsample_y.den) {
  AEGP_ItemH itemH = NULL;
  AEGP_LayerH layerH = NULL;
  suites.EffectSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
  // Set downsample factor to 1,1 for full resolution
  suites.RenderOptionsSuite2()->AEGP_SetDownsampleFactor(roH, 1, 1);
  suites.RenderSuite2()->AEGP_RenderAndCheckoutFrame(roH, NULL, NULL, &receiptH);
}
```

*Tags: `aegp`, `layer-checkout`, `memory`, `smart-render`*

---

### Why am I getting black pixels (0,0,0,0 RGBA) when rendering a full resolution frame with RenderAndCheckoutFrame?

The issue is likely that you're using AEGP_GetActiveItem and AEGP_GetActiveLayer, which don't necessarily refer to your effect's layer. Use AEGP_GetEffectLayer instead, which is guaranteed to return your effect's actual layer. The black pixels suggest the render is not finding the correct layer to render.

*Tags: `aegp`, `debugging`, `memory`, `smart-render`*

---

### How do I properly update color parameters in nested precomposition layers from an effect plugin?

When modifying color values in fill effects within precomposition layers, use AEGP_GetEffectLayer instead of AEGP_GetActiveLayer to ensure you're working with the correct effect layer. Do not call AEGP_GetNewStreamValue before AEGP_SetStreamValue unless you specifically need to read the value first—if you do read it, you must dispose of it with AEGP_DisposeStreamValue. Implement proper parameter supervision during PF_Cmd_USER_CHANGED_PARAM callbacks and ensure you're correctly updating only the affected parameters rather than all of them.

```cpp
AEGP_LayerH effectLayer = NULL;
suites.LayerSuite5()->AEGP_GetEffectLayer(pluginID, effectRefH, &effectLayer);
AEGP_ItemH item = NULL;
suites.LayerSuite5()->AEGP_GetLayerSourceItem(effectLayer, &item);
AEGP_CompH preComp;
suites.CompSuite6()->AEGP_GetCompFromItem(item, &preComp);
AEGP_StreamRefH streamRef;
suites.StreamSuite3()->AEGP_GetNewEffectStreamByIndex(pluginID, eff, 3, &streamRef);
AEGP_StreamValue2 val;
val.val.color.alphaF = params[paramIndex]->u.cd.value.alpha;
val.val.color.redF = params[paramIndex]->u.cd.value.red;
val.val.color.greenF = params[paramIndex]->u.cd.value.green;
val.val.color.blueF = params[paramIndex]->u.cd.value.blue;
suites.StreamSuite3()->AEGP_SetStreamValue(pluginID, streamRef, &val);
```

*Tags: `aegp`, `params`, `smartfx`, `ui`*

---

### How can you re-apply a mask after modifying the alpha channel in an After Effects plugin?

There are two main approaches: (1) Use PF_MaskWorldWithPath on each mask and composite them in the correct transfer mode to recreate the layer's mask matte. Note that PF_MaskWorldWithPath does not support mask expansion, so you must render expanded masks manually. Read feather, opacity, and other mask values using AEGP_GetNewMaskStream or AEGP_GetNewMaskOpacity, then access numeric values with AEGP_GetNewStreamValue. (2) A simpler alternative: checkout the layer's original pixels (without masks applied) using a hidden layer parameter and PF_CHECKOUT_PARAM (or checkout_layer for smartFX), then compare the alpha difference between the original and masked input pixels to derive the mask matte. The second method is easier but less precise than fully recreating masks.

*Tags: `aegp`, `layer-checkout`, `mfr`, `params`, `smartfx`*

---

### How can I render only a single color channel (R, G, B, or A) from a PF_EffectWorld?

There are several approaches to isolate and display a single channel from a PF_EffectWorld without subpixel sampling. The iteration suite is recommended as it is multi-threaded and easy to manage—you iterate through input and output buffers and selectively zero out unwanted channels while preserving the desired one. Alternatively, you can manually access the world's base address and directly manipulate individual PF_Plane channels in memory for maximum efficiency, though this sacrifices multi-threading support. A compositing approach can also work: create a new world filled with the desired color and composite it using composite_rect with multiply transfer mode to suppress other channels. If using SmartFX, you can declare which channels to use and let AE ignore the others, though this approach was noted as potentially risky.

```cpp
outP->alpha = inP->alpha;
outP->red = 0;
outP->green = 0;
outP->blue = inP->blue;
```

*Tags: `aegp`, `memory`, `render-loop`, `smartfx`*

---
