# Q&A: render-loop

**236 entries** tagged with `render-loop`.

---

## How do you cache frames in an After Effects effect plugin and use the cached result as input for processing the next frame?

This requires implementing frame caching within your plugin's render loop. You need to store the output of frame N, then retrieve it to use as input for frame N+1. This typically involves maintaining a persistent buffer or cache structure across multiple render calls. The exact implementation depends on whether you're using SmartFX or legacy plugins, but generally involves allocating memory to store intermediate results and managing that cache through the plugin's lifecycle to ensure the previous frame's output is available when processing subsequent frames.

*Tags: `caching`, `render-loop`, `memory`, `smartfx`, `output-rect`*

---

## When should the first frame be stored and how do you physically store an EffectWorld in sequence data?

The first frame is stored in the render function. You can store an EffectWorld directly in your sequence_data struct. On the first render, allocate a new world using PF_NEW_WORLD and copy the result into it. On subsequent frames, you can reuse and update this cached world. Use a hash or flag to track whether the result has been loaded and whether parameters have changed.

```cpp
typedef struct {
    PF_EffectWorld  resultW;
    bool            result_loaded = false;
    A_long          result_last_time;
    char            result_hash[260];
} sequenceData;

if (!seqdataP->result_loaded) {
    AEFX_CLR_STRUCT(seqdataP->resultW);
    ERR(PF_NEW_WORLD(tensorW.width, tensorW.height, PF_NewWorldFlag_NONE, &seqdataP->resultW));
}
PF_COPY(&cachingW, &seqdataP->resultW, NULL, NULL);
seqdataP->result_loaded = true;
```

*Tags: `sequence-data`, `caching`, `memory`, `render-loop`*

---

## How do you access and write to sequence data within the Render function?

You can access sequence data in the Render function by dereferencing the sequence_data pointer provided in out_data. Cast it to your sequence data struct type. You can then read from and write to any members of that struct throughout the render call. However, MFR may have implications for this workflow, so test accordingly.

```cpp
sequenceData *seqdataP = NULL;
seqdataP = *(sequenceData **)out_data->sequence_data;
// Now you can read/write to seqdataP->resultW, seqdataP->result_loaded, etc.
```

*Tags: `sequence-data`, `render-loop`, `caching`*

---

## How do you expand the output buffer with smartFX to prevent outlines from being cut off by the layer bounding box?

In smartFX, you need to modify the output rectangle to be larger than the input layer bounds. This is typically done by adjusting the output_rect parameter in your render function to expand beyond the original layer boundaries, allowing your outline effect to render fully without clipping.

*Tags: `smartfx`, `output-rect`, `render-loop`*

---

## How do plugins like Trapcode Particular cache and calculate multiple frames over time rather than just the last frame?

Plugins that need to cache multiple frames typically use their own internal state management system separate from the standard Sequence Data approach. While Sequence Data can store the last frame, plugins handling fluid simulations or particle systems that need to jump to arbitrary times in a composition use custom caching mechanisms. These plugins maintain their own frame cache, calculate values for multiple frames, and store them in memory structures that can be accessed when jumping to different times in the composition. The exact implementation depends on the plugin's architecture, but it generally involves managing a larger state buffer that holds multiple frames of simulation data rather than relying solely on the sequential frame-to-frame Sequence Data mechanism.

*Tags: `caching`, `sequence-data`, `memory`, `render-loop`*

---

## What algorithm does the Matte/Simple Choker Effect use to find and create reduced/expanded edges on alpha channels?

The questioner describes their own implementation which analyzes each transparent pixel and looks in 8 directions to find nearby opaque pixels to identify outline/choker matte pixels. However, no answer was provided in the conversation about the actual method used by Adobe's Matte/Simple Choker Effect.

*Tags: `mfr`, `memory`, `gpu`, `render-loop`, `optimization`*

---

## What algorithm does the Matte/Simple Choker Effect use to find and create reduced/expanded edges in alpha channels?

The user describes their current implementation which analyzes each transparent pixel and looks in 8 directions to find nearby opaque pixels to identify outline/choker matte pixels. However, this approach is very slow (30+ seconds). The conversation does not provide a definitive answer about what method Adobe's Choker effect actually uses, only describes the user's current slow implementation.

*Tags: `mfr`, `memory`, `gpu`, `render-loop`, `optimization`*

---

## What algorithm does the Matte/Simple Choker Effect use to create reduced/expanded edges on alpha channels?

The Matte/Simple Choker Effect likely uses a distance field approach rather than checking neighboring pixels. A distance field uses an intermediate 2D buffer of singular distance values (such as int16_t for signed distance values) to exchange memory for speed. The Jump Flooding Algorithm (JFA) is a fast method to generate distance fields and is well-suited for this purpose, avoiding the O(n²) complexity of checking neighbors for each pixel.

*Tags: `memory`, `gpu`, `render-loop`, `optimization`*

---

## Why is the neighbor-checking algorithm for creating choker mattes too slow and what is a better approach?

The neighbor-checking approach is too slow because checking 8 directions for each transparent pixel to find nearby opaque pixels has O(n²) complexity when expanding outlines by larger amounts (e.g., 30 pixels). A distance field approach is significantly faster as it uses a 2D buffer of distance values to trade memory for speed, with Jump Flooding Algorithm being a recommended fast implementation.

*Tags: `memory`, `optimization`, `render-loop`*

---

## What flags or methods are used to create a caching effect where rendering waits for frame caching to complete before updating?

This is a technique used in higher-end plugins like Particular where moving the Current Time Indicator (CTI) causes the plugin to wait for frame caching to complete before updating the render. This involves using smart render techniques and potentially compute cache mechanisms to defer rendering until cache state is ready.

*Tags: `caching`, `smart-render`, `render-loop`, `compute-cache`*

---

## How can you make After Effects re-render a frame multiple times until a calculation is complete?

You need to call render() several times in a loop until your calculation is done. This requires a flag that tells AE to re-render the frame again, and inside the render function you check with an if clause whether the render is actually finished or not.

*Tags: `render-loop`, `smart-render`, `debugging`*

---

## Should error handling be done when checking out layers, and should check-in occur immediately after use?

Yes, error handling should be implemented for checkout operations. Check-in should be done immediately after getting the value. In a loop structure, you should checkout a frame, check for errors, perform the job if no error occurred, and then check-in the frame before moving to the next iteration. Each checked-out resource must be checked in, whether it was used or not.

```cpp
For X
     Checkout frame x
         If (no err) dojob
     Err2(check-in frame x)
End of loop
```

*Tags: `layer-checkout`, `aegp`, `error-handling`, `render-loop`, `threading`*

---

## How can you obtain a drawing reference for rendering without user interaction events in After Effects plugins?

The drawing reference is typically only available through PF_Cmd_EVENT when user interaction occurs. To draw during rendering without user interaction, you need an alternative approach. One solution is to store the drawing reference when it becomes available (during any event where PF_GetDrawingReference is called), and then reuse that stored reference in your render loop. However, be cautious about thread safety and the validity of the reference across different render contexts. Another approach is to use AEGP suites to directly access composition or layer drawing capabilities if available, rather than relying on the Effect Custom UI Suite's event-driven drawing reference.

```cpp
// Store drawing ref when available from PF_Cmd_EVENT
PF_DrawingReference stored_drawing_ref = NULL;

// In PF_Cmd_EVENT handler:
if (event_extraP) {
  suite.EffectCustomUISuite1()->PF_GetDrawingReference(
    event_extraP->contextH, 
    &stored_drawing_ref
  );
}

// Later, attempt to use in render loop (verify validity first)
if (stored_drawing_ref) {
  // Use stored_drawing_ref
}
```

*Tags: `ui`, `render-loop`, `custom-ui`, `debugging`*

---

## How can you get a drawing reference from within the render loop without relying on user interaction events?

The drawing reference is typically obtained through PF_GetDrawingReference in the PF_Cmd_EVENT handler via the event_extraP context, which is only available when the user interacts with the effect. Getting a drawing reference from within the render loop without user interaction is not straightforward and requires alternative approaches, possibly involving undocumented APIs or workarounds that experienced developers like Shachar Raindel might know about.

```cpp
suites.EffectCustomUISuite1()->PF_GetDrawingReference(event_extraP->contextH, &drawing_ref);
```

*Tags: `ui`, `render-loop`, `aegp`, `debugging`*

---

## Can we call PF_PROGRESS and PF_ABORT from another thread started in the render function?

It should be possible, but you may need atomic operations or mutexes to work on all threads at once. However, there are challenges: the progress bar is only rendered at the bottom of the UI, and multiple threads may report progress out of order since they complete asynchronously. A better approach is to bake the computation in the backend (like in userchangedParam), update an arb param during the update, store the result in sequence data, and apply it (similar to warp stabilizer).

*Tags: `threading`, `render-loop`, `arb-data`, `ui`*

---

## How can we report progress while a blocking rendering computation is happening in the main thread?

The standard PF_PROGRESS approach won't work if the main thread is blocked. Instead, consider baking the computation in the backend (like in userchangedParam), updating an arb param during processing, storing the result in sequence data, and applying it in the render function (similar to how warp stabilizer works). This avoids blocking the main thread and allows proper progress reporting.

*Tags: `render-loop`, `params`, `arb-data`, `threading`*

---

## What causes PF_Interrupt_CANCEL to occur in the middle of a render in After Effects?

PF_Interrupt_CANCEL can occur during rendering when certain conditions are met, such as when CAPS LOCK is off and the composition has not been previously rendered to the RAM cache. The issue appears to be related to some interaction with keyboard state or cache status, as the interrupt does not occur when CAPS LOCK is on or when the composition has been fully previewed in the RAM cache beforehand.

*Tags: `render-loop`, `debugging`, `memory`, `caching`*

---

## How can you access the previous frame result during the compute thread in After Effects?

The user asked this question but indicated that despite reading the documentation, a full example would be more helpful. No complete answer was provided in the conversation.

*Tags: `compute-cache`, `smartfx`, `render-loop`, `memory`*

---

## How do you invalidate cached frames to force After Effects to re-render them instead of using cached versions?

The user is asking about the mechanism used by effects like Warp Stabilizer and 3D Camera Tracker to invalidate frames. No answer was provided in this conversation.

*Tags: `caching`, `render-loop`, `smart-render`*

---

## How do you invalidate rendered frames in smart render when external data changes?

Set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag and use extra->cb->GuidMixInPtr in pre-render to scan data that should trigger invalidation. You can add multiple GuidMixInPtr calls for multiple data sources. Store data that affects rendering in sequence data or static variables, not in globalData, since After Effects expects globalData to remain unchanged outside of globalsetup/setdown.

```cpp
if (extra->cb->GuidMixInPtr && !err) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan_to_force_render), reinterpret_cast<void*>(*data_to_scan_to_force_render));
}
```

*Tags: `smart-render`, `sequence-data`, `mfr`, `render-loop`*

---

## Is sequence data flattened or not?

Technically yes, sequence data can be flattened. However, since CC2022, non-flattened sequence data may not be refreshed as expected, as noted in the supervisor example.

*Tags: `sequence-data`, `render-loop`*

---

## Why do frames only invalidate after manually triggering a change like modifying the comp background color?

The issue appears to be that After Effects does not check whether frames need invalidating frequently enough. Even though the boolean value updates properly (confirmed with breakpoints) and frames should invalidate, they only do so after giving AE a 'kick' by changing the composition background color or changing a parameter and changing it back.

*Tags: `render-loop`, `smart-render`, `debugging`*

---

## How can frame N access custom data computed from the previous frame, accounting for previous effects in the chain?

The user explored using checkout Param during compute cache, but noted this approach doesn't account for previous effects. The solution involves retrieving data computed from the previous frame (same size as a frame) through the effect chain.

*Tags: `compute-cache`, `layer-checkout`, `render-loop`, `sequence-data`*

---

## Is there a limitation with AEGP_memsuite when allocating and freeing memory during the render thread?

The user observed that memory allocated using aegp_memHandle during the compute cache thread is not being freed when the freemem function is called, even though instruments show no leaks. This suggests there may be specific limitations or requirements for using AEGP_memsuite during the render thread that differ from other threads.

*Tags: `memory`, `aegp`, `render-loop`, `compute-cache`, `threading`, `debugging`*

---

## How should a plugin handle GPU out-of-memory errors differently for queue renders versus interactive scrubbing?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects may retry with lower thread counts during MFR queue renders, allowing the frame to succeed. However, during interactive scrubbing or non-queue renders, After Effects may not warn the user and simply display a black frame. The plugin should only show an out_data warning message about resolution/bitdepth requirements if the user is not rendering from the queue. Unfortunately, there is no documented way to distinguish between a render request from the queue versus interactive timeline scrubbing in the plugin API.

*Tags: `mfr`, `gpu`, `memory`, `render-loop`, `debugging`*

---

## How should a plugin handle GPU memory exhaustion errors during render versus interactive scrubbing?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects may not warn the user if they're scrubbing the timeline interactively (only black frames appear). During MFR queue renders, AE may retry with lower thread counts. One approach is to use a C++ text library to render an "Out of GPU memory" message directly onto the frame output, making the error visible to the user regardless of render context. However, there is currently no known way to distinguish between queue renders and interactive scrubbing to conditionally show warnings in out_data.

*Tags: `gpu`, `memory`, `mfr`, `render-loop`, `debugging`*

---

## How can I check out the current layer (INPUT) with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT on the INPUT layer at different times, it returns the effectworld before all effects are applied, even those that precede the current effect in the chain. To get the current layer at different times with all effects applied, you need to use a different approach than simple PF_CHECKOUT. The Echo effect demonstrates that this is possible, likely through using layer composition or render callbacks that account for the full effect stack.

*Tags: `layer-checkout`, `render-loop`, `smartfx`, `params`*

---

## Can you use AEGP render suite during the render call, or only during PreRender?

You cannot use AEGP render suite during the render call itself. However, you can use it during PreRender to get rendered frames at a given time with the previous effect reference. Alternatively, you can use smart render with frame preloading in pre-render, using the same method as the current frame.

*Tags: `aegp`, `smart-render`, `render-loop`, `smartfx`*

---

## Is it possible for one effect to render from another effect during smartrender?

Direct effect-to-effect rendering during smartrender is not possible because After Effects will not call smartrender on other layers while a current smartrender is in progress—it waits until the current smartrender is done. Workarounds include: (1) using AEGP outside of render calls but this has poor UI; (2) calling EffectCallGeneric on another plugin if that plugin implements a passthrough code for SmartRender; (3) using AEGP_RenderAndCheckoutLayerFrame_Async which works on any thread but can be buggy; (4) delegating AEGP calls to the UI thread for the next call, though smartrender happens on the render thread and cannot directly call UI thread methods.

*Tags: `smartrender`, `render-loop`, `aegp`, `threading`*

---

## How should a plugin handle processing that requires the previous frame's data (frame n-1 for frame n)?

You have two main approaches: (1) Use sequence data, which is designed for simulation plugins that need to access previous frames, or (2) Check out the input frame n-1 at render time if you only need the pre-effect input frame rather than your plugin's output from frame n-1. If you need your plugin's own output from frame n-1, you'll need to store intermediate results and manage sequential execution, as After Effects has no built-in way to force sequential frame processing.

*Tags: `sequence-data`, `render-loop`, `layer-checkout`, `caching`*

---

## How should cv::Mat pixel data be correctly copied into an After Effects PF_LayerDef structure?

When copying OpenCV Mat data to After Effects PF_LayerDef, you should not manually allocate layerDef->data since After Effects manages PF_LayerDef creation and destruction through PF_NewWorld() and PF_DisposeWorld(). The layerDef->data should already be allocated by After Effects and should not be nullptr. You should use the pre-allocated buffer and copy your OpenCV pixel data directly into it, being careful about channel ordering (OpenCV uses BGRA while After Effects uses ARGB format).

```cpp
// Correct approach: use AE-allocated buffer
PF_Pixel8* pixelData = reinterpret_cast<PF_Pixel8*>(layerDef->data);
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
        PF_Pixel8& aePixel = pixelData[y * width + x];
        aePixel.alpha = pixel[3];  // OpenCV stores alpha at index 3
        aePixel.red = pixel[2];    // Red at index 2
        aePixel.green = pixel[1];  // Green at index 1
        aePixel.blue = pixel[0];   // Blue at index 0
    }
}
```

*Tags: `memory`, `render-loop`, `layer-checkout`*

---

## Can cv::mixChannels be used to convert ARGB back to BGRA to enable memcpy of entire rows?

Yes, cv::mixChannels can be used to convert between channel formats. However, a more efficient approach is to use OpenCV's Mat constructor with the step parameter (rowbytes) to create a Mat that points directly to the layer's pixel data without copying. Use cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes). Additionally, you should benchmark different approaches - mixChannels followed by memcpy might be slower than using IterateSuite (one thread per row) and doing manual channel swizzling when copying from the matrix to the layer.

```cpp
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Tags: `memory`, `gpu`, `render-loop`, `layer-checkout`*

---

## Does smart render provide benefits for effects that require the full image each time they are processed?

Smart render optimizes by only requesting pixels required for specific processing. For effects that require the full image each time, smart render's primary optimization of selective pixel fetching provides limited benefit since the effect must process the entire image anyway. However, smart render can still provide benefits in other scenarios such as avoiding redundant computation when the effect is cached or when only part of the output is needed by downstream effects.

*Tags: `smart-render`, `render-loop`, `output-rect`, `caching`*

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

## What are the practical limits for working with a composition containing thousands of layers in After Effects?

After Effects struggles significantly even at 512 layers. As a user reports, working with over 500 layers becomes a nightmare in terms of performance and usability. The software has inherent limitations that make working with very large numbers of layers (such as 10,000) impractical, even if they are instances of only a small number of source files.

*Tags: `memory`, `performance`, `render-loop`, `caching`*

---

## Why doesn't an Artisan-like OpenGL renderer plugin exist for After Effects?

The question was posed rhetorically but not answered directly in the conversation. The discussion suggests it would be theoretically possible to create a GPU plugin with 10k layers as sprites for real-time rendering, but no explanation was provided for why such a universal renderer hasn't been made.

*Tags: `gpu`, `opengl`, `render-loop`*

---

## How can a plugin maintain a unique ID per instance when duplicating, and why does sequence data become out of sync between the UI thread and render threads?

Sequence data is the appropriate mechanism for storing per-instance unique IDs. When duplicating, catch the duplication event and assign a new unique ID to the sequence data. However, sequence data may become out of sync on render threads even though it updates correctly in UpdateUI and UserChangeParams. Two potential solutions: (1) Check if the sequence is flattened in the project, as this can force render threads to use the correct sequence value, and (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of directly accessing in_data->seq_data, as the latter has not been updated correctly since CC 2021/2022.

*Tags: `sequence-data`, `render-loop`, `threading`, `params`*

---

## Can mask outline modifications be done as a regular effect that procedurally modifies path vertices each frame rather than as a one-time destructive operation?

You cannot modify any project state in the render thread (like adding layers, changing layer properties, or masks). However, procedurally changing mask vertices might be possible using expressions. Baking the paths is also an effective alternative approach, though not as magical.

*Tags: `aegp`, `scripting`, `render-loop`, `params`*

---

## Can mask outlines be modified using AEGP_MaskOutlineSuite?

Yes, AEGP_MaskOutlineSuite can be used to modify mask outlines, but this approach would be destructive and one-time only, similar to running a script that modifies path vertices. For a procedural effect that modifies path vertices each frame before rasterization, a regular effect applied to a layer would be more appropriate.

*Tags: `aegp`, `params`, `render-loop`*

---

## What were the root causes of bugs in FastBoxBlur?

Two critical bugs were identified: (1) Incorrect bit depth detection causing memory corruption, and (2) Improper stride handling for padded buffers. The solution involved fixing bit depth detection to use rowbytes calculation and implementing proper stride handling throughout the blur algorithm.

*Tags: `memory`, `debugging`, `render-loop`*

---

## Is there a method for forcing masks on a layer to be applied after the plugin processes?

No answer was provided in the conversation.

*Tags: `smartfx`, `render-loop`, `layer-checkout`*

---

## Is there a method for forcing masks on the layer to be applied after the plugin processes?

There is no built-in method to force masks to be applied after plugin processing. As a workaround, you can have the user select the mask type to None and compute the alpha yourself within your plugin, though this requires additional implementation effort.

*Tags: `params`, `ui`, `render-loop`*

---

## How do you create a render-only version of a plugin that doesn't require additional licenses for render farms?

One approach is to make all parameters invisible while still allowing the plugin to read them, creating a read-only version of the plugin that can process renders without displaying the GUI interface.

*Tags: `deployment`, `ui`, `render-loop`, `pipl`*

---

## How can you check if the After Effects renderer is running in headless mode?

Use the AppSuite4 API with the PF_IsRenderEngine function. Create a PF_Boolean variable and call suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine) to set it to true if running in render engine mode.

```cpp
PF_Boolean bIsRenderEngine = true;
suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine);
```

*Tags: `aegp`, `render-loop`, `debugging`, `deployment`*

---

## Why not create a custom GPU context in OpenGL or Vulkan instead of using After Effects' native GPU API?

Instead of relying on After Effects' limited native GPU rendering API, you can create your own GPU context using OpenGL, Vulkan, or WebGPU. This approach allows you to maintain your CPU logic while selectively sending only what's needed to the GPU, avoiding the limitations of AE's GPU world which is less developed and has fewer available suites. gabgren and Tim Constantinov successfully used this approach, eventually settling on WebGPU after trying native AE and OpenGL.

*Tags: `gpu`, `opengl`, `vulkan`, `render-loop`, `cross-platform`*

---

## Why does GPU mode still run slow when the work is being done on GPU?

The performance issue occurs when critical operations fall back to CPU processing. For example, in UV unwrapping for 3D models, if a mask is present in GPU mode, the data passed is post-mask which breaks the mathematical requirements, forcing the hard work back to the CPU path, negating GPU performance benefits.

*Tags: `gpu`, `memory`, `performance`, `render-loop`*

---

## How can a plugin render output that extends beyond the bounds of its adjustment layer?

You need to modify the output_rect to be larger than the layer's bounds. By setting the output rectangle in your render function to extend beyond the adjustment layer's dimensions, you can render content (like text) that appears outside the layer's original boundaries while still being contained within the composition.

*Tags: `output-rect`, `render-loop`, `adjustment-layer`*

---

## How can you set text on a text layer from an effect on a frame-by-frame basis?

You need to communicate with an AEGP plugin that can set the text of a layer for each frame, since invoking scripts via the render loop does not work for this purpose.

*Tags: `aegp`, `scripting`, `render-loop`, `text`*

---

## How can a C++ plugin update a text layer every frame after rendering completes?

This is not cleanly possible due to AE's project locking during render operations. Possible workarounds include: (1) writing data to a compute cache or sequence data and reading it from the UI thread via the aegp_idle hook, (2) using AEGP_SetText from the UI thread instead of the render thread, (3) rendering text yourself in the render thread using an external library, or (4) saving to disk and having a ScriptUI panel with a timer read the data. The aegp_idle hook approach may not work without a GUI or in MediaEncoder. Setting text from the render thread causes a locked project error.

*Tags: `aegp`, `render-loop`, `threading`, `ui`, `sequence-data`, `compute-cache`*

---

## What is the best way to pass data from an AEGP plugin to an effects plugin?

Use an AEGP plugin with an idle hook to process a queue. Every frame, push your string result into a queue, and the idle hook processes the queue occasionally. The AEGP plugin and FX plugin are separate binaries, so you'll need a C interface to push strings into the queue.

*Tags: `aegp`, `threading`, `render-loop`, `memory`*

---

## What are the risks of encoding data into dead pixels and retrieving them with sampleImage?

The solution is dependent on render format which can be tricky. CPU and GPU have different byte orders in After Effects (ARGB vs RGBA), and GPU formats like CUDA may use BGRA. Any swizzeling or color format conversion could mangle the data. Additionally, after threading changes in AE, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering yet.

*Tags: `gpu`, `memory`, `render-loop`, `threading`, `cuda`*

---

## What is a method to set source text dynamically in After Effects using rendered data?

Encode and render dead pixels in your render output, then use a sampleImage expression to retrieve those pixels which can then set the source text. This method mostly works with a few caveats and also works with Null objects.

*Tags: `scripting`, `ui`, `render-loop`*

---

## Is it possible to achieve progressive rendering in an After Effects plugin that continuously shows results to the user as more samples are computed?

The conversation acknowledges the desire for progressive rendering where samples accumulate and display over time rather than waiting for all samples to complete before showing results. While the technical feasibility is not explicitly detailed in the response, the user notes they have this working outside of After Effects and finds it more interactive. The challenge with current AE plugin behavior is that it waits for complete render before display, making it feel slow compared to progressive sampling approaches.

*Tags: `smart-render`, `render-loop`, `ui`, `preview`*

---

## Did you test the GI rendering performance in quarter resolution?

The conversation suggests testing in quarter resolution as a next step to evaluate performance improvements, though the specific test results are not explicitly stated.

*Tags: `gpu`, `render-loop`, `performance`*

---

## How do you handle different color spaces when developing plugins for both After Effects and Premiere?

In After Effects, plugins typically work in ARGB colorspace. Premiere uses BGRA_8u with a special suite for 8-bit processing. For 32-bit float, you need to write your own conversion function. You can use the Iterate8Suite1 (or SuiteV2 since 2022) for ARGB, but Premiere requires a special BGRA suite. You don't have to support all color spaces—you can optimize for BGRA only and skip YUVA if needed. Basic rendering in 8/16 bit works without optimization, though you won't get the 32-bit icon next to the plugin. Check the SDK noise example for reference implementations of BGRA and YUVA colorspace handling.

*Tags: `premiere`, `color-space`, `bgra`, `argb`, `render-loop`, `optimization`*

---

## Does PF_ABORT() work in Premiere Pro the same way it does in After Effects?

No, PF_ABORT(in_data) does not abort rendering in Premiere Pro like it does in After Effects. Even when frames take over half a second to render and PF_ABORT() is called, Premiere does not set err to PF_Interrupt_CANCEL as expected. In Premiere, you appear to be required to completely finish rendering every frame that is requested, without the ability to abort mid-render like in After Effects.

*Tags: `premiere`, `render-loop`, `debugging`*

---

## Why does Premiere pass NULL for input_world in Render calls when compiled with fast optimizations, while After Effects doesn't have this issue?

When a plugin is compiled with fast optimizations, Premiere may pass input_world = NULL in the Render call, whereas the same code without optimizations receives the data correctly. This issue does not occur in After Effects. The root cause appears to be related to compiler optimization levels affecting how Premiere handles input data passing to plugins.

*Tags: `premiere`, `debugging`, `render-loop`, `smartfx`*

---

## How can you prevent the compiler from optimizing a render function in an After Effects plugin?

Add the `__attribute__ ((optnone))` compiler attribute before the render function declaration. This tells the compiler to skip optimization for that specific function, even if the rest of the plugin is compiled with optimizations enabled. This workaround was found to resolve rendering issues in some cases.

```cpp
__attribute__ ((optnone))
void renderFunction() {
  // render code
}
```

*Tags: `render-loop`, `build`, `debugging`, `macos`, `windows`*

---

## How should you handle null inoutworld parameters in Premiere Pro plugins?

If you set a condition to check if inoutworld is null and return an error, Premiere Pro will redo the render. This is a valid strategy for handling null world pointers in render functions.

```cpp
if (inoutworld == null) return err;
```

*Tags: `premiere`, `render-loop`, `debugging`, `mfr`*

---

## What causes a plugin to not work properly in Premiere when it works in After Effects?

The issue can stem from the no_params_vary flag. In After Effects, this can be replaced using MIX_GUI during the pre_render thread. However, this flag has no effect in Premiere, so alternative approaches are needed for cross-application compatibility.

*Tags: `premiere`, `params`, `render-loop`, `cross-platform`, `aegp`*

---

## How do you cache frames in an After Effects effect plugin and use the cached result as input for the next frame?

Nate asked about implementing frame caching in an effect plugin where frame 1 is processed and cached, then that cached result becomes the input for frame 2's processing, creating a cumulative effect. This is a common pattern for effects that depend on previous frame state.

*Tags: `caching`, `render-loop`, `mfr`, `memory`*

---

## How do you store and access the previous frame's EffectWorld in sequence data within the Render function?

You can store an EffectWorld in your sequence_data struct and access it during render(). Initialize the struct with fields for the cached result world, a boolean flag indicating if it's loaded, the last render time, and a hash string. In SequenceSetup, initialize these fields. In Render, check if the hash or frame timing has changed to determine if you should invalidate the cache. If the cache is valid and needed, use the stored resultW from sequence data. Store the new render result back into sequence data for the next frame using PF_NEW_WORLD and PF_COPY. Note: MFR may affect this workflow, and if you need to access the previous source frame (rather than the previous result), you'll need to use a suite instead.

```cpp
typedef struct {
    PF_EffectWorld  resultW;
    bool            result_loaded = false;
    A_long          result_last_time;
    char            result_hash[260];
} sequenceData;

static PF_Err Render(...) {
    sequenceData *seqdataP = *(sequenceData **)out_data->sequence_data;
    
    A_char result_hash[260];
    sprintf(result_hash, "%d%d%d%d%d%i",
        in_data->time_step,
        in_data->time_scale,
        in_data->quality,
        in_data->downsample_x,
        in_data->downsample_y,
        lod);
    
    // Reset cache if hash differs or frame is not sequential
    if (strcmp(result_hash, seqdataP->result_hash) != 0 || 
        current_time != (seqdataP->result_last_time + in_data->time_step)) {
        seqdataP->result_loaded = false;
    }
    
    seqdataP->result_last_time = current_time;
    strcpy(seqdataP->result_hash, result_hash);
    
    bool use_last_result = seqdataP->result_loaded && params[PARAM_ADDITIVE]->u.bd.value;
    
    if (use_last_result) {
        // Use seqdataP->resultW from previous frame
    }
    
    PF_EffectWorld cachingW;
    AEFX_CLR_STRUCT(cachingW);
    ERR(PF_NEW_WORLD(tensorW.width, tensorW.height, PF_NewWorldFlag_NONE, &cachingW));
    PF_COPY(&tensorW, &cachingW, NULL, NULL);
    
    if (!seqdataP->result_loaded) {
        AEFX_CLR_STRUCT(seqdataP->resultW);
        ERR(PF_NEW_WORLD(tensorW.width, tensorW.height, PF_NewWorldFlag_NONE, &seqdataP->resultW));
    }
    
    PF_COPY(&cachingW, &seqdataP->resultW, NULL, NULL);
    seqdataP->result_loaded = true;
    ERR(PF_DISPOSE_WORLD(&cachingW));
}
```

*Tags: `sequence-data`, `caching`, `render-loop`, `memory`*

---

## Why does a displacement map effect sometimes use the original frame instead of the previously cached frame?

Nate is experiencing inconsistent behavior with a displacement mapping effect that should feed the distorted output back as input for the next frame. The effect sometimes works correctly (distorting and caching each frame to use as the next input) but randomly reverts to using the original frame instead of the cached frame. This suggests a potential issue with the frame caching or smart render logic that determines which input frame gets used for displacement. The problem appears to be intermittent rather than systematic.

*Tags: `caching`, `smart-render`, `render-loop`, `debugging`*

---

## Why is a plugin exhibiting glitch behavior with Multi-Frame Rendering (MFR) even though the Multithreaded flag is not set?

MFR glitches can occur when different threads/CPUs are using different previous frames, causing inconsistent rendering behavior. Even without the Multithreaded flag explicitly set, the plugin may still attempt MFR, which can lead to these issues. Disabling MFR resolves the problem. The root cause appears to be improper frame state management across threads during multi-frame rendering.

*Tags: `mfr`, `threading`, `render-loop`, `debugging`*

---

## What is the behavior of After Effects regarding multi-frame rendering (MFR) and thread safety in plugins?

Even if plugins don't specify they are MFR safe, After Effects still calls them from different threads and requests frames out of sequence (for example, frame 5, then frame 3, then frame 6). The only way to prevent this non-sequential, multi-threaded behavior is to disable MFR completely.

*Tags: `mfr`, `threading`, `render-loop`*

---

## How do you expand the output buffer with smartFX to prevent outlines from being clipped by layer bounds?

When using smartFX in After Effects, if you're generating outlines around an alpha image and they're getting cut off by the layer bounding box, you need to expand the output buffer. This is typically done by modifying the output rectangle in your smartFX implementation to be larger than the layer's current bounds, ensuring that generated content (like outlines) isn't clipped.

*Tags: `smartfx`, `output-rect`, `mfr`, `render-loop`*

---

## How do you expand the result rectangle in After Effects plugin rendering?

To expand the in_result.result_rect and in_result.max_result_rect, modify them before calling UnionLRect. The typical pattern is to expand these rectangles first, then use UnionLRect to union in_result.max_result_rect with in_result.result_rect to make the result rect the same as max result rect, and finally union with the extra->output->result_rect.

```cpp
UnionLRect(&in_result.max_result_rect, &in_result.result_rect); // make result rect the same as max result rect
UnionLRect(&in_result.max_result_rect, &extra->output->result_rect);
```

*Tags: `output-rect`, `render-loop`, `aegp`*

---

## What needs to be updated when using PF_COPY or buffer operations in a plugin?

When performing buffer operations like PF_COPY, the source and destination rectangles (src and dst rects) must be updated to reflect the actual regions being operated on.

*Tags: `smartfx`, `output-rect`, `render-loop`*

---

## How do you correctly extract and transform camera and light matrices from After Effects to use in a Vulkan shader?

The user is working with After Effects camera/light matrix data for a Vulkan-based plugin. Their approach involves: (1) getting the camera matrix from AE, (2) converting it to column-major format, (3) normalizing position in clip space (-1,1) using zoom as a z factor, and (4) getting and normalizing the light matrix. They note they are not using OpenGL operations like glPerspective or glMatrixMode since they are targeting Vulkan. The user indicates they've tried model and projection matrix operations but results don't match AE's render. This suggests the camera transformation pipeline in AE may require specific handling of coordinate systems, matrix conventions (row vs column major), and potentially AE-specific camera parameters that differ from standard graphics API conventions.

*Tags: `camera`, `vulkan`, `matrix`, `shader`, `render-loop`, `gpu`*

---

## How can you cache frame data that's too large to fit in RAM?

You can save binary data directly to disk for each frame (e.g., 0000.dat, 0001.dat, etc.), similar to how Houdini handles large computation results. After Effects may have a cache API for this, but if not, you'll need to implement your own disk-based caching system to store frame data when it exceeds available memory.

*Tags: `caching`, `memory`, `render-loop`, `mfr`*

---

## What flags or methods are used in high-end plugins like Particular to cache frames and delay render updates until caching is complete?

The user is asking about caching mechanisms used in professional plugins like Particular that intelligently manage frame rendering by waiting for frame cache completion before updating the CTI (Current Time Indicator). This typically involves smart render and compute cache APIs provided by After Effects to optimize performance when scrubbing the timeline.

*Tags: `caching`, `smart-render`, `render-loop`, `compute-cache`*

---

## How does caching behavior work by default in the render function, and how can you update progress indication?

Caching calculations taking place in the render function will have default caching behavior where the output won't be updated until the plugin has populated it. For updating the loading bar specifically, use PF_Progress.

*Tags: `caching`, `render-loop`, `mfr`*

---

## How can you implement an iterative render loop where a frame needs to be re-rendered multiple times until a calculation is complete?

According to gabgren, you can call render() multiple times within a single frame processing cycle. This requires a flag to tell After Effects to re-render the frame, and inside the render function, an if clause should check whether the render is actually done or not. This pattern was used in threads discussing multi-pass rendering approaches.

*Tags: `render-loop`, `smart-render`, `aegp`*

---

## How can you store and cache multiple PF_EffectWorlds in a sequence data structure for frame-by-frame processing?

You can store arrays of PF_EffectWorld structures in your sequence data struct (e.g., PF_EffectWorld resultW[10000]) along with tracking variables like result_loaded, isCaching, result_last_time, and result_hash. In your main render function, implement a loop that caches missing previous frames when a later frame is requested by checking the last calculated frame and iterating through frames using IterateFloatSuite1()->iterate() to render each intermediate frame into the cached array before processing the currently requested frame.

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

*Tags: `caching`, `sequence-data`, `memory`, `render-loop`*

---

## What should be done after checking out a parameter value in the render loop?

After getting a parameter value through checkout in a loop, you must call checkin immediately after using the value. Holding the checkout open or delaying checkin can cause errors like 'bad parameter passed to effect callback'. Always pair checkout with checkin as soon as the parameter data is no longer needed.

*Tags: `params`, `layer-checkout`, `render-loop`, `debugging`*

---

## Should layer checkout and check-in be done immediately after use in a render loop, or can check-in be deferred?

Layer checkout and check-in should be handled carefully in render loops. For each frame checked out, you should check-in immediately after use, even if an error occurs. The recommended pattern is: for each frame, checkout the frame, perform work if no error occurs, then check-in the frame (using Err2 or equivalent error handling). Alternatively, you can defer check-in until the end of the render thread, but every checked-out layer must be checked in whether it was used or not. Error handling during checkout must also be monitored.

```cpp
For X
    Checkout frame x
    If (no err) do job
    Err2(check-in frame x)
End of loop
```

*Tags: `layer-checkout`, `render-loop`, `memory`, `aegp`, `debugging`*

---

## Why does Multi-Frame Rendering (MFR) start with multiple threads but reduce to a single thread on longer compositions?

James Whiffin observed that MFR often initiates rendering with 6-10 threads but consistently drops to single-threaded operation when compositions are sufficiently long. This behavior persists even when custom effects are disabled and only first-party Adobe effects are present, suggesting it is a characteristic of MFR itself rather than third-party plugin behavior.

*Tags: `mfr`, `threading`, `render-loop`, `performance`*

---

## What AEGP method should be used to render and checkout a layer frame?

AEGP_RenderAndCheckoutLayerFrame is the AEGP method specifically implemented for rendering and checking out layer frames.

*Tags: `aegp`, `layer-checkout`, `render-loop`*

---

## How do you reproduce warp stabilizer banner messages in the viewport—using the Canvas suite or drawing directly on the rendered frame?

The user is asking about the proper approach to display banner messages similar to the warp stabilizer effect. This involves choosing between using the Canvas suite for viewport drawing versus drawing directly on the rendered frame output.

*Tags: `ui`, `render-loop`, `aegp`, `output-rect`*

---

## How can you obtain a drawing reference in After Effects plugin render loops without requiring user interaction?

The standard approach using EffectCustomUISuite1()->PF_GetDrawingReference() only provides a drawing context during PF_Cmd_EVENT callbacks triggered by user interaction. To draw during render without user interaction (similar to VFX Stabilizer behavior), you need an alternative method to access the drawing reference that doesn't depend on the extraPV parameter passed through PF_EventExtra. This requires investigating lower-level drawing APIs or background rendering approaches that don't rely on event-driven context delivery.

*Tags: `ui`, `render-loop`, `aegp`, `debugging`*

---

## How can you draw to the composition window during rendering without relying on user interaction events?

The challenge is that PF_GetDrawingReference() is only available through PF_EventExtra during PF_Cmd_EVENT, which requires user interaction with the effect parameters or composition preview. To draw during a render loop without user interaction (similar to VFX Stabilizer behavior), you need an alternative approach to obtain the drawing reference that isn't tied to user interaction events. This is a difficult problem with limited documented solutions; consulting experienced plugin developers like Shachar Gran is recommended.

*Tags: `ui`, `render-loop`, `aegp`, `debugging`*

---

## Can you call PF_PROGRESS and PF_ABORT from a separate thread spawned during the render function?

No, you cannot call PF_PROGRESS and PF_ABORT from another thread. These functions must be called from the main thread. If you need to report progress during a blocking computation, you should consider alternative approaches such as baking the compute in a backend operation (like in userchangedParam), updating an arbitrary parameter during the update phase, storing results in sequence data, and applying them later (similar to how warp stabilizer works). This avoids blocking the main thread and allows proper progress reporting through the UI.

*Tags: `threading`, `render-loop`, `params`, `arb-data`*

---

## Why does a plugin render get interrupted with PF_Interrupt_CANCEL during queue rendering?

PF_Interrupt_CANCEL during render queue operations can occur under specific conditions. One user reported this happening after 10-15 frames when CAPS LOCK was off and the composition had not been previously previewed to RAM cache. The interrupt appears as 'User abort' but occurs without user action. The behavior did not occur when CAPS LOCK was on or when the composition was fully previewed in the RAM cache before rendering. This suggests the interrupt may be related to input handling or cache state during rendering.

*Tags: `render-loop`, `debugging`, `smartfx`*

---

## How do you invalidate cached frames in After Effects so they will be re-rendered instead of using the cache?

This question was asked about invalidating rendered frames to force After Effects to re-render them, similar to how built-in effects like Warp Stabilizer and 3D Camera Tracker work. However, no answer was provided in the conversation.

*Tags: `caching`, `render-loop`, `aegp`, `smart-render`*

---

## How do you invalidate rendered frames when data changes in a plugin using smart render?

Set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag and use GuidMixInPtr in pre-render to scan data that should force re-rendering. In pre-render, call: if (extra->cb->GuidMixInPtr && !err) { extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan), reinterpret_cast<void*>(*data_to_scan)); }. You can add multiple GuidMixInPtr calls for different data sources. This works particularly well when data is stored in sequence data structures.

```cpp
if (extra->cb->GuidMixInPtr && !err) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan_to_force_render), reinterpret_cast<void*>(*data_to_scan_to_force_render));
}
```

*Tags: `smart-render`, `params`, `sequence-data`, `render-loop`*

---

## How do you force After Effects to re-check whether frames need invalidating when using sequence data?

When sequence data isn't flattened, After Effects may not refresh frames as expected even when the data updates properly (verified via breakpoints). A workaround is to give AE a "kick" by changing composition background color or toggling a parameter, which forces it to check for invalidation. Alternatively, see the Adobe Community discussion on forcing rendering on events in separate threads: https://community.adobe.com/t5/after-effects-discussions/force-rendering-on-an-event-in-a-separate-thread-coming-back-to-the-main-ui-thread-on-an-event/m-p/13628197

*Tags: `sequence-data`, `render-loop`, `debugging`, `threading`*

---

## How can you retrieve custom data computed from a previous frame in After Effects plugins?

You can use checkout Param during compute cache to access data from previous frames. However, this approach has limitations as it doesn't account for previous effects in the chain. For frame-sized custom data, you need to handle the data flow through the compute cache mechanism while being aware that parameter checkout alone may not fully solve the problem if previous effects need to be considered.

*Tags: `compute-cache`, `params`, `render-loop`, `arb-data`*

---

## How do you access computed data from previous frames in a compute cache when using smart render?

When accessing computed data from frame N-1 in a compute cache, if that frame hasn't been computed yet, you need access to the layer at the previous frame. This creates a cascading dependency—if N-2 is also not calculated, you need the layer at N-2 as well. With smart render, pre-loading all needed layers during pre-render can be expensive, especially if the user sets the timeline cursor at an arbitrary time like 10 seconds. The solution is to use classical checkout_param during compute cache, though this loads layers without their effects applied.

*Tags: `compute-cache`, `smart-render`, `layer-checkout`, `render-loop`*

---

## Is there a memory leak when using aegp_memHandle during the compute cache thread that persists even after calling the free memory function?

A developer reported that instruments showed no leaks, but memory allocated using aegp_memHandle during the compute cache thread was not being freed in reality when the freemem function was called. This issue occurred specifically during render operations, suggesting there may be a limitation with AEGP_memsuite during the render thread that prevents proper memory deallocation.

*Tags: `memory`, `aegp`, `compute-cache`, `render-loop`, `debugging`*

---

## What is the best practice for handling GPU out-of-memory errors in After Effects plugins?

When a plugin runs out of GPU memory, it should return PF_Err_OUT_OF_MEMORY. During MFR (multi-frame rendering), After Effects may retry the frame with a lower thread count. For interactive scrubbing where the user isn't rendering from the queue, one approach is to use a C++ text library to render an error message directly onto the frame (e.g., "Out of GPU memory") so the user understands why a black frame appeared. However, there is currently no reliable way to distinguish between a render queue request and interactive timeline scrubbing to conditionally show warnings in out_data.

*Tags: `gpu`, `memory`, `mfr`, `render-loop`, `output-rect`*

---

## How can you display a warning alert in an After Effects plugin without blocking the render thread?

Create a static global boolean variable that is modified by a mutex in the render thread, then check and print the warning if the boolean is true in another thread such as pFupdate_param_UI. This approach prevents blocking the render thread while still allowing warnings to be displayed to the user.

```cpp
static bool g_warning_flag = false;
static std::mutex g_warning_mutex;

// In render thread:
{
  std::lock_guard<std::mutex> lock(g_warning_mutex);
  g_warning_flag = true;
}

// In pFupdate_param_UI thread:
if (g_warning_flag) {
  // Display warning alert
  std::lock_guard<std::mutex> lock(g_warning_mutex);
  g_warning_flag = false;
}
```

*Tags: `threading`, `render-loop`, `ui`, `debugging`*

---

## How can I check out the current layer with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT to check out the INPUT layer at different times, it returns the effectworld before all effects are applied, including effects applied before the current effect. However, this is possible to achieve with all effects applied, as demonstrated by the Echo effect. The solution involves understanding the layer checkout mechanism and potentially using different checkout strategies or timing parameters to access the fully composited layer rather than the raw input.

*Tags: `layer-checkout`, `aegp`, `mfr`, `render-loop`*

---

## Why does checking out a layer parameter in PreRender still return the layer before all effects are applied?

When using PF_CHECKOUT_PARAM to check out a layer (INPUT) in the PreRender callback, the layer is returned in its pre-effect state rather than with effects applied. This is expected behavior in the AE plugin architecture—the checkout at PreRender stage occurs before the effect chain is fully processed.

*Tags: `aegp`, `layer-checkout`, `render-loop`, `debugging`*

---

## How do you handle dependencies on previous frames in smart render when you need data from multiple prior frames?

In smart render, if your compute needs the result of a previous frame (e.g., frame N-1) but also requires frame N-2, you cannot use the smart checkout layer. Instead, you must use the classical checkout/checkin parameter approach to access the frames you need.

*Tags: `smart-render`, `layer-checkout`, `render-loop`, `sequence-data`*

---

## How can I use RenderDoc to debug GPU code in After Effects plugins without crashes?

Use RenderDoc's in-application API with process injection. Enable process injection in Tools > Settings > "Enable process injection" (hidden by default). Launch AE fully, then inject RenderDoc via File > Inject into Process before applying your plugin. In your plugin's PF_Cmd_GLOBAL_SETUP, load the RenderDoc API and use the rdoc_api pointer to manually begin and end frame captures in your render function. This approach avoids the crashes that occur when launching RenderDoc normally with AE.

```cpp
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll"))
{
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
```

*Tags: `gpu`, `debugging`, `windows`, `render-loop`*

---

## How do you use GuidMixInPtr to force re-rendering in After Effects plugins?

Use the GuidMixInPtr callback to mix in a GUID dependency that forces re-renders. First, set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag. Then in PreRender, check that in_data->version.major >= PF_AE130_PLUG_IN_VERSION and verify extraP, extraP->cb, and extraP->cb->GuidMixInPtr are non-null and not the sentinel value 0xabababababababab. Call GuidMixInPtr with a value that changes (e.g., based on time or other conditions) to trigger re-renders. For timestamps, use gettimeofday on non-Windows platforms or GetTickCount64 on Windows, then multiply by 100 and add randomness to ensure uniqueness.

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data, PF_PreRenderExtra* extraP) {
    if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION && extraP && extraP->cb && extraP->cb->GuidMixInPtr && (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab) {
        int r = time(0) + in_data->current_time;
        extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(r), reinterpret_cast<void *>(&r));
    }
}

#ifndef WIN32
struct timeval tp;
gettimeofday(&tp, NULL);
long long dummy = (long long) tp.tv_sec * 1000L + tp.tv_usec / 1000;
#else
long long dummy = GetTickCount64();
#endif
dummy = dummy * 100 + rand() % 100;
extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(dummy), reinterpret_cast<void *>(&dummy));
```

*Tags: `guid-dependencies`, `render-loop`, `prerender`, `caching`, `cross-platform`*

---

## Can one effect render from another effect during smartrender in After Effects?

During smartrender, it is not straightforward to have one effect call another effect's render. Smartrender happens on the render thread, and AE will not call smartrender on other layers if a current smartrender is already in progress—it waits until the current smartrender completes. One possible approach mentioned is to use EffectCallGeneric with a custom passthrough code if the other plugin implements it, but this has significant limitations. AEGP calls cannot be made from the render thread directly; they must be delegated to the UI thread. The async render and checkout functions (AEGP_RenderAndCheckoutLayerFrame_Async) might work from any thread but are noted as buggy. Ultimately, the questioner settled on manually implementing effects internally rather than calling other plugins dynamically.

*Tags: `smart-render`, `aegp`, `threading`, `render-loop`*

---

## How can you access the previous frame's data when processing frame n in an After Effects effect plugin?

You can check out frame n-1 at render time to access the input frame from the previous frame. For more complex cases where you need the output from your plugin's previous frame, you should use sequence data, which is specifically designed for simulation plugins that require frame-by-frame dependencies.

*Tags: `sequence-data`, `render-loop`, `layer-checkout`, `aegp`*

---

## Is it possible to force After Effects to render frames sequentially in a plugin?

No, there is no way to force After Effects to give you frames sequentially. The plugin must be designed to handle out-of-order frame rendering or implement its own logic to manage sequential processing if required.

*Tags: `render-loop`, `threading`, `aegp`*

---

## How should you properly copy pixel data from an OpenCV cv::Mat to an After Effects PF_LayerDef structure?

When copying from cv::Mat to PF_LayerDef, you should not manually allocate layerDef->data yourself—AE manages PF_LayerDef creation and destruction through PF_NewWorld()/PF_DisposeWorld(). The layerDef->data pointer should already be allocated by AE. The pixel data copy itself needs to respect both memory layout and channel ordering: OpenCV uses BGRA order (blue at index 0, green at 1, red at 2, alpha at 3) while AE's PF_Pixel8 struct has ARGB fields. Additionally, ensure you're using the correct row byte calculation and pixel indexing that accounts for the actual rowbytes value, not just width * sizeof(PF_Pixel8), as AE may add padding.

```cpp
PF_Pixel8* pixelData = reinterpret_cast<PF_Pixel8*>(layerDef->data);
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
        PF_Pixel8& aePixel = pixelData[y * (layerDef->rowbytes / sizeof(PF_Pixel8)) + x];
        aePixel.alpha = pixel[3];
        aePixel.red = pixel[2];
        aePixel.green = pixel[1];
        aePixel.blue = pixel[0];
    }
}
```

*Tags: `memory`, `params`, `render-loop`, `aegp`*

---

## How can you efficiently copy pixel data from an OpenCV Mat to an After Effects layer?

Use the Iterate Suite and copy entire rows at a time with memcpy() instead of copying pixel-by-pixel. Respect the layer's rowBytes value when advancing pointers. Ensure both buffers have matching pixel format (ARGB for AE, BGRA for OpenCV by default) and are uchar type for proper memory alignment. Advance both the OpenCV buffer and layer buffer by their respective rowByte values during the copy operation.

```cpp
static PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
    return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        cv::Vec4b* matRow = mat.ptr<cv::Vec4b>(y);
        PF_Pixel* aeRow = (PF_Pixel*)((char*)layer->data + (y * layer->rowbytes));
        memcpy(aeRow, matRow, layer->width * sizeof(PF_Pixel));
    }
}
```

*Tags: `memory`, `caching`, `output-rect`, `render-loop`*

---

## Should you use cv::mixChannels with memcpy or the IterateSuite for channel swizzling performance?

You should benchmark both approaches. Using cv::mixChannels followed by memcpy may be slower than using After Effects' IterateSuite (which provides one thread per row) and performing channel swizzling manually during the copy operation from matrix to layer. The IterateSuite approach can provide better performance by combining the channel conversion and copy in a single operation per row.

*Tags: `threading`, `render-loop`, `optimization`, `aegp`*

---

## Does smart render provide benefits for effects that require the full image for processing?

Smart render helps by only requesting pixels required for specific processing. For effects that require the full image each time they are processed (when visible), smart render may not provide additional optimization benefits since the entire image must be fetched anyway. The advantage of smart render is primarily for effects that can work on partial regions of interest.

*Tags: `smart-render`, `render-loop`, `output-rect`, `optimization`*

---

## Does smart render provide benefits for effects that require the full image each time they process?

Smart render helps by only getting pixels required for the specific processing needed. For an effect that requires the full image each time it is processed (when visible), smart render may not provide additional benefits since the entire image must be fetched anyway.

*Tags: `smart-render`, `smartfx`, `render-loop`*

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

## How do you check out a frame of the current layer after prior effects have been applied in After Effects plugins?

To check out a frame after prior effects are applied (rather than before), you need to use a different approach than PF_CHECKOUT_PARAM, which gets the frame before all prior effects. The conversation indicates this is possible but the specific method was not detailed in the exchange. The questioner was trying to retrieve a frame after the Exposure effect but before the AI Color Match effect in the effect stack.

*Tags: `aegp`, `layer-checkout`, `params`, `render-loop`*

---

## What are the limitations of working with thousands of layers in After Effects, and how do plugins like Newton handle this?

After Effects struggles with performance even at 512 layers. The user is asking whether Newton (a physics simulation plugin) encounters the same scaling issues when simulating many layers, and considering whether switching to GPU-based sprite rendering would be necessary to maintain flexibility while handling 10,000 layer instances.

*Tags: `memory`, `render-loop`, `gpu`, `performance`, `ae-limits`*

---

## Why hasn't someone created an OpenGL-based realtime GPU renderer plugin for After Effects that can handle thousands of layers as sprites?

This is a theoretical discussion about a potential feature gap in After Effects plugin ecosystem. The concept would involve creating a custom 3D renderer plugin that could leverage GPU acceleration to render 10,000+ layers in realtime by treating them as sprites, similar to what exists in other software like Artisan. No concrete answer or existing solution was provided in the conversation.

*Tags: `gpu`, `opengl`, `render-loop`, `reference`*

---

## How can you leverage After Effects' rendering engine (3D lights, effects) while maintaining control over individual layers in a plugin?

One approach is to use the GPU route with sprites while keeping flexibility for individual layers. This allows you to take advantage of AE's built-in rendering engine capabilities like 3D lights and effects on each layer, rather than trying to replicate these features manually in your own 3D plugin.

*Tags: `gpu`, `3d`, `render-loop`, `layer-checkout`, `effects`*

---

## How do you ensure an After Effects effect uses unaltered input pixels rather than output from a previous render when parameters change?

After Effects automatically provides the correct input layer on each render call. In PF_Cmd_SMART_RENDER, use extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world) to get the right input. In PF_Cmd_RENDER, use params[0]->u.ld to access the correct input layer. You don't need to do anything special—AE handles providing the unaltered input, not the output from your plugin's previous render.

```cpp
// PF_Cmd_SMART_RENDER
extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world);

// PF_Cmd_RENDER
params[0]->u.ld  // correct input layer
```

*Tags: `smart-render`, `layer-checkout`, `render-loop`, `aegp`*

---

## How can you ensure sequence data stays synchronized across render threads when duplicating a plugin instance?

When duplicating a plugin and assigning a new unique ID to sequence data, the sequence data may become out of sync on render threads even though UpdateUI and UserChangeParams receive the correct values. Two potential solutions: (1) Check if the sequence is flattened in the project, as flattening can force render threads to get the correct sequence value. (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of relying on in_data->seq_data directly, as the latter is not updated correctly since CC2021/2022.

*Tags: `sequence-data`, `threading`, `render-loop`, `params`, `debugging`*

---

## Can you procedurally modify mask vertices each frame in an After Effects plugin effect?

You cannot modify project state (including masks) in the render thread of a plugin. However, procedurally changing mask vertices might be possible using expressions. Alternatively, 'baking' the paths is an effective workaround, though not as dynamic. If the functionality is available in the JavaScript API, you could potentially run scripts from within your plugin to achieve this.

*Tags: `aegp`, `render-loop`, `params`, `scripting`*

---

## Can AEGP_MaskOutlineSuite be used to procedurally modify mask path vertices each frame as a regular effect?

AEGP_MaskOutlineSuite can modify mask outlines, but the distinction is between destructive/one-time modifications (like running a script that permanently changes path vertices) versus procedural, non-destructive modifications applied each frame before rasterization. A regular effect that procedurally modifies path vertices each frame would be the latter approach, allowing the modifications to be recalculated and applied dynamically during rendering rather than permanently altering the mask data.

*Tags: `aegp`, `mask`, `render-loop`, `params`*

---

## What causes error code 1397908844 when rendering with multiple effects in After Effects?

According to an Adobe Community post (https://community.adobe.com/t5/after-effects-discussions/error-while-rendering-with-multiple-effects-with-code-1397908844-using-pf-newworld-pf-disposeworld/td-p/14285656), this error occurs when a plugin is accidentally double-releasing a suite. The error has been documented since at least 2007 and can also occur in other Adobe applications like Illustrator.

*Tags: `debugging`, `render-loop`, `memory`, `aegp`*

---

## Why does a blur function work differently between 8-bit and 16/32-bit color depths in After Effects?

When implementing blur effects that support multiple bit depths (8-bit, 16-bit, and 32-bit), subtle differences in how path values are converted between depth formats can cause discrepancies. One workaround is to precompose the layer with the effect first, which ensures consistent processing. The issue likely stems from improper conversion of path values when switching between 8-bit and 16/32-bit processing paths.

*Tags: `mfr`, `debugging`, `render-loop`, `memory`*

---

## What were the root causes of bugs in the FastBoxBlur implementation?

Two critical bugs were identified in FastBoxBlur: (1) Incorrect bit depth detection causing memory corruption, and (2) Improper stride handling for padded buffers. The solution involved fixing bit depth detection to use rowbytes calculation and implementing proper stride handling throughout the blur algorithm.

*Tags: `memory`, `debugging`, `render-loop`, `output-rect`*

---

## How can you create a render-only version of an After Effects plugin that doesn't require a GUI license on render farms?

One approach is to make all parameters invisible while still allowing the plugin to read them, effectively creating a read-only version of the plugin. This way, the plugin can process parameters set during the authoring phase without requiring interactive UI on render farm machines.

*Tags: `params`, `ui`, `deployment`, `render-loop`*

---

## How can you check if After Effects is running in headless render engine mode?

Use the AppSuite4 API to check the render engine flag. Call PF_IsRenderEngine() to determine if the renderer is running in headless mode, which is useful for plugins that need to behave differently during background rendering versus interactive use.

```cpp
PF_Boolean bIsRenderEngine = true;
suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine);
```

*Tags: `aegp`, `render-loop`, `debugging`, `api`*

---

## Can an After Effects effect plugin compute heavy calculations and update parameter values from the render function?

This appears to be an unsolved use case. The developer wants to perform computationally expensive operations in an effect plugin's render function and have those results update a parameter slider, but current After Effects plugin architecture doesn't provide a straightforward mechanism for plugins to push parameter updates back to the host during rendering. Parameters are typically read during render, not written to.

*Tags: `params`, `render-loop`, `aegp`, `threading`*

---

## How can you apply masks to GPU-rendered output in After Effects when the GPU API only provides masked/cropped buffers?

Eric encountered this limitation while developing the Wrap It 3D camera projection plugin. After Effects' GPU rendering API (checkout_layer_pixels) only returns post-mask-applied, cropped buffers with no GPU equivalent of PF_CHECKOUT_PARAM to access the raw unmasked source. Attempted workarounds included using PF_OutFlag2_REVEALS_ZERO_ALPHA with expanded PreRender requests, but this didn't provide the full unmasked layer to GPU. A manual CPU→GPU upload of the unmasked source defeats GPU acceleration benefits. The core limitation is that AE's native GPU world has no mechanism to provide raw unmasked source data on GPU. As a workaround, users may need to use track mattes in GPU mode, or consider building a custom GPU context (OpenGL, Vulkan, or WebGPU) outside of After Effects' native GPU pipeline to maintain full control over the source data.

*Tags: `gpu`, `layer-checkout`, `masks`, `aegp`, `render-loop`*

---

## What GPU rendering approach did Red Giant Universe migrate to from CPU-only plugins?

Red Giant Universe transitioned from CPU-only plugins with OpenGL in the background to a custom GPU renderer backend that supports Metal, DirectX, and OpenGL. They now provide GPU buffer interop for every GPU buffer type available, allowing plugins to leverage native GPU rendering across different platforms.

*Tags: `gpu`, `metal`, `opengl`, `cross-platform`, `render-loop`*

---

## How can a plugin render content that extends beyond the adjustment layer bounds?

When rendering content that exceeds the layer boundaries (such as text on an adjustment layer smaller than the composition), you need to use the output_rect parameter to specify the actual rendering region. By setting output_rect to encompass the full area where your content will be drawn, rather than constraining it to the layer bounds, the plugin can render beyond the adjustment layer's dimensions and display the full content within the composition.

*Tags: `output-rect`, `render-loop`, `aegp`, `ui`*

---

## How can a C++ plugin update a text layer every frame during rendering without locking the project?

This is a fundamental limitation in After Effects: calling AEGP_SetText from the render thread causes the project to lock. Several workarounds exist: (1) Write data to sequence data or compute cache from the render thread, then read and apply it from the UI thread using the aegp_idle hook, which is called multiple times per second; (2) Use your own text rendering engine in the render thread with an external library; (3) Write to disk and have a ScriptUI panel with a timer read it back (though this is unreliable). However, none of these work reliably in MediaEncoder or headless rendering without a GUI. The fundamental issue is that modifying a text layer from render could create infinite loops, which is why AE locks the project.

*Tags: `aegp`, `render-loop`, `threading`, `ui`, `sequence-data`, `compute-cache`*

---

## What is a reliable method to pass data from an effect plugin to a text layer?

An AEGP plugin using an idle hook is the best approach. Each frame, push string results into a queue, and periodically process the queue in the idle hook. This requires two binaries (AEGP plugin and FX plugin) communicating through a C interface. Alternatively, you can encode data into dead pixels in your render and use sampleImage expressions to retrieve those pixels and set source text, though this has caveats with render format reliability.

*Tags: `aegp`, `text-layer`, `plugin-communication`, `render-loop`, `idle-hook`*

---

## What is a method to pass text data from a render into an expression in After Effects?

One semi-reliable method is to encode and render dead pixels in your render output, then use a sampleImage expression to retrieve those pixels, which can then set the source text. This technique works with both regular layers and Null objects, though it has a few caveats.

*Tags: `scripting`, `expressions`, `render-loop`, `workaround`*

---

## What are the practical challenges and solutions when designing multi-layer effects in After Effects plugins?

According to Tobias Fleischer (reduxFX), a plugin-per-layer approach was abandoned in favor of a single plugin with multiple layer parameters due to render order troubles. The successful approach involved creating a single plugin that accepts 10 layer parameters to pull in pixels from multiple layers, applies lighting/material options, and composites everything together within that single plugin. While this worked well, AE's rigid parameter interface made it somewhat clunky to use. The key insight is that centralizing the multi-layer logic into a single plugin avoids render order dependency issues that arise from distributing logic across multiple per-layer plugins.

*Tags: `render-loop`, `params`, `ui`, `layer-checkout`, `smartfx`*

---

## Is it possible to create a multi-layer global illumination plugin that discovers material metadata effects on other layers and properly communicates dependencies to After Effects' caching system?

This is a complex architectural question posed by fad regarding whether a 'renderer' effect can enumerate other layers, discover per-layer material definition effects, check out layer pixels/masks/transforms, and have those dependencies properly tracked by AE's dependency tracking and disk cache system. While Tobias Fleischer's experience suggests multi-layer effects are viable, the specific mechanism for communicating hidden dependencies (non-explicit UI parameter dependencies) to AE's caching system to trigger rerenders when materials change remains an open design challenge requiring careful SDK integration and likely custom dependency management.

*Tags: `render-loop`, `caching`, `compute-cache`, `layer-checkout`, `smartfx`, `memory`*

---

## What approach did reduxFX use to handle multi-layer compositing in After Effects plugins?

Instead of using a plugin-per-layer approach (which caused render order issues), reduxFX implemented a single plugin with 10 layer parameters that could pull in pixels from multiple layers, apply lighting/material options, and composite everything together within that single plugin. While AE's rigid parameter interface made it somewhat clunky to use, this approach worked surprisingly well and was later ported to node-based hosts like Nuke and Natron where the multi-input approach worked more naturally.

*Tags: `params`, `render-loop`, `mfr`, `ui`*

---

## Is it possible to achieve progressive rendering in an After Effects plugin where samples render incrementally and display results to the user in real-time?

gabgren mentioned porting functionality from outside AE where progressive sampling over time with continuous result display is possible, making the interaction feel more responsive. They noted that the current AE plugin workflow requires waiting for all samples to render before seeing results on screen, which feels slow. They're exploring whether AE plugins can support this type of progressive rendering approach.

*Tags: `smart-render`, `render-loop`, `ui`, `mfr`, `performance`*

---

## What is a good approach for implementing multi-layer effect controls in After Effects plugins?

A viable user-friendly approach is to use layer parameters on the main effect combined with dummy effects on other layers, rather than requiring users to switch to artisan rendering mode. This integrates well with other effects and provides better usability while maintaining acceptable performance. Testing at quarter resolution can help evaluate performance characteristics.

*Tags: `aegp`, `params`, `ui`, `layer-checkout`, `render-loop`*

---

## How do 3D Camera Tracker and Warp Stabilizer output progress and live update without purging cache?

According to an After Effects developer, these built-in effects use internal APIs that are not available to plugin developers. The effects "cheat" and do not operate within the constraints of a normal AE Effect API plug-in. Developers have attempted workarounds involving refresh kicks, but these have proven unreliable across macOS and Windows. A feature request was made in 2023 to expose this capability to third-party developers, but it remains unavailable through the public API.

*Tags: `aegp`, `smart-render`, `caching`, `render-loop`, `debugging`, `cross-platform`*

---

## How do you handle colorspace differences between After Effects and Premiere Pro plugins?

After Effects uses ARGB colorspace while Premiere Pro uses BGRA_8u. For 8-bit BGRA, Premiere has a special suite available in the AE SDK (see the sdknoise example which demonstrates BGRA and YUVA colorspace support). For 32-bit float, you need to write your own conversion function. You can choose to support only specific colorspaces in global setup rather than all modes Premiere supports. Basic rendering in 8/16-bit works without optimization, though you won't get the 32-bit icon indicator.

*Tags: `premiere`, `colorspace`, `bgra`, `argb`, `render-loop`*

---

## How can you specify a custom output region in Premiere transitions when smartFX is not available?

Premiere does not support smartFX, which means you cannot use the smart render feature to request only a specific region of the buffer. When making transitions, Premiere will provide frames at the sequence resolution regardless of the actual footage size. This is a limitation of Premiere's plugin architecture compared to After Effects, where smartFX allows you to define output rectangles and optimize rendering to only the needed buffer region.

*Tags: `smartfx`, `premiere`, `output-rect`, `render-loop`*

---

## Does PF_ABORT() work the same way in Premiere as it does in After Effects?

Unlike in After Effects where PF_ABORT(in_data) can interrupt rendering and set err to PF_Interrupt_CANCEL, this mechanism does not work reliably in Premiere. Even when frames take significant time to render (over half a second), calling PF_ABORT() does not trigger an abort. In Premiere, you may need to finish rendering every frame that is requested rather than relying on abort functionality to interrupt the render process.

*Tags: `premiere`, `render-loop`, `aegp`, `cross-platform`*

---

## Why does Premiere pass NULL for input_world in Render when compiled with fast optimizations, but not in After Effects?

A developer reported that when compiling a plugin with fast optimizations enabled, Premiere passes input_world = NULL in the Render call, whereas without optimizations the data is present. The same code does not exhibit this problem in After Effects. This suggests the issue may be related to optimization levels affecting how Premiere handles or passes buffer data to plugins during rendering.

*Tags: `premiere`, `smartfx`, `render-loop`, `debugging`, `build`*

---

## How can you prevent the compiler from optimizing a render function that is causing issues in an After Effects plugin?

Add the `__attribute__ ((optnone))` compiler attribute before the render function declaration. This tells the compiler to skip optimization for that specific function, even if the rest of the plugin is optimized. This was identified as a workaround by the community, though it may indicate an underlying issue worth investigating further.

```cpp
__attribute__ ((optnone))
void render() {
  // render function implementation
}
```

*Tags: `build`, `debugging`, `render-loop`, `optimization`*

---

## How should ARGB_8u be defined in globalSetup for proper colorspace handling?

When using ARGB_8u colorspace, it needs to be properly defined in the globalSetup function. The user inquired about the correct definition method and suggested checking colorspace settings and adding null checks for inoutworld (if inoutworld is null return err) to see if Premiere will redo the render.

*Tags: `premiere`, `params`, `debugging`, `render-loop`*

---

## What is the issue with the no_params_vary flag and how does it differ between After Effects and Premiere?

The no_params_vary flag was identified as the origin of a problem in plugin behavior. In After Effects, this flag's behavior can be replaced using MIX_GUI during the pre_render thread. However, in Premiere, the flag does nothing, indicating different parameter variation handling between the two applications.

*Tags: `premiere`, `params`, `mfr`, `render-loop`*

---

## How can I continuously update the ECW (Enhanced Custom UI) drawing while dragging a 2D point parameter?

ECW custom UIs receive idle calls on regular intervals only when the cursor is within their perimeter. To force continuous updates during parameter interactions, call PF_RefreshAllWindows during the interaction. This forces a redraw of the ECW, though it is resource-intensive. If the point parameter is supervised, this approach will keep the ARB viewer updated in real-time as the user drags the point.

*Tags: `ui`, `ecw`, `params`, `render-loop`*

---

## How should you detect parameter changes that occur from keyframe timeline scrubbing rather than direct user interaction?

Parameter synchronization across user interaction, timeline scrubbing, and rendering presents challenges. For detecting changes during timeline scrubbing, you can use PF_Cmd_UPDATE_PARAMS_UI. However, a more robust approach is to use expressions instead of keyframes on dependent parameters. You can set expressions programmatically during UPDATE_PARAMS_UI using AEGP_SetExpression, which handles all three scenarios (user interaction, timeline scrubbing, and rendering) consistently.

*Tags: `params`, `aegp`, `render-loop`*

---

## Why does transform_world give better results when entering parameter values numerically versus dragging sliders?

When working with transform_world, numeric input (typing a value and pressing enter) produces better results than slider dragging. This is theorized to be related to how After Effects halts the transformation process during interactive slider adjustments to provide a smoother user experience, though this behavior may be legacy code from before CC2015's separate rendering thread was introduced.

*Tags: `transform_world`, `params`, `ui`, `render-loop`*

---

## Can you use transform_world multiple times in a loop with the same output buffer as input for subsequent transformations?

No, you cannot use the same buffer as both input and output for transform_world, as it reads and overwrites simultaneously, resulting in corrupted output. Instead, create a temporary buffer and alternate between buffers: transform input to temp, temp to output, output back to temp, repeating as needed. Never overwrite the original input buffer that After Effects caches, as this causes bizarre bugs.

```cpp
// Correct approach for repeated transformations
// 1. create a new buffer "temp"
// 2. transform the input to temp
ERR(in_data->utils->transform_world(in_data->effect_ref, in_data->quality, PF_MF_Alpha_STRAIGHT, in_data->field, input, &composite_mode, NULL, &matrix1, 1L, TRUE, &output->extent_hint, temp));
// 3. transform temp to output
ERR(in_data->utils->transform_world(in_data->effect_ref, in_data->quality, PF_MF_Alpha_STRAIGHT, in_data->field, temp, &composite_mode, NULL, &matrix2, 1L, TRUE, &output->extent_hint, output));
// 4. repeat by transforming output back to temp for next iteration
```

*Tags: `transform_world`, `memory`, `render-loop`, `aegp`, `debugging`*

---

## Is it better to apply multiple transform_world operations separately or combine them into a single matrix multiplication?

Always combine transformations into a single matrix multiplication rather than applying multiple transform_world renders. Matrix multiplication of 3x3 matrices is extremely cheap (~20 operations), while each transform_world render performs millions of pixel operations. Additionally, multiple sequential transforms cause image degradation (blur/jaggedness) because pixels that don't land on exact whole pixels get blended, compounding with each render. A single combined transformation applied once produces superior quality and performance.

*Tags: `transform_world`, `performance`, `optimization`, `render-loop`*

---

## Does the number of available threads returned by AEGP_GetNumThreads change during an After Effects session?

To the best of knowledge, the number of available threads does not change throughout an AE session. The decision on using PF_Iterations_ONCE_PER_PROCESSOR is an optimization decision that depends on your algorithm. For image processing, PF_Iterations_ONCE_PER_PROCESSOR is rarely optimal because some threads finish their chunk before others and cannot help with remaining work on other threads. Using n iterations instead assures minimal loss of available CPU power for most image processing algorithms.

*Tags: `threading`, `render-loop`, `aegp`, `optimization`*

---

## What are the memory safety considerations when using AEGP_RenderAndCheckoutLayerFrame_Async?

According to community discussion with Adobe's AE team, there is a known memory release problem when using the async version of renderAndCheckout. As a result, developers have reverted to using the synchronous version instead, rendering one frame at a time on each idle cycle to avoid UI lag while properly managing memory.

*Tags: `aegp`, `memory`, `async`, `render-loop`, `threading`*

---

## How can you safely implement async layer frame rendering in an AEGP plugin?

Create an AsyncManager class that wraps AEGP_RenderAndCheckoutLayerFrame_Async with proper callback handling. The async call is thread-safe as long as only one thread calls it at a time. Use a static callback function that receives the request via refcon, retrieves the world from the frame receipt, and invokes a user-provided callback function. Store the callback as a heap-allocated std::function pointer passed through the refcon parameter, then delete it after use to prevent memory leaks.

```cpp
class AsyncRenderManager {
public:
  void renderAsync(LayerRenderOptionsPtr optionsH, std::function<void(WorldPtr)> callbackF) {
    auto callbackPtr = new std::function<void(WorldPtr)>(callbackF);
    auto ret = std::async(std::launch::async, [optionsH, callbackPtr]() {
      RenderSuite().renderAndCheckoutLayerFrameAsync(optionsH, callback,
        reinterpret_cast<AEGP_AsyncFrameRequestRefcon>(callbackPtr));
    });
  }
  
  static A_Err callback(AEGP_AsyncRequestId request_id, A_Boolean was_canceled, A_Err error,
    AEGP_FrameReceiptH receiptH, AEGP_AsyncFrameRequestRefcon refconP0) {
    auto callbackPtr = reinterpret_cast<std::function<void(WorldPtr)> *>(refconP0);
    if (callbackPtr && *callbackPtr) {
      FrameReceiptPtr ptr = std::make_shared<AEGP_FrameReceiptH>(receiptH);
      auto world = RenderSuite().getReceiptWorld(ptr);
      (*callbackPtr)(world);
      delete callbackPtr;
    }
    return error;
  }
};
```

*Tags: `aegp`, `threading`, `async`, `memory`, `render-loop`*

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

## How can sequence data be safely shared and synchronized between PF_Cmd_EVENT and SmartPreRender calls?

Sequence data is intentionally separate between the render and UI threads by design, allowing render threads to execute asynchronously without data changes from other threads. To share data between them, use global data (shared across threads but not instance-specific) or store render instance data with an identifier such as comp item ID + layer ID + effect index on layer, then have the UI thread look up the correct data using a mutex. Note that arb/hidden params can only be written from the UI thread; render threads cannot modify project data.

*Tags: `sequence-data`, `threading`, `render-loop`, `aegp`, `arb-data`*

---

## How can I access decoded video frames in a specific order for face detection in an After Effects plugin?

You can request any project item's image at any time using AEGP_RenderAndCheckoutFrame and AEGP_RenderSuite4. However, After Effects renders frames on-demand and in random order by design. For sequential processing requirements, consider two approaches: (1) fast sequential pre-processing upfront at the cost of user experience, or (2) slower random-access processing for simplicity. Examples like AE's camera tracker and Lockdown perform sequential background processing while keeping the UI responsive. You may also need to use AEIO_InqNextFrameTime() to determine frame order and build a custom data structure to track the sequence.

*Tags: `aegp`, `smart-render`, `caching`, `render-loop`, `sequence-data`, `ui`*

---

## What is the primary rendering philosophy of After Effects for plugin developers?

After Effects uses an on-demand, random-access rendering scheme rather than sequential frame processing. This design prioritizes user experience by rendering only requested frames in any order. Plugins that need sequential processing must work against this architecture, but can implement background processing with progress indication (like the camera tracker) to maintain UI responsiveness while performing pre-processing tasks.

*Tags: `render-loop`, `smart-render`, `ui`, `caching`*

---

## How should I store time-dependent curve data for a custom parameter and ensure the UI updates as the user draws?

Use an arbitrary parameter (PF_ADD_ARBITRARY2) to store your data. The workflow is: user interacts with UI → data is stored in a LOCAL array → local array is serialized and stored in the arb parameter → AE automatically invalidates cached frames and re-renders. Before displaying the UI, read the value from the arb param, deserialize it into your local array, and draw accordingly. This also ensures undo/redo operations reflect correctly in the UI. You don't need to convert data to a string; serialize it into one continuous block of memory using whatever method works for your data type.

*Tags: `params`, `ui`, `arb-data`, `caching`, `render-loop`*

---

## Should I use multithreading or MFR in my After Effects plugin, or can both be used together?

Both multithreading (MT) and Multi-Frame Rendering (MFR) can coexist in plugins. While they may seem incompatible at first, MFR and MT don't necessarily compete for CPU resources because not all steps in the rendering pipeline of a single frame are multi-threadable. MFR excels at handling non-multi-threadable steps, while the multi-threadable parts are often CPU cache-friendly and don't compete with MFR threads. The Adobe After Effects engineering team has extensively tested these scenarios and optimized the mechanism to handle a wide range of rendering scenarios, including plugins that aren't MFR-enabled and those relying on GPU rendering.

*Tags: `mfr`, `threading`, `render-loop`, `sdk`*

---

## How can I create an After Effects effect that only provides settings without changing the layer visually?

There are several approaches: (1) Use AE's utils->Copy() function to copy the input buffer to the output buffer, which is fast and straightforward. (2) Set the PF_OutFlag_AUDIO_EFFECT_ONLY flag to prevent AE from calling your effect for image rendering—it will only be called for audio, which you must implement by copying input to output. This is faster than copying the image buffer. (3) Try setting the output rect size to 0 or negative during pre-render to make AE skip rendering, though this approach has inconsistent results. The simplest approach is using utils->Copy() to pass through the input unchanged.

```cpp
static PF_Err
Render(	PF_InData		*in_data,
		PF_OutData		*out_data,
		PF_ParamDef		*params[],
		PF_LayerDef		*output)
{
	PF_Err	err	= PF_Err_NONE;
	// Use utils->Copy() to copy input to output
	return err;
}
```

*Tags: `aegp`, `render-loop`, `memory`, `output-rect`*

---

## How can you force rendering from a separate thread in After Effects plugins?

You cannot directly set out_flags from a separate thread. Instead, use the idle hook, which fires 30-50 times per second on the main UI thread. From the idle hook, you can: (1) change an invisible parameter value to force re-render, (2) call command numbers to purge cache, or (3) use AEGP_EffectCallGeneric to have the effect instance act on itself. The idle hook is the recommended approach for syncing work from separate threads back to the main thread.

```cpp
// In idle hook (main thread)
// Change invisible param to trigger re-render
out_data->out_flags |= PF_OutFlag_REFRESH_UI | PF_OutFlag_FORCE_RERENDER;
```

*Tags: `threading`, `render-loop`, `aegp`, `ui`, `debugging`*

---

## How do you handle WebSocket messages in a separate thread while still triggering UI updates in After Effects?

Use an idle hook callback combined with a separate worker thread. The worker thread (e.g., using tinythread via easywsclient) handles WebSocket messages and stores state. The idle hook, which runs on the main UI thread at 30-50 Hz, polls for new messages and triggers parameter changes or cache purges. This pattern ensures all UI modifications happen on the main thread while external communication occurs asynchronously.

*Tags: `threading`, `ui`, `render-loop`, `cross-platform`*

---

## Why does setting PF_OutFlag_NON_PARAM_VARY cause memory usage to increase dramatically?

Setting PF_OutFlag_NON_PARAM_VARY or PF_OutFlag_WIDE_TIME_INPUT causes After Effects to call FrameSetup>Render>FrameSetDown for each frame instead of once, which can appear to increase memory usage significantly. However, this is often normal behavior where After Effects is caching frames. The memory will be released when purging AE's memory cache or when the memory is needed for new frame renders. Ensure you implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup, preventing accumulation across multiple frame renders.

*Tags: `memory`, `caching`, `render-loop`, `aegp`*

---

## How should I handle memory allocation when rendering multiple frames with NON_PARAM_VARY flag?

When using PF_OutFlag_NON_PARAM_VARY to render multiple frames, you must implement PF_Cmd_FRAME_SETDOWN to properly free memory allocated during PF_Cmd_FRAME_SETUP. This prevents memory accumulation as FrameSetup and FrameSetDown are called for each frame. Without implementing FRAME_SETDOWN, allocated memory from each frame setup will accumulate rather than being released between frames.

*Tags: `memory`, `render-loop`, `aegp`, `debugging`*

---

## How can a C++ effect plugin modify transform parameters of other layers in a project?

Use the StreamSuite to affect any parameter (which are internally streams) in the project. The 'project dumper' sample project demonstrates how to access streams. However, since CC2015, you cannot change the project during a render call. To modify project parameters, use UI events (anything other than render and pre-render calls) when you can modify the project as needed.

*Tags: `aegp`, `params`, `render-loop`, `streaming`, `sdk`*

---

## Why does a checked-out layer get resized and positioned incorrectly when the effect layer is moved and re-rendered?

The issue occurs because when a layer is moved partially outside the composition, After Effects optimizes rendering by asking plugins to render only the visible rectangle. If you checkout the full layer and copy it to a smaller output buffer, the copy operation resizes the source buffer to match the destination buffer size, causing the layer to be squeezed. The solution is to calculate the offset between the layer position and composition center, then use composite_rect() instead of copy() to properly position the checked-out layer in the output buffer while respecting the output_worldP->origin_x and output_worldP->origin_y values.

```cpp
// Calculate offset
A_long x = (A_long)(effectLayerData.position.x - in_data->width / 2);
A_long y = (A_long)(effectLayerData.position.y - in_data->height / 2);

// Use composite_rect instead of copy, and check origin values
err = worldTransformSuite->composite_rect(in_data->effect_ref,
  in_data->quality,
  PF_MF_Alpha_STRAIGHT,
  in_data->field,
  &checkout_layer_worldP->extent_hint,
  checkout_layer_worldP,
  &compMode,
  NULL,
  output_worldP);
```

*Tags: `layer-checkout`, `aegp`, `render-loop`, `output-rect`*

---

## How can I display text in an After Effects plugin render buffer?

After Effects' API doesn't offer built-in text generating functions for the render buffer (unlike the UI buffer which has the drawbot suite). However, you can fill the render buffer using OS-native text tools: use GDI+ on Windows or Quartz on macOS to generate text in a native OS buffer, then copy that content back to After Effects' render buffer.

*Tags: `ui`, `windows`, `macos`, `render-loop`, `output-rect`*

---

## How should you properly store and reuse a frame snapshot across multiple renders in an AE plugin?

You can create a temporary EffectWorld and store it in sequence_data, then use it as the output for your effect across multiple frames. Alternatively, you can access the pixel data directly via effect_worldP->data (the base address of the world's buffer), save it to a file using a library like tinypng, and then import it back as needed. You should avoid copying data directly onto a layer's source buffer because AE may refresh that buffer without the plugin's knowledge, and the buffer may have already been cached elsewhere in the pipeline.

```cpp
effect_worldP->data  // base address of the world's buffer for direct pixel access
```

*Tags: `sequence-data`, `memory`, `render-loop`, `aegp`, `caching`*

---

## Why doesn't changing a plugin parameter invalidate cached frames in After Effects?

This is actually normal After Effects behavior. When a parameter changes, the composition render cache is invalidated and the displayed result is correct. However, After Effects intentionally does not release the RAM of previously cached frames—it keeps old results in memory in case the user undoes changes, until RAM is needed for something else. To verify this, purge After Effects' RAM and the memory usage will drop back to the original level, confirming the RAM accumulation is AE's design and not a memory leak.

*Tags: `caching`, `memory`, `params`, `render-loop`*

---

## When handling parameter changes with PF_Cmd_USER_CHANGED_PARAM, what flag should be set to force re-rendering?

Set the `PF_OutFlag_FORCE_RERENDER` flag in `out_data->out_flags` at the end of your parameter change handler to force After Effects to re-render the layer when the parameter is modified.

```cpp
case PF_Cmd_USER_CHANGED_PARAM:
  err = HandleEvent(in_data, out_data, params, output, reinterpret_cast<const PF_UserChangedParamExtra*>(extra));
  break;

// In HandleEvent:
out_data->out_flags = out_data->out_flags | PF_OutFlag_FORCE_RERENDER;
```

*Tags: `params`, `render-loop`, `aegp`*

---

## Why does memory consumption increase drastically when playing an After Effects plugin and not return to normal after removing the effect layers?

The memory increase is likely not a leak but rather After Effects caching rendered frames. To verify this, purge RAM after playing or previewing the composition—if memory returns to the pre-play level after purging, it confirms the memory was being held by the cache, not leaked by the plugin. If a parameter change should invalidate cached frames but doesn't, you can force invalidation by changing the value of an invisible parameter.

*Tags: `memory`, `caching`, `render-loop`, `debugging`*

---

## Will changing non-PUI_ONLY parameters trigger an automatic re-render in After Effects?

Yes, modifying any parameter that is not marked as PUI_ONLY should automatically trigger a re-render. If a re-render is not occurring after parameter changes via SetStreamValue, verify that the modified parameters are not PUI_ONLY flags.

*Tags: `params`, `render-loop`, `smart-render`*

---

## If SmartFX is implemented in a 32-bit After Effects plugin, do I still need to implement the Render() event?

If SmartFX is implemented, After Effects does not send RENDER or FRAME_SETUP calls anymore; instead it sends Smart Render and Pre-render calls. However, if you need Premiere Pro compatibility, you should keep both mechanisms implemented since Premiere Pro does not use SmartFX and still calls Render and Frame Setup events.

*Tags: `smartfx`, `render-loop`, `premiere`, `aegp`*

---

## Why don't slave parameters update when the master parameter is animated and scrubbed through the timeline?

Since CC2015, the render thread is not allowed to modify the project in any way, which prevents parameter updates during rendering. Additionally, with MFR (Multiple Frame Rendering), multiple frames are rendered simultaneously, making it ambiguous what value a parameter should have. If the slave parameter only affects the display UI and doesn't influence the actual render, you can work around this by: (1) creating a custom UI that redraws on time changes using its draw event, or (2) creating an AEGP with an idle_hook that scans the current composition for your effect and modifies it as needed. However, if the slave parameter needs to affect the render or be read by expressions in other effects/layers, this design pattern is not supported by After Effects' workflow.

*Tags: `params`, `render-loop`, `mfr`, `ui`, `aegp`*

---

## Why does a plugin's blur effect keep increasing when changing iteration parameters?

The plugin was writing into the input layer buffer and corrupting it. After Effects provides the original input buffer, not a copy that can be modified. When iterating the blur in a loop and changing the iteration parameter, the corrupted input from the previous operation causes cumulative blur effects. The issue stems from assigning the input layer directly (PF_EffectWorld tmpWrld = params[0]->u.ld) which creates a reference rather than a true copy.

```cpp
// Incorrect - creates reference:
PF_EffectWorld tmpWrld = params[0]->u.ld;

// Correct approach - allocate new buffer:
// AEGP_WorldSuite3->AEGP_New()
// AEGP_FillOutPFEffectWorld()  // creates a PF_EffectWorld wrapper for the allocated AEGP_WorldH
```

*Tags: `params`, `memory`, `render-loop`, `aegp`*

---

## What is the correct way to bypass rendering in an After Effects effect plugin?

To bypass rendering in an AE effect plugin, you have a few options: (1) Try setting the output rectangle during pre-render to a 0-sized rect or a rect where the right value is smaller than the left value to tell AE to skip the render, though this approach is not well-documented and may require experimentation. (2) The more reliable and practical approach is to manually copy the input buffer to the output buffer in your render function. According to community experts, the overhead of copying is negligible even when stacking multiple such effects, making this a pragmatic solution that works reliably on both audio and non-audio layers.

*Tags: `render-loop`, `output-rect`, `aegp`, `reference`*

---

## How should you handle frame interruption in After Effects plugins to avoid crashes when advancing frames quickly?

When a user interrupts frame processing (by advancing to another frame, clicking buttons, etc.), After Effects does not forcefully kill the rendering thread. Instead, the plugin must call PF_ABORT() during the render call (not pre-render) to check if AE wants processing to stop. If PF_ABORT returns true, you should return a PF_Interrupt_CANCEL error message to tell AE the output buffer should not be used. Crashes during interruption are often caused by re-entrancy issues—if your crash occurs when accessing shared data structures like matrices, ensure thread-safety by using mutexes or allocating structures on the stack rather than globally. Call PF_ABORT periodically during lengthy processing to maintain responsive interactivity while scrubbing or changing parameters.

```cpp
if (err = PF_ABORT(in_dataP)) {
  err = PF_Interrupt_CANCEL;
  return err;  // Must return after setting the error
}
```

*Tags: `threading`, `render-loop`, `memory`, `debugging`, `aegp`*

---

## Why does my C++ plugin show the wrong initial frame when loading, and how do I fix it?

The issue is likely caused by AE's dynamic resolution feature, which automatically reduces the composition's resolution to maintain interactive speeds. You can verify this by disabling it in the composition window settings (lightning icon). However, the real solution is to implement smartFX and report a GUID during SmartPreRender to tell AE when a frame differs from what it has cached. Additionally, you must account for the downsample factors passed in the in_data struct, which is common practice for AE plugin developers.

*Tags: `smartfx`, `caching`, `render-loop`, `debugging`*

---

## Why do I see artifacts and previously rendered shapes appearing in my plugin output when modifying parameters?

The issue stems from not properly accounting for AE's dynamic resolution feature and downsample factors. When drawing with cairo and copying to the AE buffer, you need to handle the downsample factors from the in_data struct. Additionally, ensure you're properly clearing the canvas at the beginning of each render pass and not relying on cached buffers. The enlargement of shapes during computation is typically a symptom of dynamic resolution being applied without proper downsample factor handling in your pixel copy operations.

```cpp
PF_FILL(NULL, NULL, output); // Clean the canvas at the beginning
cairoRefcon.data = cairo_image_surface_get_data(surface);
cairoRefcon.stride = cairo_image_surface_get_stride(surface);
cairoRefcon.output = *output;
ERR(suites.IterateSuite1()->AEGP_IterateGeneric(output->height, &cairoRefcon, cairoCopy8));
```

*Tags: `caching`, `render-loop`, `params`, `debugging`, `memory`*

---

## How can I make a button parameter call UserChangedParam without triggering a render?

Use the PF_PUI_STD_CONTROL_ONLY flag on your button parameter. This flag prevents render calls when the button is clicked. Note that this flag has side effects: parameter changes won't trigger renders, the parameter won't be keyframable, and the value won't be stored with the project (resets to default on reload).

```cpp
PF_PUI_STD_CONTROL_ONLY
```

*Tags: `params`, `ui`, `render-loop`*

---

## How do you animate shapes in an After Effects plugin as the user moves the timeline slider?

Use in_data->current_time to get the current frame's time step and in_data->time_step to get the number of steps per second. AE calls the plugin to render a frame with a given time stamp. When rendering a sequence, AE calls the plugin multiple times, each with a corresponding timestamp. You can then compute your shape function shape(t) for each requested frame time, allowing dynamic shapes with varying numbers of paths over time.

```cpp
double x = 100;
double y = 100;
double R = 50;
double phi1 = 0.;
double phi2 = 2*MF_PI;
cairo_t *cr = cairo_create(surface);
cairo_arc(cr, x, y, R, phi1, phi2);
// Use in_data->current_time to get frame time and call shape(t) for animation
```

*Tags: `mfr`, `params`, `render-loop`, `aegp`*

---

## How does sequence data get synchronized between the UI thread and render thread in Smart Rendering?

Sequence data synchronization between UI and render threads happens at specific times only: when a new instance is created and during specific UI events. Synchronization occurs when PF_Cmd_USER_CHANGED_PARAM is triggered, and in CLICK and DRAG events if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented. The mechanism involves PF_Cmd_GET_FLATTENED_SEQUENCE_DATA and PF_Cmd_SEQUENCE_RESETUP for flattening and unflattening respectively. However, PreRendering and SmartRendering do not automatically trigger this flattening. For reliable data persistence, Adobe recommends using invisible arb parameters which sync on every change, or implementing a message-passing mechanism through global data structures accessible to both threads.

*Tags: `sequence-data`, `smart-render`, `arb-data`, `threading`, `render-loop`*

---

## How can you refresh the UI to show parameter changes made in PF_Cmd_USER_CHANGED_PARAM?

The PF_OutFlag_REFRESH_UI flag can only be set from specific command selectors (PF_Cmd_EVENT, PF_Cmd_RENDER, PF_Cmd_DO_DIALOG), not from PF_Cmd_USER_CHANGED_PARAM where parameter changes typically need to be reflected. One workaround is to use AEGP_ExecuteScript() to execute JavaScript that triggers a UI update, or restructure the plugin logic to defer UI updates until one of the supported command selectors is called.

*Tags: `params`, `ui`, `render-loop`, `aegp`, `scripting`*

---

## How can I create off-screen image buffers in After Effects SDK and draw to them?

It is possible to create off-screen image buffers in After Effects SDK. You can use intermediate buffers by working with the output buffer handed to the effect. The buffer has a 'data' pointer for the base address of image pixels and a 'rowbytes' variable for the number of bytes per row (stride). You can draw to custom buffers and then copy the result into the output buffer. After Effects provides native buffers called 'worlds' in the SDK documentation that can be used as intermediates.

*Tags: `sdk`, `memory`, `render-loop`, `output-rect`*

---

## How can I force an overlay to refresh when ARB parameter data changes without requiring mouse movement?

Use the PF_Event_IDLE event which is automatically sent to composition UIs. In the idle event handler, check system time or other factors to determine if a refresh is needed. If so, set PF_OutFlag_REFRESH_UI or call PF_InvalidateRect(), which will trigger a PF_Event_DRAW call to redraw the overlay and reflect the updated ARB data.

*Tags: `ui`, `arb-data`, `aegp`, `render-loop`*

---

## Can I call a command selector like PF_Cmd_DO_DIALOG in the render function?

No, since CC2015 the render thread cannot perform operations that change the project. Instead, you should use alternative approaches: (1) Use an idle hook to send a message from the render thread to an AEGP, which then creates a window during the idle hook call when it's safe. (2) Use SEQUENCE_SETUP, which is called whenever a new instance of your effect is applied. (3) Use GLOBAL_SETUP, which is called once per session when the effect is first applied. These approaches allow you to open dialogs for user input (like username/password) without trying to trigger commands from the render thread.

*Tags: `render-loop`, `aegp`, `threading`, `ui`*

---

## How do you force re-rendering when masks are deleted or added in an AEGP plugin?

Set the PF_OutFlag2_DEPENDS_ON_UNREFERENCED_MASKS flag in your plugin's global setup. This tells After Effects that your effect depends on mask information even when those masks aren't directly referenced as parameters, ensuring the render refreshes when masks change.

*Tags: `aegp`, `mfr`, `params`, `render-loop`*

---

## Why doesn't PF_Cmd_RENDER execute when an audio plugin uses PF_OutFlag_I_USE_AUDIO on a precomposed audio-only layer?

When a plugin sets PF_OutFlag_I_USE_AUDIO and the source layer is precomposed with no visual layers, After Effects may not trigger PF_Cmd_RENDER. The solution is to add PF_OutFlag_NON_PARAM_VARY to the global output flags in the plugin's global setup. This flag tells AE that the plugin's output varies based on non-parameter data (in this case, audio data), ensuring render commands are properly triggered even when the precomp contains only audio. Note: This flag is not required when the source layer is not precomposed, as AE automatically handles render triggering in that case.

*Tags: `audio`, `params`, `aegp`, `render-loop`, `layer-checkout`*

---

## Why is arbitrary data always showing default values in PF_cmd_Render and PF_cmd_SmartRender, but correct values in PF_cmd_User_changed_params?

The issue is typically in how the arbitrary data handle is being allocated and passed back to After Effects. Arbitrary data should be flat—the entire data should be one big handle, not a handle containing other handles. When allocating new arbitrary data, create a single handle of the required size (data size + null termination), copy the data into it directly, and pass it to AE. When you pass the handle as an arb value, it becomes owned by After Effects, so do not unlock or release it. Also, use AEGP_GetNewStreamValue instead of PF_checkout_param to retrieve parameter values.

```cpp
PF_Handle newArbH = PF_NEW_HANDLE(newString.length() + 1);
A_char* newExpMemP = (char*)newArbH;
strcpy(newExpMemP, newString.c_str());
params[SKELETON_ARBITRARY_EXPRESSION]->u.arb_d.value = newArbH;
// Do not unlock/release - handle now belongs to AE
```

*Tags: `arb-data`, `params`, `render-loop`, `memory`*

---

## What does the max_sleepPL argument in an idle hook function represent?

The A_long *max_sleepPL argument in an idle hook function represents the maximum sleep time in units of 1/60 seconds. It functions as an upper bound ('wait no longer than X * 1/60 seconds') rather than a fixed duration, providing a way to control idle hook execution frequency without requiring longer waits.

*Tags: `aegp`, `render-loop`, `threading`*

---

## How can I debug intermittent C++ exceptions that occur between PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_RENDER?

According to community expert shachar carmi, intermittent exceptions at the same memory address are likely caused by stack overflow or buffer overrun. The recommended debugging strategy is to use binary search: disable half your code and test to see if the problem manifests, then progressively narrow down the suspect code by testing the remaining halves. This methodical approach helps isolate the problematic section without relying on traditional debuggers, which may not be effective for these types of memory issues.

*Tags: `debugging`, `memory`, `render-loop`, `windows`*

---

## Does using AEGP_SetSelection force a render, and how can I avoid it?

AEGP_SetSelection should not force a render by itself. If you're experiencing unwanted renders when calling this function, it's likely due to another plugin in the composition that relies on collection state as part of its pre-render GUID, or a bug in your own code (such as missing break statements in case statements). The unexpected render behavior is probably not caused by AEGP_SetSelection directly, but by dependent plugins or logic that responds to the selection change.

*Tags: `aegp`, `render-loop`, `debugging`*

---

## What functionality should be implemented in the Render function versus the 8-bit and 16-bit functions?

Your plugin's entry point function gets invoked with a RENDER or SMART_RENDER call. As long as your plugin responds to that call, you can implement whatever functionality you need. There is no strict separation required—you have flexibility in how you organize your plugin's main functionality across these functions.

*Tags: `render-loop`, `aegp`, `pipl`, `reference`*

---

## What happens when a user interrupts rendering by scrubbing the playhead in After Effects?

When the user interrupts rendering by moving the playhead, After Effects does not wait for the render function to complete, and frame setdown is not called in-between. The plugin should call PF_ABORT to check if After Effects wants to quit the current frame's render. If the plugin decides to honor the abort request, it should return PF_Interrupt_CANCEL as the error code from the main entry point.

*Tags: `render-loop`, `aegp`, `interruption`, `mfr`*

---

## What happens if a plugin doesn't check for interrupt with PF_ABORT during rendering?

After Effects does not forcibly kill plugins in mid-call. Instead, AE tells the plugin it wants to abort and waits for the render call to return. The call should return either PF_ErrNONE or PF_Interrupt_CANCEL. If PF_ErrNONE is returned, AE might cache that result. If PF_Interrupt_CANCEL is returned, AE will discard the output buffer content as it has been reported as invalid.

*Tags: `render-loop`, `aegp`, `caching`, `mfr`*

---

## Why is sequence_data different between PF_Cmd_RENDER and PF_Cmd_USER_CHANGED_PARAM in After Effects CC 2015 and later?

In After Effects CC 2015 and later, the render thread and UI thread have separate sequence_data handles that only synchronize at specific occasions. This is a design change from earlier versions like CS5.5. To force synchronization, set PF_OutFlag_FORCE_RERENDER during UserChangedParam, which forces a flatten call on the UI thread and an unflatten on the render thread, syncing the render thread's sequence data with the UI data.

*Tags: `sequence-data`, `threading`, `params`, `render-loop`, `ui`*

---

## How can data be shared between render and UI threads if sequence_data synchronization is not reliable?

Global data is shared between threads and can be used as an alternative solution for render-to-UI-thread communication when sequence_data synchronization is not reliable. However, relying on updating UI sequence_data from the render function is not supported by design in CC 2015 and later versions.

*Tags: `threading`, `memory`, `render-loop`, `ui`*

---

## How can I force a rerender after PF_Cmd_SEQUENCE_RESETUP or PF_Cmd_SEQUENCE_SETUP when effect state changes?

Use GuidMixInPtr to incorporate custom state flags (such as a watermarked or license status flag) into the state that After Effects uses to determine whether a cached frame is valid. When you push the relevant flag into the mix, After Effects will automatically trigger a re-render when that state changes. This feature is documented in the CC2015 SDK and later versions.

*Tags: `sequence-data`, `caching`, `compute-cache`, `sdk`, `render-loop`*

---

## What is the performance difference between using a JavaScript scheduled task versus idle_hook for detecting changes in After Effects?

While a direct performance comparison was not formally tested, idle_hook is intuitively less resource-intensive than a scheduled task. For AEGPs, the update menu hook may be called when selections change, though it's uncertain whether it fires when the selection changes or only when the relevant menu is exposed.

*Tags: `aegp`, `scripting`, `performance`, `render-loop`*

---

## How can I access suite functions from within a pixel iteration function?

Use the refcon (reference context) argument of the iterate() function to pass a struct containing pointers to the suites and any other data you need. Define a struct with your suite references and other variables, pass a pointer to an instance of this struct as the refcon argument to iterate(), and then cast it back to your struct type inside the pixel function to access the suites.

```cpp
typedef struct {
  PF_in_data *in_data;
  AEGP_Suites &suites;
  bool someVariable;
  float someOtherVariable;
} myIterationData;

// In calling function
myIterationData data;
data.suites = &suites;
data.someVariable = FALSE;
suites.Iterate16Suite1()->iterate(
  arg,
  arg,
  arg,
  (void*)&data
);

// In pixel function
myIterationData &data = *(myIterationData*)refcon;
if(data.someVariable) {
  data.suites.whateverSuite()->
}
```

*Tags: `aegp`, `mfr`, `reference`, `render-loop`*

---

## When should I use PreRender and SmartRender in After Effects plugins, and what are their benefits?

PreRender (PF_Cmd_SMART_PRE_RENDER) and SmartRender (PF_Cmd_SMART_RENDER) are supported only in After Effects and are mandatory if you wish to support 32bpc (32-bit per channel) rendering. They also offer rendering pipeline improvements. However, if your plugin needs to work in Premiere Pro as well, you must support the old rendering pipeline with FrameSetup() and regular Render() calls instead.

*Tags: `smart-render`, `mfr`, `ae`, `render-loop`, `reference`*

---

## Is there a flag to disable multithreading support for an After Effects plugin?

No, there is no flag to disable multithreading for plugins. All plugins are expected to handle multi-threaded rendering. If your render code is not thread-safe, you could use a mutex at the top of your rendering function to allow only one thread at a time, but this would severely cripple performance. The recommended approach is to make your plugin's render function multi-threadable instead.

*Tags: `threading`, `render-loop`, `aegp`*

---

## How should plugins handle non-thread-safe code in the render function?

While there is no way to disable multithreading entirely, if your code is not thread-safe, you can use a mutex at the top of your rendering function to serialize access and allow only one thread at a time to execute the critical section. However, this approach significantly reduces performance benefits of multithreading. The better solution is to refactor your plugin to make the render function properly thread-safe and multi-threadable.

*Tags: `threading`, `render-loop`, `memory`*

---

## How can I optimize subpixel sampling performance in After Effects plugins?

To optimize subpixel sampling, acquire the sampling suite once before your rendering loop using AEFX_AcquireSuite, get a direct pointer to it, and release it afterwards. Avoid acquiring and releasing the suite for each sample operation. Alternatively, define the AEGP_SuitesHandler before the loop and pass a pointer to it to the sampling function, though acquiring the suite directly is likely faster since it reduces function call overhead.

*Tags: `mfr`, `aegp`, `performance`, `sampling`, `render-loop`*

---

## How can an effect plugin listen to changes in another layer and react to them?

An effect plugin can monitor changes in another layer by creating an invisible parameter with an expression that links to a source parameter on the target layer. When the source parameter changes (e.g., text content), the invisible parameter will update and trigger a re-render. To ensure the expression evaluates correctly, use PF_PUI_INVISIBLE to mark the parameter as invisible rather than AEGP_GetDynamicStreamFlags.

*Tags: `params`, `ui`, `render-loop`, `aegp`*

---

## How can I access output width and height values in a UI event handler?

The out_data.width and out_data.height values are only available during rendering and will be 0 in UI event handlers. To access these values in a DragHandle or DoClick event, you need to cache the desired values in sequence_data during the render call, then retrieve them during the event call. Note that in After Effects 13.5+, sequence_data cannot be changed from the render thread—only changes from the UI thread persist to the project and render thread.

*Tags: `ui`, `render-loop`, `sequence-data`, `threading`, `params`*

---

## How can I share data between the render thread and UI event handlers?

Since After Effects 13.5, sequence_data modified in the render thread cannot be accessed in UI threads. Two workarounds are: (1) add a pointer to sequence_data that is shared between threads (requiring logic to distinguish live pointers from saved invalid ones), or (2) store data in the global_data handle which is shared between threads and instances (requiring logic to identify which data belongs to which instance). The recommended approach is caching values in sequence_data during render and reading them back in event handlers from the UI thread.

*Tags: `sequence-data`, `threading`, `arb-data`, `ui`, `render-loop`*

---

## How can a plugin detect when a RAM preview starts in After Effects?

Use command_hook with command number 2285 to detect when a RAM preview begins. Command 2415 can be used to detect Play (spacebar) events. Note that PF_Cmd_RENDER may not be called for cached frames during RAM preview.

*Tags: `aegp`, `render-loop`, `debugging`, `caching`*

---

## What are the best practices for running computationally intensive algorithms during the PF_Cmd_RENDER call in After Effects plugins?

During a render call, you can execute any code except project modifications. For memory allocation, use After Effects' memory and handle suites rather than standard system allocation, otherwise you compete with AE for resources and risk crashes. If you get a hard crash error like 'Crash in progress', it often indicates failed memory allocation. Debug your code during an AE session to identify issues. Ensure your algorithm works correctly and uses AE's memory management APIs.

*Tags: `render-loop`, `memory`, `debugging`, `aegp`*

---

## Why doesn't changing a layer parameter's path trigger a re-render in After Effects?

Layer parameters track changes in the layer's source pixels only, not transforms, masks, or effects. This is intended behavior in After Effects. A layer selector provides the selected layer's source pixels, and only changes to those pixels trigger re-renders. To work around this, you can use an expression that checks the path value and drives a slider parameter to force updates, or alternatively create a light parented to the tracked layer with the I_USE_3D_LIGHTS flag set (though this approach has its own complications).

*Tags: `params`, `layer-checkout`, `render-loop`, `aegp`*

---

## Why is my image appearing smeared when passing vector<int> pixel data through an iteration function in After Effects?

The issue is likely related to incorrect pixel data ordering or indexing assumptions. The user was storing pixels in one order (x0y0, x0y1, x0y2... x1y0, x1y1, x1y2...) but the iteration function was expecting them in a different order (x0y0, x1y0, x2y0... x0y1, x1y1, x2y1...), causing the image to appear distorted or on its side. To debug, simplify the project to verify that a basic pixel copy (outP = *inP) works correctly first, then incrementally add complexity to identify where the indexing breaks down.

```cpp
static PF_Pixel8
*getXY(PF_EffectWorld &def, int x , int y){
  return (PF_Pixel*)def.data + y * (def.rowbytes / sizeof(PF_Pixel)) + x;
}
```

*Tags: `render-loop`, `memory`, `debugging`, `output-rect`*

---

## How should pixel data be correctly indexed when extracting values from a PF_EffectWorld and storing them in a vector for later retrieval?

When extracting pixel data from a PF_EffectWorld, use proper pointer arithmetic accounting for rowbytes: cast the data pointer to PF_Pixel, then offset by (y * (rowbytes / sizeof(PF_Pixel)) + x). Store pixels in a consistent order (RGBA for each pixel sequentially). When retrieving from the vector in the iteration function, ensure the indexing matches: for pixel at (x, y), access indices starting at ((y * width + x) * 4) for RGBA values in order. The key is maintaining consistency between storage order and retrieval order.

```cpp
// Storage in vector
for(int i = 0; i < tInfo.width; i++){
  for(int j = 0; j < tInfo.height; j++){
    PF_Pixel8 currentPixel = *getXY(*tInfo.input, i, j);
    data.push_back(currentPixel.red);
    data.push_back(currentPixel.green);
    data.push_back(currentPixel.blue);
    data.push_back(currentPixel.alpha);
  }
}

// Retrieval in iteration function
int idx = (yL * tInfo.width + xL) * 4;
outP->red = tInfo->data[idx];
outP->green = tInfo->data[idx + 1];
outP->blue = tInfo->data[idx + 2];
outP->alpha = tInfo->data[idx + 3];
```

*Tags: `memory`, `render-loop`, `output-rect`, `debugging`*

---

## How can I resize the output buffer in an After Effects plugin and determine the correct size before rendering?

You cannot access the input image during Frame_Setup, but you have two options: (1) Save necessary data during the render call and trigger a re-render using PF_ForceRerender from AdvItemSuite, during which you can change the output size; or (2) Use the layer dimensions from in_data->width and in_data->height, which represent the unaffected layer dimensions. For more advanced control, consider migrating to SmartFX, which uses pre-render calls instead of Frame_Setup and supports 32-bit color depth.

*Tags: `output-rect`, `frame-setup`, `render-loop`, `aegp`*

---

## Can I change the output size during a re-render in After Effects plugins?

Yes, you can change the output size during a re-render. When you call for a re-render using PF_ForceRerender, you receive a Frame_Setup call first, during which you can modify the size even if the frame was previously rendered. This allows you to adjust output dimensions based on calculations from the previous render pass.

*Tags: `output-rect`, `frame-setup`, `aegp`, `render-loop`*

---

## In what order does After Effects render frames when using the Render button?

After Effects does not necessarily render frames in sequential order when you hit the Render button. The application uses an internal algorithm to determine which frames should be rendered based on what is already cached in memory, which can result in non-sequential frame rendering order.

*Tags: `smart-render`, `render-loop`, `caching`*

---

## How can I iterate through 16-bit and 32-bit per channel pixels directly without using the iterate suites?

You can iterate 16bpc and 32bpc pixels directly by casting the buffer data to the appropriate pixel type. For 16-bit pixels, use: `PF_Pixel16 pix = (PF_Pixel16)buffer->data + xCount + yCount * (buffer->rowbytes / sizeof(Pixel16));`. Convert similarly for 32-bit pixels. The rest of the process is the same as in the CCU sample. Replace all PF_Pixel8 references with PF_Pixel16 or PF_Pixel32 as needed, and update constants like PF_MAX_CHAN8 to PF_MAX_CHAN16 (or 1.0f for 32-bit). For 16-bit support, set PF_OutFlag_DEEP_COLOR_AWARE in out_data->out_flags. For 32-bit support, use SmartFX (see the smartyPants sample).

```cpp
PF_Pixel16 pix = (PF_Pixel16)buffer->data + xCount + yCount * (buffer->rowbytes / sizeof(Pixel16));
```

*Tags: `mfr`, `memory`, `render-loop`, `output-rect`*

---

## How can I ensure a function runs after parameter values are updated in an After Effects plugin?

Instead of directly modifying parameter values and setting change flags, use AEGP_SetStreamValue() for instantaneous changes. There is no function guaranteed to be called after values change, but UPDATE_PARAMS_UI might get called and a render call is likely. Alternatively, you can set a flag in the sequence data to notify yourself of post-change operations that need to take place during UPDATE_PARAMS_UI or render calls.

```cpp
params[POINT]->u.td.x_value = FLOAT2FIX(pointX);
params[POINT]->u.td.y_value = FLOAT2FIX(pointY);
params[POINT]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `sequence-data`, `render-loop`*

---

## How can I check out only a portion of a frame at a different time instead of the entire frame?

You can use checkout_layer_pixels() which takes a request info structure as an argument that allows you to request a smaller rectangle. Alternatively, use the render suite to request the source item directly and set an ROI (region of interest) to get only the part you need. The render suite method doesn't hint AE in advance about your intentions, so AE won't cache items in advance—test both methods to see which is faster for your use case, as getting a potentially cached full frame might sometimes be faster than requesting a partial frame without caching.

*Tags: `layer-checkout`, `aegp`, `caching`, `output-rect`, `memory`, `render-loop`*

---

## How should you safely read camera data from an effect plugin to avoid crashes during parameter dragging?

Use PF_ABORT before reading camera data to check if an abort signal has been triggered. If you get an abort signal, skip reading the data. This helps prevent crashes that can occur during parameter dragging operations, particularly when dealing with composition cameras and keyframing operations.

*Tags: `aegp`, `params`, `debugging`, `render-loop`*

---

## Why does calling AEGP_ExecuteScript from within an effect plugin crash After Effects?

Executing script commands that modify the scene state (like applying a preset to a layer) from within an effect plugin causes a crash because the script invalidates locked/checked-out memory objects that are currently passed to the effect plugin via in_data. The effect is in mid-call when the scene state changes, creating memory conflicts. You cannot add an effect to a layer while your effect is currently executing. The solution is to defer scene modifications until after the effect has finished execution.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

## How do I rasterize a mask and use it to restrict pixel operations in my effect plugin?

You can rasterize a mask by filling a dummy buffer with color and applying the mask using the MaskWorldWithPath function from the mask suite. However, this requires the mask mode to be set to something other than NONE. A workaround is to use a temporary buffer for rasterization without affecting the project state. Note that calling AEGP_SetMaskMode is undoable and will prompt users to save changes. For querying whether pixels are contained in a mask, rasterize the mask extent into a temporary buffer and query it directly.

*Tags: `mask`, `aegp`, `memory`, `render-loop`*

---

## Can you access the comp overlay buffer during a render call using Drawbot?

No, you cannot access the comp overlay buffer during a render call, regardless of whether you're using Drawbot. Instead, you must cache whatever you want to render during a "draw" event call, and then use that cache during the render call. Drawbot's data structures are opaque and don't offer tools for reading data back out. If you need to draw during render, consider using third-party drawing tools or OS-level drawing tools instead, such as OpenGL.

*Tags: `drawbot`, `render-loop`, `caching`, `ui`*

---

## How do you properly calculate row bytes and gutters when converting pixel iteration code from 8-bit to 16-bit in After Effects?

When converting from 8-bit to 16-bit rendering, you need to adjust your row byte calculations to account for the different pixel sizes. For 8-bit (PF_Pixel8), divide rowbytes by sizeof(PF_Pixel8) to get the row width in pixels. The same principle applies to 16-bit (PF_Pixel16), but you must use sizeof(PF_Pixel16) instead. The gutter is calculated as: gutter = (rowbytes / pixelSize) - width. Common issues arise from not properly accounting for the pixel size difference between bit depths, which can cause row byte calculations to be incorrect and lead to rendering failures.

```cpp
A_long sizeOfPF_Pixel8 = sizeof(PF_Pixel8);
A_long sizeInputRowBytes = (input_worldP->rowbytes / sizeOfPF_Pixel8);
A_long sizeOutputRowBytes = (output_worldP->rowbytes / sizeOfPF_Pixel8);
in_gutterL = sizeInputRowBytes - input_worldP->width;
out_gutterL = sizeOutputRowBytes - output_worldP->width;
```

*Tags: `mfr`, `render-loop`, `memory`, `reference`*

---

## Why does buffer height calculation blow up to around 500 when iterating through rows with zero gutter values?

The issue was caused by incorrect casting of the world as the base address for pixels. The correct approach is to use AEGP_GetBaseAddr32(worldH, &baseAddress) to properly obtain the base address for pixel access, rather than casting the world handle directly.

```cpp
ERR(ws3P->AEGP_GetBaseAddr32(worldH, &baseAddress));
```

*Tags: `memory`, `buffer`, `render-loop`, `aegp`, `debugging`*

---

## What flag should be used when creating a new world buffer for deep color pixel processing?

When creating a new world for processing, use the PF_NewWorldFlag_DEEP_PIXELS flag instead of PF_NewWorldFlag_NONE to properly allocate buffers for 16-bit and 32-bit color depths. This ensures the world is correctly initialized for deep color operations.

```cpp
ERR(in_data->utils->new_world(in_data->effect_ref, width, height, PF_NewWorldFlag_DEEP_PIXELS, &outputWorld));
```

*Tags: `memory`, `params`, `render-loop`, `aegp`*

---

## How can you handle PF_ColorDef values in 16-bit and 32-bit projects when the standard color picker uses 8-bit values?

Instead of using the standard PF_ColorDef color picker which stores A_u_char values, sample the color picker as an A_FpLong directly from the parameter stream during render, then convert the floating-point value to your target 16-bit or 32-bit format.

```cpp
// Sample color picker as A_FpLong and convert to 16bit value
A_FpLong colorValue;
// Read colorValue from stream as A_FpLong
// Convert to 16bit representation
```

*Tags: `params`, `ui`, `render-loop`, `aegp`*

---

## Why is a newly added layer not visible in the composition, showing the source layer instead on the current frame?

Adding a layer during a PF_Cmd_RENDER call causes rendering issues because the layers involved in the render are listed and checked prior to the render call. When you add a new layer during rendering, you mess with pre-made lists of render items. The solution is to add the new layer to the composition during other calls, preferably during PF_Cmd_UPDATE_PARAMS_UI or during idle process, not during render.

```cpp
ERR(suites.LayerSuite7()->AEGP_AddLayer(res_item, cResCompH, &res_layer));
```

*Tags: `aegp`, `layer-checkout`, `render-loop`, `debugging`, `params`*

---

## Why can't I access parameter input layer pixel data in the DoClick function?

The incoming image data is only available during the render call, not in DoClick. However, there are several workarounds: (1) Store input/output pixels in sequence data during render for later access, though cached frames won't update; (2) Use AEGP_GetReceiptWorld() to fetch pixels of any video item at any time, though rendering composites can be slow; (3) In CS4 and earlier, use the UI drawing mechanism during the draw event; (4) Best practice: store click location and flags in sequence data during DoClick, then trigger a re-render to process pixels during the render call.

*Tags: `doclick`, `pixel`, `params`, `sequence-data`, `render-loop`, `aegp`*

---

## How do I use multiple sliders and pass their values to the Render function in After Effects plugins?

Use the refcon (a void pointer that points to a structure or class) to pass multiple slider values to the iterate() function. Instead of creating separate iterate() calls for each slider, consolidate your processing into a single iteration call by storing all slider values in a single refcon structure. This reduces overhead since each iteration function call has its own CPU cost. The refcon is flexible—you can add as many separate values as needed to it, and you're not required to treat the input pixel as input and output pixel as output in a specific way.

```cpp
suites.iterate8suite()->iterate(
  lines_so_far,
  total_lines,
  input_world,
  &output->extent_hint,
  refcon,  // Pass structure containing all slider values here
  ProcessFunction,
  output_world);
```

*Tags: `params`, `ui`, `render-loop`, `sdk`*

---

## How can I implement a custom pixel iteration function similar to PF_Iterate8Suite1::iterate?

To implement a custom iteration function, start by examining the 'shifter' sample which implements one of the iteration functions. For direct pixel data access without the iteration suite, consult the 'CCU' sample, particularly its render function. You can also use the iterate_generic() function to obtain threading services without additional functionality. For optimal performance, thread rows rather than individual pixels to reduce CPU overhead per pixel.

*Tags: `mfr`, `render-loop`, `threading`, `reference`, `pixel-iteration`*

---

## What sample plugins demonstrate pixel iteration and direct pixel data access in After Effects?

The 'shifter' and 'CCU' samples are recommended references. The 'shifter' sample implements iteration functions, while the 'CCU' sample demonstrates how to access pixel data directly without using the iteration suite, particularly in its render function implementation.

*Tags: `reference`, `mfr`, `open-source`, `render-loop`*

---

## How can I access the composited color of layers beneath the current layer in After Effects?

There is no direct API to access intermediate composition buffers during normal effect rendering, as After Effects does not render bottom-to-top and optimizes by skipping rendering of fully opaque regions. However, several workarounds exist: (1) Write an AEGP artisan-type plugin (see the 'arti' sample) which handles composition rendering and has access to intermediate buffers, though this is very difficult; (2) Create a duplicate composition with only needed layers and render it using AEGP_GetReceiptWorld(); (3) Apply your effect on an adjustment layer, which receives the composited buffer of underlying layers, then use checked-out layer params to access original sources; (4) Use sampleImage() expressions on hidden parameters to sample pixel data, though this is slow and limited; (5) Ask the user to provide the background as a layer parameter, a common practice in effects that need background information.

*Tags: `aegp`, `render-loop`, `layer-checkout`, `sdk`, `params`*

---

## What is the AEGP artisan sample plugin and how does it relate to composition rendering?

The 'arti' sample is an AEGP (After Effects General Plugin) of type 'artisan' that demonstrates how to write a custom composition renderer. Artisan plugins replace After Effects' default renderer (like 'advanced3D') and have direct access to intermediate composition rendering results. This makes them capable of accessing composited buffers at different rendering stages, unlike standard effect plugins which only see the final result. This approach is recommended for advanced scenarios requiring intermediate buffer access, though it is noted as being very difficult to implement correctly.

*Tags: `aegp`, `reference`, `render-loop`, `open-source`*

---

## How can I force a render on a custom effect from AEGP without adding the action to the undo/redo stack?

Create an undo group with a NULL instead of a name. Anything that happens in that group will NOT be added to the undo stack. Alternatively, you can use AEGP_SetStreamValue() to change an invisible parameter's value, which will trigger a render without affecting the result, but this approach will add the action to the undo/redo stack unless wrapped in a NULL undo group.

*Tags: `aegp`, `undo`, `render-loop`, `params`*

---

## What is the correct way to trigger a force render on a custom effect from AEGP?

You can use AEGP_SetStreamValue() to modify an invisible parameter value, which will trigger a render without affecting the effect output. Note that setting PF_OutFlag_Force_Rerender in PF_CMD_COMPLETELY_GENERAL does not work for triggering renders from AEGP.

*Tags: `aegp`, `params`, `render-loop`*

---

## What is the difference between using layer parameters for effects versus using AEGP for multi-layer access?

For effect-type plugins, use a layer parameter with PF_CHECKOUT_PARAM to access multiple layers selected by the user. For AEGP (After Effects General Plugin) development, the approach is more complex and requires using AEGP_RenderAndCheckoutFrame, which involves finding the project item you want to reference. The layer parameter approach is simpler for effects and allows users to dynamically select which layers to access.

*Tags: `aegp`, `params`, `layer-checkout`, `render-loop`*

---

## Can I use sequence_data to store a circular buffer for frame caching in an After Effects temporal filter effect?

Yes, you can use sequence_data to store a circular buffer. Allocate memory using PF_NEW_HANDLE during PF_Cmd_SEQUENCE_SETUP (though you can allocate at any time). Check if in_data->sequence_data == NULL to determine if memory has been allocated. Be aware of important considerations: (1) You cannot reliably know if cached frames are still valid after user edits, (2) Frames may be acquired out of order if the user scrubs randomly—track timestamps to handle this, (3) You won't know when scrubbing ends and sequential rendering begins, (4) If you set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING, the entire buffer is saved in the project file. Modern After Effects caches frames internally, so re-requesting past frames often retrieves cached data rather than forcing re-renders. The sequence_data lifecycle includes: PF_Cmd_SEQUENCE_SETUP (new instance), PF_Cmd_SEQUENCE_RESETUP (project load/paste/save), PF_Cmd_SEQUENCE_FLATTEN (save/copy preparation), and PF_Cmd_SEQUENCE_SETDOWN (cleanup).

```cpp
if (in_data->sequence_data == NULL) {
  // Allocate memory
  in_data->sequence_data = PF_NEW_HANDLE(buffer_size);
} else {
  // Memory already allocated, reuse it
}
```

*Tags: `sequence-data`, `memory`, `caching`, `aegp`, `render-loop`*

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
