# Q&A: smartfx

**59 entries** tagged with `smartfx`.

---

## What are the benefits of SmartFX (SmartRender) over regular Render?

SmartFX: (1) Only requests pixels needed for the specific processing area via PreRender. (2) Required for 32bpc support. (3) Allows checking out frames with prior effects applied via checkout_layer_pixels. (4) Supports MFR (Multi-Frame Rendering) better. (5) Provides more control over buffer management. Even if your effect needs the full image, SmartFX is recommended for 32bpc support and future-proofing.

*Contributors: [**fad**](../contributors/fad/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2024-06-10 · Tags: `smart-render`, `smartfx`, `render`, `32bpc`, `pre-render`*

---

## Does Premiere support SmartFX for requesting only a portion of the input buffer?

Premiere does not support SmartFX. You cannot officially request only a specific region of the buffer like you can in AE. As a workaround, you can manually iterate through the data using row bytes to delimit the part you need. Be careful with negative row byte values in Premiere.

*Contributors: [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2023-07-13 · Tags: `premiere`, `smartfx`, `buffer`, `region-of-interest`, `row-bytes`, `transition`*

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

## Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Contributors: [**Mirza Kadic (EFEKT)**](../contributors/mirza-kadic-efekt/), [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2025-07-10 · Tags: `smartfx`, `prerender`, `smartrender`, `code-organization`, `cpp`, `dll`*

---

## How do you generate unique checkout IDs for checkout_layer/checkout_layer_pixels in SmartFX with multiple layers and instances?

This is a known challenge when dealing with multiple layers, multiple instances, and nested loops in SmartFX PreRender/SmartRender. The checkout ID must be unique across all checkouts. No definitive solution was provided in the discussion, but the issue typically arises with many cloned plugin instances resulting in 'checkout id is not unique' errors.

*Source: aescripts discord · 2025-09-13 · Tags: `smartfx`, `checkout-layer`, `unique-id`, `prerender`, `smartrender`, `multiple-instances`*

---

## How do you cache frames in an After Effects effect plugin and use the cached result as input for processing the next frame?

This requires implementing frame caching within your plugin's render loop. You need to store the output of frame N, then retrieve it to use as input for frame N+1. This typically involves maintaining a persistent buffer or cache structure across multiple render calls. The exact implementation depends on whether you're using SmartFX or legacy plugins, but generally involves allocating memory to store intermediate results and managing that cache through the plugin's lifecycle to ensure the previous frame's output is available when processing subsequent frames.

*Tags: `caching`, `render-loop`, `memory`, `smartfx`, `output-rect`*

---

## How do you expand the output buffer with smartFX to prevent outlines from being cut off by the layer bounding box?

In smartFX, you need to modify the output rectangle to be larger than the input layer bounds. This is typically done by adjusting the output_rect parameter in your render function to expand beyond the original layer boundaries, allowing your outline effect to render fully without clipping.

*Tags: `smartfx`, `output-rect`, `render-loop`*

---

## Do you need to use AEGP for adding time values in the compute cache API?

The conversation indicates this is a question about the proper API approach for time value handling in the compute cache, but a definitive answer was not provided in the chat.

*Tags: `compute-cache`, `aegp`, `smartfx`*

---

## How can you access the previous frame result during the compute thread in After Effects?

The user asked this question but indicated that despite reading the documentation, a full example would be more helpful. No complete answer was provided in the conversation.

*Tags: `compute-cache`, `smartfx`, `render-loop`, `memory`*

---

## How can I check out the current layer (INPUT) with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT on the INPUT layer at different times, it returns the effectworld before all effects are applied, even those that precede the current effect in the chain. To get the current layer at different times with all effects applied, you need to use a different approach than simple PF_CHECKOUT. The Echo effect demonstrates that this is possible, likely through using layer composition or render callbacks that account for the full effect stack.

*Tags: `layer-checkout`, `render-loop`, `smartfx`, `params`*

---

## Can you use AEGP render suite during the render call, or only during PreRender?

You cannot use AEGP render suite during the render call itself. However, you can use it during PreRender to get rendered frames at a given time with the previous effect reference. Alternatively, you can use smart render with frame preloading in pre-render, using the same method as the current frame.

*Tags: `aegp`, `smart-render`, `render-loop`, `smartfx`*

---

## Can arb data parameter values be accessed in paramssetup?

Arb data can be accessed in paramssetup using the standard access functions (similar to how it's accessed in other functions like smartrender), allowing you to read arbP member variables.

*Tags: `arb-data`, `params`, `smartfx`*

---

## When do I need to implement PF_Cmd_FRAME_SETUP to customize output size?

You only need to implement PF_Cmd_FRAME_SETUP for non-SmartFX effects if your effect expands the output buffer size, such as with glow or drop shadow effects. For effects that maintain the same output dimensions as the input, this command is not necessary.

*Tags: `mfr`, `smartfx`, `output-rect`*

---

## Does smart render provide any benefit for effects that require the full image each time it is processed?

Smart render helps by only getting pixels required for specific processing. For effects that require the full image each time it is processed, smart render would not provide additional benefit since the entire image must be obtained regardless.

*Tags: `smart-render`, `smartfx`, `output-rect`*

---

## What is required to support 32 bits per channel (32 bpc) in an After Effects effect?

You must use SmartFX if you want your effect to support 32 bpc.

*Tags: `smartfx`, `params`*

---

## Is SmartFX the only way to detect if an input frame actually changed between renders?

SmartFX provides built-in mechanisms to detect frame changes, but it's not necessarily the only way. You can also implement custom change detection by comparing frame data or checksums, monitoring layer properties (scale, position, effects), and using the render queue system to track what has actually changed. SmartFX simplifies this by providing smart rendering capabilities that automatically optimize based on detected changes.

*Tags: `smartfx`, `smart-render`, `render-loop`*

---

## Is SmartFX the only way to detect if an input frame actually changed?

No, SmartFX is not the only way. You can use PF_GetCurrentState to tell if an input has changed. Additionally, if an input frame changes you should certainly get a re-render. If you need to force another prerender for other changes, you can mix anything else into the guid mix in ptr.

*Tags: `smartfx`, `render-loop`, `smart-render`*

---

## How are transforms applied to input buffers in the dumb pipeline versus SmartFX?

In SmartFX, if the layer is continuously rasterised, you receive the input buffer with transforms already applied. If it's not continuously rasterised, the transforms are applied after your effect populates the output buffer. In the dumb pipeline, transforms are applied after the effect populates the output buffer.

*Tags: `smartfx`, `render-loop`, `output-rect`*

---

## How do you check out a frame of the current layer after prior effects are applied instead of before them?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects. However, this approach may not work in Premiere Pro. An alternative approach using Cmd_RENDER may be needed for cross-application compatibility.

*Tags: `layer-checkout`, `smartfx`, `premiere`, `render-loop`*

---

## How difficult would it be to match After Effects' lighting calculations in a custom 3D plugin?

This is a challenging task because it requires matching AE's specific lighting parameters and their meanings, such as spotlight falloff softness and the balance between ambient and direct lighting. Many plugins have implemented this, though the effort is significant.

*Tags: `gpu`, `opengl`, `metal`, `smartfx`*

---

## Why does SmartRender never get called in Release builds even when PreRender returns PF_Err_NONE?

The issue was caused by a mismatch between the SDK code version used for building (2025.2) and the SDK headers (2023). The debug configuration worked fine because it didn't have this version mismatch. Ensure that both the SDK code and headers are from the same version when building.

*Tags: `smartfx`, `smart-render`, `debugging`, `build`, `release`*

---

## What is the exact behavior of the GuidMixInPtr callback from extra->cb in PreRender?

The GuidMixInPtr callback in PreRender can return a result that indicates whether a new render is needed or not. However, using or disabling this callback alone may not resolve SmartRender not being called if there are other underlying issues like SDK version mismatches.

```cpp
extra->cb->GuidMixInPtr
```

*Tags: `smartfx`, `smart-render`, `params`*

---

## Is there a method for forcing masks on a layer to be applied after the plugin processes?

No answer was provided in the conversation.

*Tags: `smartfx`, `render-loop`, `layer-checkout`*

---

## How can you limit the output buffer to a specific region when Premiere doesn't support smartFX for transitions?

Since Premiere Pro does not support smartFX like After Effects does, you cannot use the smartFX output_rect mechanism to specify a smaller region of interest. When building transitions for Premiere, the plugin receives frames at the full sequence resolution even if the source footage is smaller. You need to work with the full buffer dimensions provided by Premiere's API rather than requesting a subset.

*Tags: `smartfx`, `premiere`, `output-rect`, `transition`*

---

## Why does Premiere pass NULL for input_world in Render calls when compiled with fast optimizations, while After Effects doesn't have this issue?

When a plugin is compiled with fast optimizations, Premiere may pass input_world = NULL in the Render call, whereas the same code without optimizations receives the data correctly. This issue does not occur in After Effects. The root cause appears to be related to compiler optimization levels affecting how Premiere handles input data passing to plugins.

*Tags: `premiere`, `debugging`, `render-loop`, `smartfx`*

---

## How should you store GPU and CPU data across multiple frames in After Effects plugins?

For GPU work, store data in global data or sequence data and access it later. For CPU work, cache the PF_EffectWorld in the global or sequence data. This approach may need validation against the new 3-way-checkout mechanism.

*Tags: `gpu`, `caching`, `sequence-data`, `memory`, `smartfx`*

---

## How do you expand the output buffer with smartFX to prevent outlines from being clipped by layer bounds?

When using smartFX in After Effects, if you're generating outlines around an alpha image and they're getting cut off by the layer bounding box, you need to expand the output buffer. This is typically done by modifying the output rectangle in your smartFX implementation to be larger than the layer's current bounds, ensuring that generated content (like outlines) isn't clipped.

*Tags: `smartfx`, `output-rect`, `mfr`, `render-loop`*

---

## What needs to be updated when using PF_COPY or buffer operations in a plugin?

When performing buffer operations like PF_COPY, the source and destination rectangles (src and dst rects) must be updated to reflect the actual regions being operated on.

*Tags: `smartfx`, `output-rect`, `render-loop`*

---

## Why does a plugin render get interrupted with PF_Interrupt_CANCEL during queue rendering?

PF_Interrupt_CANCEL during render queue operations can occur under specific conditions. One user reported this happening after 10-15 frames when CAPS LOCK was off and the composition had not been previously previewed to RAM cache. The interrupt appears as 'User abort' but occurs without user action. The behavior did not occur when CAPS LOCK was on or when the composition was fully previewed in the RAM cache before rendering. This suggests the interrupt may be related to input handling or cache state during rendering.

*Tags: `render-loop`, `debugging`, `smartfx`*

---

## Why is checkout_layer_pixels returning NULL data for a layer parameter even though the layer is selected?

Tim Constantinov reported an issue where after calling checkout_layer_pixels on a layer selected in a layer parameter, the data field equals NULL. The problem occurs in SmartRender after checkout_layer_pixels is called on GPU_SKELETON_LAYER. The issue appears to be intermittent and may be related to the Adobe-supplied GPU template. The exact cause wasn't definitively identified in the conversation, but Tim noted this issue resolved itself spontaneously in the past.

```cpp
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `layer-checkout`, `gpu`, `smartfx`, `debugging`, `params`*

---

## How should a plugin handle GPU out-of-memory errors differently for interactive scrubbing versus render queue operations?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects handles it differently depending on context. During MFR (Multi-Frame Rendering) queue renders, AE may retry with a lower thread count. However, during interactive timeline scrubbing, AE doesn't warn the user and may display a black frame instead. The challenge is that PF_Err_OUT_OF_MEMORY warnings in out_data will fail the render, and there is currently no documented method to distinguish between a render queue request versus interactive scrubbing. Plugins must accept that they cannot reliably warn users about resolution/bitdepth being too high for available GPU memory in interactive contexts without failing the frame.

*Tags: `gpu`, `memory`, `mfr`, `debugging`, `smartfx`*

---

## How can I force After Effects to invalidate cached frames when I modify global data in my plugin?

Simply setting PF_OutFlag_FORCE_RERENDER in outflags while changing a GuidMixInPtr value in extra->cb may not always work reliably. One workaround is to "kick" AE by triggering an internal frame cache check, such as changing the composition background color and changing it back. However, if you properly use GuidMixInPtr for forcing rerenders, it should work without this workaround—if you're experiencing issues, it may be worth reviewing source code examples from developers who have successfully implemented this.

*Tags: `compute-cache`, `smartfx`, `aegp`, `params`*

---

## Does smart render provide benefits for effects that require the full image each time they process?

Smart render helps by only getting pixels required for the specific processing needed. For an effect that requires the full image each time it is processed (when visible), smart render may not provide additional benefits since the entire image must be fetched anyway.

*Tags: `smart-render`, `smartfx`, `render-loop`*

---

## Is SmartFX required for 32-bit per channel color depth support?

Yes, you must use SmartFX if you want your effect to support 32 bpc (32-bit per channel) color depth.

*Tags: `smartfx`, `params`*

---

## How can I detect if an input frame has actually changed in an After Effects plugin?

You can use PF_GetCurrentState to tell if an input has changed. Additionally, if an input frame changes you should certainly get a re-render. If you need to force another prerender, you can mix it into the guid mix in ptr.

*Tags: `params`, `render-loop`, `smartfx`, `layer-checkout`*

---

## How does transform application differ between SmartFX and standard pipelines in After Effects?

In SmartFX, if a layer is continuously rasterized, you receive the input buffer with transforms already applied. If not continuously rasterized, the transforms are applied after your effect populates the output buffer. This means you won't know if transforms changed in the standard pipeline unless using SmartFX.

*Tags: `smartfx`, `render-loop`, `mfr`, `output-rect`*

---

## How do you check out a frame of the current layer after prior effects are applied in After Effects?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects have been applied. This allows you to access the layer state after upstream effects like Exposure have been rendered, but before the current effect processes it. However, this approach may not work in Premiere Pro, requiring alternative solutions for cross-application compatibility.

*Tags: `layer-checkout`, `smartfx`, `render-loop`, `premiere`, `aegp`*

---

## What are the practical challenges and solutions when designing multi-layer effects in After Effects plugins?

According to Tobias Fleischer (reduxFX), a plugin-per-layer approach was abandoned in favor of a single plugin with multiple layer parameters due to render order troubles. The successful approach involved creating a single plugin that accepts 10 layer parameters to pull in pixels from multiple layers, applies lighting/material options, and composites everything together within that single plugin. While this worked well, AE's rigid parameter interface made it somewhat clunky to use. The key insight is that centralizing the multi-layer logic into a single plugin avoids render order dependency issues that arise from distributing logic across multiple per-layer plugins.

*Tags: `render-loop`, `params`, `ui`, `layer-checkout`, `smartfx`*

---

## Is it possible to create a multi-layer global illumination plugin that discovers material metadata effects on other layers and properly communicates dependencies to After Effects' caching system?

This is a complex architectural question posed by fad regarding whether a 'renderer' effect can enumerate other layers, discover per-layer material definition effects, check out layer pixels/masks/transforms, and have those dependencies properly tracked by AE's dependency tracking and disk cache system. While Tobias Fleischer's experience suggests multi-layer effects are viable, the specific mechanism for communicating hidden dependencies (non-explicit UI parameter dependencies) to AE's caching system to trigger rerenders when materials change remains an open design challenge requiring careful SDK integration and likely custom dependency management.

*Tags: `render-loop`, `caching`, `compute-cache`, `layer-checkout`, `smartfx`, `memory`*

---

## How can you specify a custom output region in Premiere transitions when smartFX is not available?

Premiere does not support smartFX, which means you cannot use the smart render feature to request only a specific region of the buffer. When making transitions, Premiere will provide frames at the sequence resolution regardless of the actual footage size. This is a limitation of Premiere's plugin architecture compared to After Effects, where smartFX allows you to define output rectangles and optimize rendering to only the needed buffer region.

*Tags: `smartfx`, `premiere`, `output-rect`, `render-loop`*

---

## Why does Premiere pass NULL for input_world in Render when compiled with fast optimizations, but not in After Effects?

A developer reported that when compiling a plugin with fast optimizations enabled, Premiere passes input_world = NULL in the Render call, whereas without optimizations the data is present. The same code does not exhibit this problem in After Effects. This suggests the issue may be related to optimization levels affecting how Premiere handles or passes buffer data to plugins during rendering.

*Tags: `premiere`, `smartfx`, `render-loop`, `debugging`, `build`*

---

## How do you properly expand the buffer in SmartFX to avoid asymmetric expansion?

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

*Tags: `smartfx`, `buffer-expansion`, `smart-render`, `output-rect`*

---

## Is there a working sample demonstrating buffer expansion in SmartFX?

Yes, shachar carmi created a working sample based on the 'smarty pants' sample that demonstrates buffer resizing in SmartFX. The sample is available at https://drive.google.com/file/d/1w4Fc9rtGXjU-YT4CC-6q01ft2MOvOLtt/view?usp=sharing. Additionally, an updated version that implements Gaussian blur with buffer expansion as the blur size grows is available at https://drive.google.com/file/d/1JWtvacj-jt5I8XfqxCzz3k_8lGi_nLaI/view?usp=sharing. These samples show how to properly structure the PreRender and SmartRender functions to handle expanded buffers correctly.

*Tags: `smartfx`, `smart-render`, `reference`, `open-source`*

---

## Where can you find an open-source directional blur plugin with buffer expansion implementation?

The AE-Motion repository on GitHub contains a Directional Blur plugin modified to use buffer expansion: https://github.com/NotYetEasy/AE-Motion/tree/main/DirectionalBlur. The repository includes all the plugin source code and can be compiled to test buffer expansion functionality in a real blur plugin implementation.

*Tags: `open-source`, `reference`, `smartfx`, `github`*

---

## Can iterate_generic be used to parallelize multiple transform_world calls across threads?

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

*Tags: `threading`, `render-loop`, `aegp`, `smartfx`, `performance`*

---

## Why is there no performance improvement when calling suite functions repeatedly inside iterate_generic callbacks?

Repeatedly acquiring suites inside iteration callbacks creates significant overhead that negates parallel execution benefits. Suite acquisition should be done once from the main thread before the iteration, and then pointers to the callbacks should be passed to the iteration function. Pre-acquiring suites can improve performance by orders of magnitude compared to acquiring them for every iteration.

*Tags: `threading`, `performance`, `caching`, `smartfx`, `optimization`*

---

## How do I write the generate_key function for the compute cache API without access to in_data?

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

*Tags: `compute-cache`, `aegp`, `mfr`, `smartfx`*

---

## If SmartFX is implemented in a 32-bit After Effects plugin, do I still need to implement the Render() event?

If SmartFX is implemented, After Effects does not send RENDER or FRAME_SETUP calls anymore; instead it sends Smart Render and Pre-render calls. However, if you need Premiere Pro compatibility, you should keep both mechanisms implemented since Premiere Pro does not use SmartFX and still calls Render and Frame Setup events.

*Tags: `smartfx`, `render-loop`, `premiere`, `aegp`*

---

## Why does my C++ plugin show the wrong initial frame when loading, and how do I fix it?

The issue is likely caused by AE's dynamic resolution feature, which automatically reduces the composition's resolution to maintain interactive speeds. You can verify this by disabling it in the composition window settings (lightning icon). However, the real solution is to implement smartFX and report a GUID during SmartPreRender to tell AE when a frame differs from what it has cached. Additionally, you must account for the downsample factors passed in the in_data struct, which is common practice for AE plugin developers.

*Tags: `smartfx`, `caching`, `render-loop`, `debugging`*

---

## What other techniques can help trigger re-renders when external layer properties change in AEGP plugins?

Use GuidMixInPtr to force in external factors, set 'non param vary' in global setup, or consider implementing SmartFX for more sophisticated dependency tracking. Note that GuidMixInPtr may require SmartFX implementation and additional tweaking.

*Tags: `aegp`, `smartfx`, `mfr`, `params`*

---

## How can you adjust the level of motion blur when using transform_world in a plugin?

According to shachar carmi, transform_world spreads 16 instances across provided matrices to create motion blur. The level of motion blur can be adjusted in three ways: (1) by controlling the length of the blur (shutter angle), (2) by changing the number of samples in the blur, or (3) by adjusting the mix between the motion-blurred result and non-motion-blurred render. Providing more matrices gives a more refined look with better rotations and curved paths, though it doesn't increase the total number of samples. With 2 matrices you get 16 linear positions between them; with 3 matrices you get 8 samples from mat1 to mat2 and another 8 from mat2 to mat3.

*Tags: `transform_world`, `motion-blur`, `smartfx`, `premiere`, `params`*

---

## Why does applying a buffer-expanding effect like blur after a SmartFX effect cause an offset in the output?

When using SmartFX to expand the output buffer to the full composition size, subsequent buffer-expanding effects (such as blur or drop shadow) can alter the output origins. To fix this, you need to explicitly set the output->origin_x and output->origin_y parameters in your SmartFX preRender function. This ensures that downstream effects don't inadvertently offset your effect's output.

```cpp
UnionLRect(&req.rect, &extra->output->result_rect);
UnionLRect(&req.rect, &extra->output->max_result_rect);
// Also set:
extra->output->origin_x = /* appropriate value */;
extra->output->origin_y = /* appropriate value */;
```

*Tags: `smartfx`, `buffer`, `output-rect`, `sdk`*

---

## Can an After Effects plugin be used as-is in Premiere Pro?

Yes, an AE plugin can be used in Premiere Pro if it meets two criteria: (1) it supports the old 'render' and 'frame setup' calls, not only SmartFX 'smart render' and 'pre-render' calls (since Premiere doesn't support SmartFX), though a plugin can support both in parallel; (2) it doesn't use any AEGP suites, which Premiere doesn't support. Additionally, Premiere Pro requires support for BGRA pixel format in addition to AE's ARGB format.

*Tags: `premiere`, `aegp`, `smartfx`, `cross-platform`*

---

## How do you get frame data like PF_world from sequence_data in After Effects plugins?

sequence_data is a PF_Handle that you should cast to whatever structure you're using. Since you know the structure you're working with, you can cast the handle directly to access the corresponding memory region. The key distinction is between smartFX and non-smartFX effects: non-smartFX effects get input images through params[0]->u.ld, while both types can use PF_CHECKOUT_PARAM() to specify source times. smartFX effects use checkout_layer_pixels() instead. For any source time checkout, you must set the PF_OutFlag_WIDE_TIME_INPUT flag. For accessing pixels outside layer params, use AEGP_GetReceiptWorld().

*Tags: `sequence-data`, `smartfx`, `layer-checkout`, `params`*

---

## What are the differences between PF_CHECKOUT_PARAM and the checkout_layer/checkout_layer_pixels methods for accessing layer parameters in After Effects plugins?

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

*Tags: `layer-checkout`, `params`, `smartfx`, `memory`, `caching`*

---

## How do I properly update color parameters in nested precomposition layers from an effect plugin?

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

*Tags: `aegp`, `params`, `ui`, `smartfx`*

---

## How can you re-apply a mask after modifying the alpha channel in an After Effects plugin?

There are two main approaches: (1) Use PF_MaskWorldWithPath on each mask and composite them in the correct transfer mode to recreate the layer's mask matte. Note that PF_MaskWorldWithPath does not support mask expansion, so you must render expanded masks manually. Read feather, opacity, and other mask values using AEGP_GetNewMaskStream or AEGP_GetNewMaskOpacity, then access numeric values with AEGP_GetNewStreamValue. (2) A simpler alternative: checkout the layer's original pixels (without masks applied) using a hidden layer parameter and PF_CHECKOUT_PARAM (or checkout_layer for smartFX), then compare the alpha difference between the original and masked input pixels to derive the mask matte. The second method is easier but less precise than fully recreating masks.

*Tags: `aegp`, `params`, `layer-checkout`, `mfr`, `smartfx`*

---

## How can I render only a single color channel (R, G, B, or A) from a PF_EffectWorld?

There are several approaches to isolate and display a single channel from a PF_EffectWorld without subpixel sampling. The iteration suite is recommended as it is multi-threaded and easy to manage—you iterate through input and output buffers and selectively zero out unwanted channels while preserving the desired one. Alternatively, you can manually access the world's base address and directly manipulate individual PF_Plane channels in memory for maximum efficiency, though this sacrifices multi-threading support. A compositing approach can also work: create a new world filled with the desired color and composite it using composite_rect with multiply transfer mode to suppress other channels. If using SmartFX, you can declare which channels to use and let AE ignore the others, though this approach was noted as potentially risky.

```cpp
outP->alpha = inP->alpha;
outP->red = 0;
outP->green = 0;
outP->blue = inP->blue;
```

*Tags: `aegp`, `render-loop`, `memory`, `smartfx`*

---
