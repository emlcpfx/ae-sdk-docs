# Q&A: layer-checkout

**135 entries** tagged with `layer-checkout`.

---

## What is the issue when trying to checkout multiple frames of a layer into an array during smart render?

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

*Tags: `layer-checkout`, `smart-render`, `sequence-data`, `memory`*

---

## Is it illegal to checkout a parameter multiple times?

You cannot checkout parameters during sequence setup, resetup (when AE is loading the project), or sequence setdown. The world checkout can sometimes be empty in these contexts. Additionally, you must handle checkout errors and check in the parameter immediately after using it in loops.

*Tags: `params`, `layer-checkout`, `aegp`, `debugging`*

---

## When should parameter check-in be performed relative to parameter usage?

The check-in must be done immediately after getting the parameter value, especially in loop examples where the parameter is accessed multiple times.

*Tags: `params`, `layer-checkout`, `aegp`*

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

## How can you set a layer default parameter value?

Layer defaults do not have a direct 'value' property. You can try using PF_LayerDefault_NONE or use the AEGP suite to modify layer defaults.

```cpp
param.u.ld = PF_LayerDefault_NONE
```

*Tags: `params`, `aegp`, `layer-checkout`*

---

## Where can I find code to get all active camera properties?

The Resizer example in the SDK contains code for accessing camera properties and interfacing with camera layers.

*Tags: `aegp`, `camera`, `sdk`, `layer-checkout`*

---

## How can I properly detect if a layer can be checked out in the userChangedParam callback without getting infinitely locked during effect duplication or closing?

Instead of attempting layer checkout in userChangedParam during shutdown or duplication, check the validity of layer data by examining width/height and Lrect values. Invalid layers will have random, oversized (>100000), or negative values. However, a more reliable approach is to avoid layer checkout during these callback scenarios, as the layer data may be unavailable or invalid during effect duplication and shutdown sequences.

*Tags: `layer-checkout`, `params`, `threading`*

---

## How do you get a Layer Handle from a layer parameter (PF_ADD_LAYER) to access its properties and stream?

A layer parameter only returns a World (pixel data) and not the actual layer definition or handle. It is not possible to retrieve the Layer Handle from a PF_ADD_LAYER parameter to access the layer's properties and stream. You will need to use an alternative approach.

*Tags: `params`, `layer-checkout`, `aegp`*

---

## How do you get a layer parameter value and retrieve the actual layer object from it in AEGP?

Use AEGP_GetNewEffectForEffect to get the effect handle, then AEGP_GetNewEffectStreamByIndex to get the stream for the parameter. Call AEGP_GetNewStreamValue to get the layer ID value, then use AEGP_GetLayerFromLayerID with the parent composition handle to get the target layer handle. Remember to dispose of the effect handle and stream handles when done.

```cpp
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, param_index, &currStreamH));
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &currLayerH));
ERR(suites.LayerSuite5()->AEGP_GetLayerParentComp(currLayerH, &compPH));
if (meH && currStreamH && currLayerH && compPH)
{
    ERR(suites.StreamSuite3()->AEGP_GetNewStreamValue(globP->my_id,currStreamH,AEGP_LTimeMode_LayerTime,&currTime,TRUE, &val));
    if (!err)
    {
        AEGP_LayerIDVal layerId = val.val.layer_id;
        ERR(suites.LayerSuite8()->AEGP_GetLayerFromLayerID(compPH, layerId, &targetLayerH));
    }
}
```

*Tags: `aegp`, `params`, `layer-checkout`*

---

## What is the difference between checking out a layer parameter with PF versus using AEGP streams?

When you checkout a layer parameter using the PF interface, you get a PF_Effect_world. When you use AEGP streams, you get the layer index instead.

*Tags: `aegp`, `params`, `layer-checkout`*

---

## How can a plugin render frame N that depends on custom data from previous frames without checking out all 100 previous frames in memory during pre-render?

This is a known challenge with recursive operations in After Effects plugins. The issue is that smart render checks needed frames during pre-render, which forces checking out all intermediate frames. The checkout param solution doesn't account for previous effects. AEGP_CacheAndCheckoutFrame doesn't trigger the pre_render/smart render thread, so it doesn't solve the problem. A potential approach is to use layer checkout parameters during compute cache operations, but this requires careful design to avoid excessive memory usage and to ensure previous effects are properly evaluated.

*Tags: `smart-render`, `compute-cache`, `layer-checkout`, `memory`, `aegp`*

---

## How can frame N access custom data computed from the previous frame, accounting for previous effects in the chain?

The user explored using checkout Param during compute cache, but noted this approach doesn't account for previous effects. The solution involves retrieving data computed from the previous frame (same size as a frame) through the effect chain.

*Tags: `compute-cache`, `layer-checkout`, `render-loop`, `sequence-data`*

---

## Why is layer data NULL after checkout_layer_pixels when a layer is selected in the layer selection parameter?

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

*Tags: `layer-checkout`, `params`, `debugging`, `smart-render`*

---

## Why does checkout_layer_pixels return NULL data for a layer parameter after checkout_layer succeeds?

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

*Tags: `layer-checkout`, `params`, `smart-render`, `gpu`, `aegp`*

---

## How can I check out the current layer (INPUT) with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT on the INPUT layer at different times, it returns the effectworld before all effects are applied, even those that precede the current effect in the chain. To get the current layer at different times with all effects applied, you need to use a different approach than simple PF_CHECKOUT. The Echo effect demonstrates that this is possible, likely through using layer composition or render callbacks that account for the full effect stack.

*Tags: `layer-checkout`, `render-loop`, `smartfx`, `params`*

---

## How can AEGPSetLayerTransferMode be used to set a layer as a trackmatte in After Effects versions before 23?

AEGPSetLayerTransferMode has limitations in older versions - it only works if the layer is already a trackmatte, and cannot set a layer to become a trackmatte if it isn't already one. The user reported this issue when trying to set a layer's transfer mode to ALPHA trackmatte using LayerSuite7. After Effects 23 and newer have a new method available that provides better backwards compatibility, though the specific method name was not detailed in the conversation.

```cpp
AEGP_LayerTransferMode mode = {};
mode.mode = PF_Xfer_IN_FRONT;
mode.track_matte = AEGP_TrackMatte_ALPHA;
mode.flags = 0;
ERR(suites.LayerSuite7()->AEGP_SetLayerTransferMode(layerH, &mode));
```

*Tags: `aegp`, `layer-checkout`*

---

## What is AEGP_RenderAndCheckoutLayerFrame_Async and can it be called from smart_render command?

AEGP_RenderAndCheckoutLayerFrame_Async is an asynchronous layer rendering and checkout function that can work on any thread, making it potentially usable from smartrender command, though it is known to be buggy.

*Tags: `smartrender`, `aegp`, `layer-checkout`, `threading`*

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

## How do you check out a frame of the current layer after prior effects are applied instead of before them?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects. However, this approach may not work in Premiere Pro. An alternative approach using Cmd_RENDER may be needed for cross-application compatibility.

*Tags: `layer-checkout`, `smartfx`, `premiere`, `render-loop`*

---

## How do you check out a frame of the current layer after prior effects are applied, rather than before them?

PF_CHECKOUT_PARAM gets the current frame before all prior effects. To get the frame after prior effects (like after Exposure but before the current effect), you need to use a different checkout method. The conversation indicates this is possible but the specific method was not detailed in this exchange.

*Tags: `layer-checkout`, `aegp`, `params`*

---

## How do you ensure that an effect uses unaltered input pixels rather than the plugin's previously rendered output when a parameter changes?

After Effects automatically provides the correct input layer. In PF_Cmd_SMART_RENDER, use extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world) to get the right input. In PF_Cmd_RENDER, use params[0]->u.ld to access the correct input layer. You don't need to do anything special; AE handles giving you the unaltered input, not your plugin's previous output.

```cpp
// PF_Cmd_SMART_RENDER
extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world);

// PF_Cmd_RENDER
params[0]->u.ld // correct input layer
```

*Tags: `smart-render`, `layer-checkout`, `params`*

---

## How can you determine the bit depth of a PF_LayerDef without using the world suite?

You can check the world_flags field of the PF_LayerDef. If PF_WorldFlag_DEEP is set, it's 16-bit. If PF_WorldFlag_RESERVED1 is set, it's 32-bit. Otherwise, it's 8-bit. This method has been tested and appears to work reliably in practice, though theoretically rowbytes padding could affect accuracy in edge cases.

```cpp
int get_bitdepth(const PF_LayerDef& layer) {
    if (layer.world_flags & PF_WorldFlag_DEEP) {
        return 16;
    }
    if (layer.world_flags & PF_WorldFlag_RESERVED1) {
        return 32;
    } else {
        return 8;
    }
}
```

*Tags: `layer-checkout`, `params`*

---

## Is it safe to rely on PF_WorldFlag_RESERVED1 to detect 32-bit depth in production plugins?

Yes, it appears safe to trust this method in actual plugins. According to the original code author, this approach was derived from SDK examples, the AE SDK mailing list, or reverse-engineering work done during actual plugin development (specifically a 3D renderer plugin requiring f32 support). The method has proven reliable in production use.

*Tags: `layer-checkout`, `params`, `debugging`*

---

## What is the purpose of the rowbytes parameter in layer data?

Rowbytes exists as an optimization so that layers can be cropped without having to allocate extra memory. You can crop a layer by simply updating width and height and making the data pointer point to the new top-left pixel, leaving rowbytes untouched while keeping a reference to the old data for proper deallocation. AE does this often, such as when dragging a layer partially off the composition.

*Tags: `memory`, `output-rect`, `layer-checkout`*

---

## What is the purpose of rowbytes in After Effects layer data?

rowbytes exists as an optimization so that layers can be cropped without allocating extra memory. You can crop a layer by simply updating width and height and making the data pointer reference the new top-left pixel, leaving rowbytes untouched while keeping a reference to the old data for proper cleanup. After Effects does this often, such as when dragging a layer partially off the composition.

*Tags: `memory`, `layer-checkout`, `output-rect`, `caching`*

---

## Does rowbytes change when a layer is moved partially off-comp and then cache is purged?

When you load a layer and move it partially off-comp, rowbytes will be larger than the minimum required. However, after purging cache, rowbytes will likely return to 4*width (or 4*4*width for 32-bit data), indicating the cache optimization is recalculated.

*Tags: `memory`, `caching`, `compute-cache`, `layer-checkout`*

---

## Is there a method for forcing masks on a layer to be applied after the plugin processes?

No answer was provided in the conversation.

*Tags: `smartfx`, `render-loop`, `layer-checkout`*

---

## Why does the precomp result show only a masked region as the result rect with everything else black?

When the precomp is created, the result rect is determined by the masked layer inside it. Since the precomp is full comp size but contains only a masked layer, only the area around the mask is included in the result rect, leaving everything else black.

*Tags: `mfr`, `output-rect`, `layer-checkout`*

---

## Is it possible to create a multi-layer 2D global illumination plugin where a renderer effect discovers and uses material properties from dummy effects on other layers?

This approach is theoretically possible with the After Effects SDK. You can use the AEGP suite to enumerate layers in a composition and checkout their properties. However, the key challenge is properly communicating dependencies to AE's internal dependency tracking system. You need to explicitly declare all layer and effect dependencies your renderer effect relies on, so that changes to materials or other layers trigger re-renders. This ensures the disk cache is used correctly—cached results are only used when all dependencies remain unchanged. The AEGP layer enumeration and property checkout APIs support this pattern, though it requires careful dependency management to avoid either missing updates or unnecessary re-renders.

*Tags: `aegp`, `layer-checkout`, `caching`, `params`, `compute-cache`*

---

## How can a renderer effect communicate its dependencies on other layers and effects to AE's caching system?

Dependencies must be explicitly registered through the AEGP API when your renderer effect enumerates and checks out layers. When you call layer checkout functions to read pixel data, masks, shapes, and transforms from dependent layers, and query properties of effects on those layers, AE tracks these accesses as dependencies. To ensure proper cache invalidation, you should checkout all relevant layer properties at render time—if any of those checked-out properties change, AE will invalidate the cached result and trigger a re-render. This allows the disk cache to function correctly by only using cached data when all dependencies are identical.

*Tags: `aegp`, `caching`, `layer-checkout`, `compute-cache`, `dependency`*

---

## Is it possible to create a multi-layer effect that discovers material properties from per-layer dummy effects and performs global illumination rendering while properly communicating dependencies to After Effects' caching system?

Tobias Fleischer shared that he attempted a plugin-per-layer approach years ago but abandoned it due to render order troubles. Instead, he recommended using a single plugin with multiple layer parameters that pull in pixels from different layers, apply material/lighting options, and composite everything within that single plugin. This approach worked well despite AE's rigid parameter interface being somewhat clunky. The viability of the frankensteined multi-layer approach with proper dependency tracking depends on whether the SDK allows explicit communication of implicit dependencies to AE's caching system, though existing solutions tend to consolidate the logic into a single effect rather than relying on scattered per-layer effects.

*Tags: `params`, `layer-checkout`, `caching`, `smart-render`, `output-rect`*

---

## Is it possible to make a multi-layer 2D global illumination effect where a renderer effect discovers material metadata effects on other layers and communicates these dependencies to AE's caching system?

It is theoretically possible using AEGP_RenderAndCheckoutLayerFrame to enumerate layers and pass their GUIDs via GuidMixInPtr() to register dependencies. However, a more practical approach is to use hidden layer parameters (up to 999) with a script button that assigns layers from the composition to these parameters, allowing checkout_layer() to work efficiently with SmartPreRender. You can also mix in layer order information via GuidMixInPtr() to handle dynamic layer changes without manual updates.

*Tags: `aegp`, `smart-render`, `layer-checkout`, `caching`, `compute-cache`*

---

## Can checkout_layer() be used on arbitrary composition layers during PF_Cmd_SMART_PRE_RENDER that aren't declared as effect parameters?

According to the SDK documentation, you can only grab layer parameters that are explicitly declared as layer parameters on the effect. To work with arbitrary layers, use hidden layer parameters that a script can populate, or use AEGP_RenderAndCheckoutLayerFrame which works with any layer IDs.

*Tags: `smart-render`, `layer-checkout`, `aegp`*

---

## Does AEGP_RenderAndCheckoutLayerFrame perform the expensive rendering immediately or defer it until AEGP_GetReceiptWorld is called?

AEGP_RenderAndCheckoutLayerFrame performs the expensive render operation immediately and returns an AEGP_FrameReceiptH. You can then call AEGP_GetReceiptGuid() on the receipt to get a GUID that can be mixed into GuidMixInPtr() for dependency tracking. There is both a synchronous and asynchronous version available; the async version could potentially be called during SmartPreRender and awaited during the render call/thread.

*Tags: `aegp`, `smart-render`, `layer-checkout`, `threading`*

---

## How should you handle caching when scraping data from dummy effects on other layers?

Call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects, and AE will cache everything fine.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`*

---

## How should material controls be organized when using child plugins on layers?

It is more convenient to have each set of material controls directly on the layer it is affecting rather than having indirection through the main effect's parameter list. This allows users to access layer-specific properties without having to search through a potentially long list of parameters in the main effect.

*Tags: `params`, `ui`, `layer-checkout`*

---

## How can dummy effects be used to specify layer properties without manual renaming?

The main effect loops over each layer's effects and looks for specifically-named instances of slider/checkbox controls. A more polished dummy effect can be created for specifying layer properties so users don't have to manually add slider controls and rename them, though the principle remains the same: these dummy effects wouldn't do anything themselves, just serve as property containers that the main effect reads from.

*Tags: `params`, `ui`, `layer-checkout`*

---

## Why does a plugin on a Premiere adjustment layer only receive the first keyframe value regardless of current time when checking out parameters?

This is a known issue in Premiere where plugins applied to adjustment layers with multiple keyframes on parameters receive only the value at the first keyframe, regardless of the current playhead time during parameter checkout. The issue affects all parameter types tested including sliders, angles, checkboxes, and 2D points. This behavior does not occur in After Effects, suggesting it is specific to Premiere's adjustment layer implementation.

*Tags: `premiere`, `params`, `layer-checkout`, `debugging`*

---

## What is the correct approach for checking out multiple frames of a layer parameter during Smart Render for frame-based caching?

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

*Tags: `layer-checkout`, `caching`, `sequence-data`, `smart-render`, `memory`*

---

## Is it illegal to checkout a parameter multiple times in After Effects plugins?

Checkout of parameters can be problematic in certain lifecycle stages. You cannot perform parameter checkout during sequence setup, resetup (when AE is loading the project), or sequence setdown. It's important to handle checkout errors properly and ensure that checkin is called immediately after using the parameter value, rather than holding the checkout open.

*Tags: `params`, `layer-checkout`, `debugging`, `aegp`*

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

## How do you set a layer default parameter value in After Effects plugins?

Layer definitions do not have a direct 'value' property. Instead, you can use PF_LayerDefault_NONE or similar PF_LayerDefaults constants to set layer default parameters. For more complex parameter manipulation, using the AEGP API is recommended as an alternative approach.

```cpp
param.u.ld = PF_LayerDefault_NONE;
```

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How can I safely detect if a layer is available for checkout in a userChangedParam callback?

A workaround is to validate layer data by checking width/height and Lrect values—if they are random, over 100,000, or negative, the layer is not properly available for checkout. Note that Param[0].u.ld.data will be empty if checkout hasn't occurred in that thread. However, this is not a fully reliable solution, and the proper approach may require additional checks during effect duplication and shutdown phases.

*Tags: `layer-checkout`, `params`, `arb-data`, `shutdown`, `threading`*

---

## What AEGP method should be used to render and checkout a layer frame?

AEGP_RenderAndCheckoutLayerFrame is the AEGP method specifically implemented for rendering and checking out layer frames.

*Tags: `aegp`, `layer-checkout`, `render-loop`*

---

## How do you get a Layer Handle from a layer parameter (PF_ADD_LAYER) in After Effects plugins?

Based on investigation, layer parameters (PF_ADD_LAYER) only provide access to the rendered World (pixels) of the selected layer, not the actual layer definition or handle. This means you cannot directly retrieve layer properties or streams through a layer parameter. The available suite functions do not support this capability, so alternative approaches must be used instead of relying on the layer parameter to access layer definitions.

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How do you retrieve a layer parameter value and get the actual layer handle from an effect in After Effects?

Use AEGP_GetNewEffectForEffect to get the effect handle, then AEGP_GetNewEffectStreamByIndex to get the stream for the parameter. Call AEGP_GetNewStreamValue to get the layer ID value, then use AEGP_GetLayerFromLayerID with the parent composition handle to get the actual layer handle. Remember to dispose of the effect handle and stream handles after use.

```cpp
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, param_index, &currStreamH));
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &currLayerH));
ERR(suites.LayerSuite5()->AEGP_GetLayerParentComp(currLayerH, &compPH));
if (meH && currStreamH && currLayerH && compPH) {
    ERR(suites.StreamSuite3()->AEGP_GetNewStreamValue(globP->my_id, currStreamH, AEGP_LTimeMode_LayerTime, &currTime, TRUE, &val));
    if (!err) {
        AEGP_LayerIDVal layerId = val.val.layer_id;
        ERR(suites.LayerSuite8()->AEGP_GetLayerFromLayerID(compPH, layerId, &targetLayerH));
    }
}
```

*Tags: `aegp`, `params`, `layer-checkout`*

---

## What is the difference between checking out a layer parameter with PF_Effect_world versus using AEGP streams?

When you checkout a layer parameter using the PF interface, you get a PF_Effect_world directly. When using AEGP streams, you get an index value instead, which you then need to resolve to an actual layer handle using the AEGP layer suite functions.

*Tags: `aegp`, `params`, `layer-checkout`*

---

## How can a plugin implement recursive operations where frame N depends on computed data from previous frames without checking out all intermediate frames during pre-render?

This is a known challenge when combining smart render with compute caches that require sequential frame dependencies. The user attempted several approaches: (1) using a compute cache with smart render, but this requires checking out all previous frames during pre-render when rendering frame 100, which is memory-inefficient; (2) using checkout params during compute cache, which doesn't account for previous effects; (3) using AEGP_CacheAndCheckoutFrame, which doesn't trigger pre_render/smart render callbacks on checked-out frames. A solution may involve manually managing frame dependencies outside the smart render system or using layer checkout strategically to load only necessary intermediate results rather than relying on automatic frame dependency detection.

*Tags: `smart-render`, `compute-cache`, `layer-checkout`, `memory`, `aegp`, `sequence-data`*

---

## How do you access computed data from previous frames in a compute cache when using smart render?

When accessing computed data from frame N-1 in a compute cache, if that frame hasn't been computed yet, you need access to the layer at the previous frame. This creates a cascading dependency—if N-2 is also not calculated, you need the layer at N-2 as well. With smart render, pre-loading all needed layers during pre-render can be expensive, especially if the user sets the timeline cursor at an arbitrary time like 10 seconds. The solution is to use classical checkout_param during compute cache, though this loads layers without their effects applied.

*Tags: `compute-cache`, `smart-render`, `layer-checkout`, `render-loop`*

---

## Why is layer pixel data NULL after checkout_layer_pixels for a selected layer parameter?

When checkout_layer_pixels returns a non-NULL world pointer but the data field within it is NULL, this typically indicates that the layer parameter has no layer selected or the layer is disabled/invisible. Ensure that in your layer parameter definition you're using an appropriate default (like PF_LayerDefault_NONE or a specific layer) and verify in PreRender that the layer checkout succeeded. The issue may also resolve itself after rebuilding or reloading the plugin. Check that the layer parameter is actually populated with a valid layer selection before SmartRender is called.

```cpp
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `layer-checkout`, `smart-render`, `params`, `debugging`*

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

## How can I get text layer path vertices in true layer space that match expression path.points() behavior, especially when the layer is parented?

PF_PathDataSuite does not return vertices in true layer space as documented in the SDK. Instead, it returns vertices in a hybrid space that is affected by 2D transforms (xy-translation, shear from parented transforms, Z Rotation) but not by z-translation, Orientation, or X/Y Rotation properties. To convert PF_PathDataSuite output to true layer space (matching what path.points() returns in expressions), you need to reverse the 2D transforms while preserving the unaffected 3D properties. Use AEGP_GetLayerToWorldXform() to convert to world space after obtaining layer-space vertices. The discrepancy exists because PF_PathDataSuite appears to return vertices in a space between composition and layer space, unlike expressions which correctly return layer-space coordinates independent of layer transform properties.

*Tags: `aegp`, `layer-checkout`, `params`, `reference`, `cross-platform`*

---

## How can I check out the current layer with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT to check out the INPUT layer at different times, it returns the effectworld before all effects are applied, including effects applied before the current effect. However, this is possible to achieve with all effects applied, as demonstrated by the Echo effect. The solution involves understanding the layer checkout mechanism and potentially using different checkout strategies or timing parameters to access the fully composited layer rather than the raw input.

*Tags: `layer-checkout`, `aegp`, `mfr`, `render-loop`*

---

## Why does checking out a layer parameter in PreRender still return the layer before all effects are applied?

When using PF_CHECKOUT_PARAM to check out a layer (INPUT) in the PreRender callback, the layer is returned in its pre-effect state rather than with effects applied. This is expected behavior in the AE plugin architecture—the checkout at PreRender stage occurs before the effect chain is fully processed.

*Tags: `aegp`, `layer-checkout`, `render-loop`, `debugging`*

---

## What is the difference between checkout_layer_pixels and PF_Checkout?

checkout_layer_pixels should be used instead of PF_Checkout for layer pixel operations in After Effects plugins.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How do you handle dependencies on previous frames in smart render when you need data from multiple prior frames?

In smart render, if your compute needs the result of a previous frame (e.g., frame N-1) but also requires frame N-2, you cannot use the smart checkout layer. Instead, you must use the classical checkout/checkin parameter approach to access the frames you need.

*Tags: `smart-render`, `layer-checkout`, `render-loop`, `sequence-data`*

---

## How can you set a layer to be a trackmatte using AEGPSetLayerTransferMode with backwards compatibility before AE 23?

James Whiffin reported that AEGPSetLayerTransferMode only works to set trackmatte mode if the layer is already a trackmatte, but won't convert a non-trackmatte layer to one. He notes that AE 23+ has a new method for this purpose, but for backwards compatibility with earlier versions, the standard AEGP_SetLayerTransferMode approach has this limitation. The code structure involves creating an AEGP_LayerTransferMode struct with the desired transfer mode and track matte type, then calling the LayerSuite7 function.

```cpp
AEGP_LayerTransferMode mode = {};
mode.mode = PF_Xfer_IN_FRONT;
mode.track_matte = AEGP_TrackMatte_ALPHA;
mode.flags = 0;
ERR(suites.LayerSuite7()->AEGP_SetLayerTransferMode(layerH, &mode));
```

*Tags: `aegp`, `layer-checkout`, `backwards-compatible`*

---

## How can you access the previous frame's data when processing frame n in an After Effects effect plugin?

You can check out frame n-1 at render time to access the input frame from the previous frame. For more complex cases where you need the output from your plugin's previous frame, you should use sequence data, which is specifically designed for simulation plugins that require frame-by-frame dependencies.

*Tags: `sequence-data`, `render-loop`, `layer-checkout`, `aegp`*

---

## Do I need to use PF_NewWorld to modify the output PF_LayerDef in the Render function, or can I directly adjust its data?

You should not modify the layer settings (width, height, rowbytes) directly as they are defined by the host app. Instead of directly assigning mat.data to layerDef->data, you should copy the data from your matrix into the layerDef->data buffer using a loop while respecting layerDef->rowbytes. The proper approach is demonstrated in the skeleton plugin, which uses the Iterate Suite to write into output->data rather than allocating output itself.

```cpp
static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        for (int x = 0; x < layer->width; ++x) {
            cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
            PF_Pixel& aePixel = *sampleIntegral32(*layer, x, y);
            aePixel.alpha = pixel[3];
            aePixel.red = pixel[2];
            aePixel.green = pixel[1];
            aePixel.blue = pixel[0];
        }
    }
}
```

*Tags: `aegp`, `memory`, `layer-checkout`, `output-rect`*

---

## How can you efficiently convert an After Effects layer to OpenCV Mat format while handling pixel format conversion and ROI?

When converting a PF_LayerDef to cv::Mat, use PF_GET_PIXEL_DATA8 to get the pixel data pointer, then create a Mat with the proper stride parameter (rowbytes). Use cv::mixChannels with a fromTo mapping array to handle channel reordering (e.g., ARGB to BGRA: fromTo[] = {0, 3, 1, 2, 2, 1, 3, 0}). The Mat constructor accepts a step parameter for row stride: cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes). This allows proper handling of extent_hint for ROI-based processing.

```cpp
static cv::Mat ConvertLayerToMat(PF_LayerDef* layerDef, PF_InData* in_data) {
    PF_Pixel8* pixelData = nullptr;
    PF_GET_PIXEL_DATA8(layerDef, NULL, &pixelData);
    int width = layerDef->width;
    int height = layerDef->height;
    cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
    cv::Rect roi(
        in_data->extent_hint.left - in_data->output_origin_x,
        in_data->extent_hint.top - in_data->output_origin_y,
        width, height
    );
    if (roi.x < 0 || roi.y < 0 || roi.x + roi.width > width || roi.y + roi.height > height) {
        throw std::runtime_error("ROI goes outside the image dimensions.");
    }
    cv::Mat argbRoi = argb(roi);
    cv::Mat bgra(height, width, CV_8UC4);
    int fromTo[] = {0, 3, 1, 2, 2, 1, 3, 0};
    cv::mixChannels(&argbRoi, 1, &bgra, 1, fromTo, 4);
    return bgra;
}
```

*Tags: `opencv`, `pixel-format`, `memory`, `layer-checkout`, `output-rect`*

---

## What is the OpenCV Mat step parameter and how does it help with custom pixel data?

OpenCV's Mat class supports a step parameter (also called rowbytes) that allows you to create a Mat from user-allocated pixel data while specifying the stride/row pitch. This is documented in the OpenCV Mat constructor. When working with After Effects layer data, pass layerDef->rowbytes as the step parameter to correctly handle the memory layout of the pixel buffer, enabling efficient operations like mixChannels and memcpy.

```cpp
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Tags: `opencv`, `memory`, `caching`, `layer-checkout`*

---

## How can I detect if an input frame has actually changed in an After Effects plugin?

You can use PF_GetCurrentState to tell if an input has changed. Additionally, if an input frame changes you should certainly get a re-render. If you need to force another prerender, you can mix it into the guid mix in ptr.

*Tags: `params`, `render-loop`, `smartfx`, `layer-checkout`*

---

## How do you check out a frame of the current layer after prior effects are applied in After Effects?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects have been applied. This allows you to access the layer state after upstream effects like Exposure have been rendered, but before the current effect processes it. However, this approach may not work in Premiere Pro, requiring alternative solutions for cross-application compatibility.

*Tags: `layer-checkout`, `smartfx`, `render-loop`, `premiere`, `aegp`*

---

## How can you access layer pixels in Premiere Pro during parameter changes if checkout_layer_pixels is not available in Cmd_RENDER?

checkout_layer_pixels is available in PF_SmartRenderCallbacks for After Effects, but for Premiere Pro compatibility, you need to access layer pixels during the user changed parameter callback instead of the render command. The approach involves handling pixel checkout in the parameter change event rather than the standard render path.

*Tags: `premiere`, `aegp`, `params`, `layer-checkout`, `cross-platform`*

---

## How do you check out a frame of the current layer after prior effects have been applied in After Effects plugins?

To check out a frame after prior effects are applied (rather than before), you need to use a different approach than PF_CHECKOUT_PARAM, which gets the frame before all prior effects. The conversation indicates this is possible but the specific method was not detailed in the exchange. The questioner was trying to retrieve a frame after the Exposure effect but before the AI Color Match effect in the effect stack.

*Tags: `aegp`, `layer-checkout`, `params`, `render-loop`*

---

## Can hiding layers by default improve performance when dealing with many layers in After Effects?

Hiding layers by default can be an effective optimization strategy to reduce the performance load on the UI and API calls, though the specific performance gains depend on the implementation and number of layers being processed.

*Tags: `ui`, `performance`, `optimization`, `layer-checkout`*

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

## Is there an open-source reference for After Effects plugin development and layer handling?

The virtualritz/after-effects GitHub repository (https://github.com/virtualritz/after-effects) contains useful reference code for plugin development, including examples of working with PF_LayerDef structures, bit depth detection, and other AEGP-related functionality.

*Tags: `open-source`, `reference`, `aegp`, `layer-checkout`*

---

## What is the purpose of the rowbytes field in layer data structures?

The rowbytes field exists as an optimization to allow layers to be cropped without allocating extra memory. A layer can be cropped by simply updating width and height and making the data pointer reference the new top-left pixel, while leaving rowbytes untouched. After Effects uses this technique frequently, for example when dragging a layer partially off the composition.

*Tags: `memory`, `layer-checkout`, `optimization`*

---

## How can you apply masks to GPU-rendered output in After Effects when the GPU API only provides masked/cropped buffers?

Eric encountered this limitation while developing the Wrap It 3D camera projection plugin. After Effects' GPU rendering API (checkout_layer_pixels) only returns post-mask-applied, cropped buffers with no GPU equivalent of PF_CHECKOUT_PARAM to access the raw unmasked source. Attempted workarounds included using PF_OutFlag2_REVEALS_ZERO_ALPHA with expanded PreRender requests, but this didn't provide the full unmasked layer to GPU. A manual CPU→GPU upload of the unmasked source defeats GPU acceleration benefits. The core limitation is that AE's native GPU world has no mechanism to provide raw unmasked source data on GPU. As a workaround, users may need to use track mattes in GPU mode, or consider building a custom GPU context (OpenGL, Vulkan, or WebGPU) outside of After Effects' native GPU pipeline to maintain full control over the source data.

*Tags: `gpu`, `layer-checkout`, `masks`, `aegp`, `render-loop`*

---

## How can a plugin effect communicate implicit dependencies on other layers and effects to After Effects' caching system?

When building multi-layer effects that discover and depend on other layers' properties at runtime, you need to explicitly communicate these dependencies to AE's render dependency tracking system. This involves checking out the layers you depend on and notifying AE of those checkouts so the cache knows to invalidate when those layers change. The renderer effect should enumerate dependent layers, check out their pixels/masks/shapes/transforms, and use the appropriate AEGP calls to register these dependencies so AE's smart render and disk cache systems understand the effect needs to re-render when those layers are modified.

*Tags: `aegp`, `layer-checkout`, `caching`, `smart-render`, `compute-cache`, `memory`*

---

## Is it feasible to build a 2D global illumination plugin using material property metadata effects on individual layers with a separate renderer effect?

Yes, this architecture is theoretically viable with the AE SDK. The pattern involves creating dummy effects on individual layers that only expose material parameters (emissive, diffuse, glass properties, etc.) without rendering output, while a separate renderer effect on an adjustment layer or solid discovers and interprets these metadata effects at render time. The key challenge is properly communicating cross-layer dependencies to AE's caching system so changes to materials trigger re-renders of the expensive global illumination computation while still benefiting from disk caching when unrelated elements change.

*Tags: `aegp`, `params`, `layer-checkout`, `caching`, `smart-render`, `rendering`*

---

## What are the practical challenges and solutions when designing multi-layer effects in After Effects plugins?

According to Tobias Fleischer (reduxFX), a plugin-per-layer approach was abandoned in favor of a single plugin with multiple layer parameters due to render order troubles. The successful approach involved creating a single plugin that accepts 10 layer parameters to pull in pixels from multiple layers, applies lighting/material options, and composites everything together within that single plugin. While this worked well, AE's rigid parameter interface made it somewhat clunky to use. The key insight is that centralizing the multi-layer logic into a single plugin avoids render order dependency issues that arise from distributing logic across multiple per-layer plugins.

*Tags: `render-loop`, `params`, `ui`, `layer-checkout`, `smartfx`*

---

## Is it possible to create a multi-layer global illumination plugin that discovers material metadata effects on other layers and properly communicates dependencies to After Effects' caching system?

This is a complex architectural question posed by fad regarding whether a 'renderer' effect can enumerate other layers, discover per-layer material definition effects, check out layer pixels/masks/transforms, and have those dependencies properly tracked by AE's dependency tracking and disk cache system. While Tobias Fleischer's experience suggests multi-layer effects are viable, the specific mechanism for communicating hidden dependencies (non-explicit UI parameter dependencies) to AE's caching system to trigger rerenders when materials change remains an open design challenge requiring careful SDK integration and likely custom dependency management.

*Tags: `render-loop`, `caching`, `compute-cache`, `layer-checkout`, `smartfx`, `memory`*

---

## Can I use checkout_layer() during PF_Cmd_SMART_PRE_RENDER to access arbitrary composition layers not exposed as effect parameters?

According to maxon developer Alex Bizeau, you can only grab layer params directly during SMART_PRE_RENDER. However, you can use AEGP_RenderAndCheckoutLayerFrame with any layer IDs, and both sync and async versions exist. For multiple layers, a workaround is to use hidden layer parameters (up to 999) assigned via a button script that updates the comp, so checkout_layer() can access them. You may also handle dynamic layer order/removal by looping through the comp in SmartPreRender and mixing layer order into GuidMixInPtr() for dependency tracking.

*Tags: `smart-render`, `layer-checkout`, `aegp`, `params`, `caching`*

---

## How can I make a multi-layer effect that depends on material properties defined by effects on other layers while maintaining smart render efficiency?

The most robust approach is to use hidden layer parameters (you can have up to 999) and a button parameter that triggers a script to assign layers from the composition to those parameters one by one. Alternatively, loop through the comp in SmartPreRender and mix layer order into GuidMixInPtr() for dependency tracking. This allows the renderer effect to check out all layers efficiently while maintaining AE's caching and smart render system. You only need to click an 'Update Comp' button when you add layers; layer order changes or removals can be handled automatically by the SmartPreRender logic.

*Tags: `smart-render`, `layer-checkout`, `params`, `caching`, `compute-cache`*

---

## How can you efficiently cache data when scraping parameters from dummy effects on multiple layers?

When implementing a control scheme using layer parameters on the main effect with dummy effects on other layers, call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects. This ensures After Effects caches everything properly and maintains good performance even when reading all layers one by one.

*Tags: `aegp`, `params`, `caching`, `memory`, `layer-checkout`*

---

## What is a good approach for implementing multi-layer effect controls in After Effects plugins?

A viable user-friendly approach is to use layer parameters on the main effect combined with dummy effects on other layers, rather than requiring users to switch to artisan rendering mode. This integrates well with other effects and provides better usability while maintaining acceptable performance. Testing at quarter resolution can help evaluate performance characteristics.

*Tags: `aegp`, `params`, `ui`, `layer-checkout`, `render-loop`*

---

## How can you synchronize layer data changes in an After Effects plugin using AEGP?

According to Alex Bizeau from maxon, you need to periodically check the project timestamp for changes, then gather your layers and check children for potential updates. This approach is computationally expensive but is noted as the proper synchronization alternative available with AEGP.

*Tags: `aegp`, `layer-checkout`, `caching`, `memory`*

---

## How can I checkout masks and paths from a different layer using the After Effects SDK?

You can fetch any mask using AEGP_MaskSuite6, then read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape parameter, you need to traverse the layer's dynamic streams. However, be aware that mask and shape vertex data is delivered unmodified—masks won't include the "expand" parameter modification, and shapes won't show rounding if applied by the user. Additionally, some shapes are parametric (like star shapes) and have no shape parameter to read.

*Tags: `sdk`, `layer-checkout`, `aegp`, `masks`, `sequence-data`*

---

## How do you get the name of the selected layer in an After Effects plugin?

To get the name of the selected layer, first call AEGP_GetNewCollectionFromCompSelection() to get the current selection. Then traverse the collection using AEGP_GetCollectionNumItems() and AEGP_GetCollectionItemByIndex() to get individual collection items. Each item returns an AEGP_CollectionItemV2 structure. Check the 'type' field to identify if it's a layer. If it is a layer type, extract the AEGP_LayerH from the union field u.layer. Finally, pass this AEGP_LayerH handle to AEGP_GetLayerName() to retrieve the layer name.

```cpp
// Get selected layers
AEGP_CollectionH collectionH;
AEGP_GetNewCollectionFromCompSelection(NULL, &collectionH);

A_long num_items;
AEGP_GetCollectionNumItems(collectionH, &num_items);

for (A_long i = 0; i < num_items; i++) {
    AEGP_CollectionItemV2 item;
    AEGP_GetCollectionItemByIndex(collectionH, i, &item);
    
    if (item.type == AEGP_CollectionItemType_LAYER) {
        AEGP_LayerH layer_h = item.u.layer.layer_handle;
        A_UTF16Char layer_name[AEGP_MAX_LAYER_NAME_LEN];
        AEGP_GetLayerName(layer_h, layer_name);
    }
}
```

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How should text input be stored with a layer when using arbitrary parameters?

Arbitrary parameters can store the text string and are preferred over sequence data because they support undo/redo operations. The arb parameter itself can be hidden from the user if desired. You can implement a custom UI on the arb parameter that displays the current text value and is clickable, or alternatively use a separate button parameter to trigger the input dialog while the arb parameter stores the data invisibly.

*Tags: `params`, `arb-data`, `ui`, `layer-checkout`*

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

## How do I retrieve a layer's position after checking it out in an After Effects plugin?

To get a layer's position, use AEGP_GetNewLayerStream with the AEGP_LayerStream_POSITION parameter. The origin_x and origin_y fields from a checked-out layer only give the 0,0 location within the grabbed buffer, not the layer's actual position in the composition. If you need the layer's source dimensions instead, use AEGP_GetNewStreamValue to get the AEGP_LayerIDVal, then AEGP_GetLayerFromLayerID, AEGP_GetLayerSourceItem, and finally AEGP_GetItemDimensions.

```cpp
AEFX_CLR_STRUCT(checkout);
ERR(PF_CHECKOUT_PARAM(in_data,
  PARENT_LAYER_ID,
  in_data->current_time,
  in_data->time_step,
  in_data->time_scale,
  &checkout));
// Use AEGP_GetNewLayerStream with AEGP_LayerStream_POSITION
// origin_x and origin_y only give buffer-relative coordinates, not composition position
```

*Tags: `layer-checkout`, `aegp`, `params`, `reference`*

---

## How do you disable a layer's video switch from a C++ plugin effect?

Use AEGP_GetEffectLayer to get the layer handle from the effect reference, then use AEGP_SetLayerFlag with AEGP_LayerFlag_VIDEO_ACTIVE set to FALSE to disable the layer's video output. This is more efficient for render performance than setting opacity to 0.

```cpp
AEGP_LayerH layerH;
suites.AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
suites.AEGP_SetLayerFlag(layerH, AEGP_LayerFlag_VIDEO_ACTIVE, FALSE);
```

*Tags: `aegp`, `layer-checkout`, `performance`, `sdk`*

---

## How can I access and modify layer styles like inner shadow using the AE SDK?

Use AEGP_DynamicStreamSuite4 to access layer styles. You need to explore the layer's dynamic stream groups to find the specific style properties you want to modify, such as inner shadow values.

*Tags: `aegp`, `layer-checkout`, `sdk`, `reference`*

---

## How do you get the original foreground layer pixels when implementing a custom blend mode effect on an adjustment layer?

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

*Tags: `layer-checkout`, `aegp`, `blend-mode`, `pipl`, `params`*

---

## What is the difference between the old checkout mechanism and checkout_layer_pixels() when fetching layer pixels?

The old checkout mechanism (PF_CHECKOUT_PARAM) fetches the original layer pixels pre-masks and pre-effects, making it suitable for implementing custom blend modes. The newer checkout_layer_pixels() function fetches pixels post-masks and post-effects. For blend mode implementations where you need the original foreground pixels, use the old PF_CHECKOUT_PARAM mechanism instead.

*Tags: `layer-checkout`, `pipl`, `params`, `blend-mode`*

---

## How can I read layer transform data like position, anchor, and scale in an After Effects plugin?

To get a layer's numeric transform values (position, anchor, scale), use AEGP_GetNewLayerStream to get the parameter's stream and AEGP_GetNewStreamValue to retrieve its numeric value. However, if you need all of the layer's transformations in the composition including parenting or camera movement effects that affect the layer's position on screen, use AEGP_GetLayerToWorldXform mixed with the camera matrix. Note that you'll need to get AEGP_PluginID during GlobalSetup(), which is required for many AEGP methods.

*Tags: `aegp`, `params`, `layer-checkout`, `transform`*

---

## How do I get the layer handle from a selected layer in a layer parameter?

The layer parameter's u.ld field contains a layer ID. Use the AEGP_GetLayerFromLayerID function to convert that layer ID into a layer handle, which you can then use to access the layer's transforms and other properties.

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How do I access input from different layers and at different times in an After Effects plugin?

Inputs from different layers and at different times can be accessed using layer parameters. Every effect has a default layer param (index 0) used to checkout the input layer. To get images from other times, look up 'WIDE_TIME' in the After Effects SDK documentation. Both the effect's own layer and other layers are accessed via layer parameters.

*Tags: `sdk`, `params`, `layer-checkout`*

---

## How can I detect if a layer is a shape layer using the AEGP?

Use the function AEGP_GetLayerObjectType() and check if the result equals AEGP_ObjectType_VECTOR to determine if a layer is a shape layer.

```cpp
AEGP_GetLayerObjectType()
// Returns AEGP_ObjectType_VECTOR for shape layers
```

*Tags: `aegp`, `shape`, `layer-checkout`, `sdk`*

---

## What do the origin_x and origin_y fields in PF_EffectWorld represent when checking out a shape layer?

The origin_x and origin_y fields in the PF_EffectWorld structure represent the offset position of the checked-out shape within its buffer. When checking out a shape layer, the resulting buffer is the size of the shape itself (not the layer), and the origin fields indicate where that shape is positioned relative to the layer's coordinate space.

*Tags: `layer-checkout`, `shape`, `output-rect`, `sdk`*

---

## How can I copy a layer parameter choice from one PF_Param_LAYER to another programmatically?

To copy a layer parameter from one PF_Param_LAYER to another, get the layer id from the first parameter using AEGP_GetNewStreamValue(), then set that layer_id value to the second parameter using AEGP_SetStreamValue(). The layer_id is accessed via value2P.val.layer_id.

```cpp
AEGP_GetNewStreamValue();//get layer id from first param
value2P.val.layer_id;//this is the gotten layer id.
AEGP_SetStreamValue();//set it to the second param.
```

*Tags: `params`, `aegp`, `layer-checkout`*

---

## Why doesn't PF_Cmd_RENDER execute when an audio plugin uses PF_OutFlag_I_USE_AUDIO on a precomposed audio-only layer?

When a plugin sets PF_OutFlag_I_USE_AUDIO and the source layer is precomposed with no visual layers, After Effects may not trigger PF_Cmd_RENDER. The solution is to add PF_OutFlag_NON_PARAM_VARY to the global output flags in the plugin's global setup. This flag tells AE that the plugin's output varies based on non-parameter data (in this case, audio data), ensuring render commands are properly triggered even when the precomp contains only audio. Note: This flag is not required when the source layer is not precomposed, as AE automatically handles render triggering in that case.

*Tags: `audio`, `params`, `aegp`, `render-loop`, `layer-checkout`*

---

## Why does AEGP_RenderAndCheckoutLayerFrame cause a hard crash when trying to get layer pixels?

The issue was caused by incorrect pointer declaration. Instead of declaring AEGP_FrameReceiptH *receiptPH (pointer to handle), declare it as AEGP_FrameReceiptH receiptPH (handle itself) and pass it as &receiptPH to the function. Additionally, pass null values for the cancel function parameters instead of uninitialized variables.

```cpp
AEGP_FrameReceiptH receiptPH;
ERR(suites.RenderSuite5()->AEGP_RenderAndCheckoutLayerFrame(optionsH, NULL, NULL, &receiptPH));
ERR(suites.RenderSuite4()->AEGP_GetReceiptWorld(receiptPH, &worldH));
```

*Tags: `aegp`, `layer-checkout`, `debugging`, `memory`*

---

## How should an importer plugin interact with an effect plugin to share custom file data?

An importer plugin allows custom file types to be imported into After Effects and placed in compositions. There are two main scenarios for importer-to-effect interaction: (1) The importer parses the file on demand and populates an image buffer, which can then be read by the effect like any other layer source. After Effects can cache the result for faster access on subsequent frame renders. This approach expands usability for custom file types. (2) The effect uses the layer selector only to get the selected layer's ID, retrieves the source project item and file path from it, and then parses the file directly in the effect. This approach allows for data files without graphic interpretation and enables selective storage of data rather than loading all file data into RAM. Both scenarios are valid, and you can combine both approaches—parsing in the importer while also allowing the effect to fetch and parse the file path independently.

*Tags: `importer`, `aegp`, `layer-checkout`, `caching`, `arb-data`*

---

## How can an After Effects plugin determine the index of the layer it is currently applied to?

Use the PFInterfaceSuite1 to get the effect layer, then use LayerSuite8 to get the layer index. First call AEGP_GetEffectLayer(in_data->effect_ref, &layerH) to get the layer handle, then call AEGP_GetLayerIndex(layerH, layer_indexPL) to retrieve the layer's index.

```cpp
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
suites.LayerSuite8()->AEGP_GetLayerIndex(layerH, layer_indexPL);
```

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How can you differentiate between an adjustment layer and a precomp in After Effects plugin development?

Use AEGP_GetLayerFlags with the AEGP_LayerFlag_ADJUSTMENT_LAYER flag to determine if the adjustment switch is on or off for a layer. Both adjustment layers and precomps are considered video layers by AEGP_GetLayerObjectType, so this flag is necessary to distinguish them.

```cpp
AEGP_LayerH layerH;
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH));
ERR(suites.LayerSuite5()->AEGP_GetLayerFlags(layerH, &layerFlags));
// Check if AEGP_LayerFlag_ADJUSTMENT_LAYER is set in layerFlags
```

*Tags: `aegp`, `layer-checkout`, `params`*

---

## How do I access pixel data from different frames in an After Effects plugin?

The 'checkout' sample project in the After Effects SDK demonstrates how to retrieve images from points in time other than the currently processed frame. This is useful for plugins that need to reference previous or future frames, such as temporal effects or frame-blending operations.

*Tags: `sdk`, `reference`, `layer-checkout`, `frame-access`, `sample`*

---

## How do you set a Layer Parameter value from within the After Effects SDK?

Use AEGP_SetStreamValue() to set layer parameter values. First, get the effect layer and its parent composition using AEGP_GetEffectLayer() and AEGP_GetLayerParentComp(). Create the desired layer and get its ID with AEGP_GetLayerID(). Then retrieve the effect stream by index using AEGP_GetNewEffectStreamByIndex(), get a new stream value with AEGP_GetNewStreamValue(), set the layer_id field in the AEGP_StreamValue2 structure to your new layer's ID, and finally call AEGP_SetStreamValue() to apply the change. Remember to dispose of the stream value and stream when done.

```cpp
AEGP_StreamValue2 layerParamStreamVal;
ERR(suites.StreamSuite4()->AEGP_GetNewEffectStreamByIndex(aegp_plugin_id, effectRefH, PARAM_CONTENTLAYER_1_ID, &streamRefH));
ERR(suites.StreamSuite4()->AEGP_GetNewStreamValue(aegp_plugin_id, streamRefH, AEGP_LTimeMode_LayerTime, &zeroTime, TRUE, &layerParamStreamVal));
layerParamStreamVal.val.layer_id = newLayerID1;
ERR(suites.StreamSuite4()->AEGP_SetStreamValue(aegp_plugin_id, streamRefH, &layerParamStreamVal));
ERR(suites.StreamSuite4()->AEGP_DisposeStreamValue(&layerParamStreamVal));
ERR(suites.StreamSuite4()->AEGP_DisposeStream(streamRefH));
```

*Tags: `aegp`, `params`, `layer-checkout`, `sdk`*

---

## How can you get a layer handle from a composition handle to access composition settings like blending mode when working bottom-up through nested compositions?

There is no direct API to convert a composition handle (AEGP_CompH) to a layer handle (AEGP_LayerH). Instead, you must scan the project top-down: use AEGP_GetItemFromComp() to get the project item for the composition you're searching for, then scan all project items using AEGP_GetNextProjItem(). For each composition found, scan its layers using AEGP_GetLayerSourceItem() and match the layer's source with your target composition's source. This allows you to identify which layer in a parent composition references your nested composition, giving you the required AEGP_LayerH.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How do you get frame data like PF_world from sequence_data in After Effects plugins?

sequence_data is a PF_Handle that you should cast to whatever structure you're using. Since you know the structure you're working with, you can cast the handle directly to access the corresponding memory region. The key distinction is between smartFX and non-smartFX effects: non-smartFX effects get input images through params[0]->u.ld, while both types can use PF_CHECKOUT_PARAM() to specify source times. smartFX effects use checkout_layer_pixels() instead. For any source time checkout, you must set the PF_OutFlag_WIDE_TIME_INPUT flag. For accessing pixels outside layer params, use AEGP_GetReceiptWorld().

*Tags: `sequence-data`, `smartfx`, `layer-checkout`, `params`*

---

## Why doesn't changing a layer parameter's path trigger a re-render in After Effects?

Layer parameters track changes in the layer's source pixels only, not transforms, masks, or effects. This is intended behavior in After Effects. A layer selector provides the selected layer's source pixels, and only changes to those pixels trigger re-renders. To work around this, you can use an expression that checks the path value and drives a slider parameter to force updates, or alternatively create a light parented to the tracked layer with the I_USE_3D_LIGHTS flag set (though this approach has its own complications).

*Tags: `params`, `layer-checkout`, `render-loop`, `aegp`*

---

## How can you update a layer's mask using the After Effects SDK?

Use the AEGP_MaskSuite and AEGP_MaskOutlineSuite to update layer masks. Note that these suites require copying mask vertices individually. For rare operations or simpler mask copying between layers, consider using AEGP_ExecuteScript() with JavaScript, which may offer a more straightforward solution.

*Tags: `aegp`, `mask`, `sdk`, `layer-checkout`*

---

## How can you add one composition as a layer into another composition using ExtendScript?

You can add a composition item as a layer to another composition using the `comp.layers.add()` method in ExtendScript. The syntax is `comp.layers.add(otherCompItem);` where `otherCompItem` is the composition you want to add. More details can be found in the official After Effects scripting guide.

```cpp
comp.layers.add(otherCompItem);
```

*Tags: `scripting`, `extendscript`, `layer-checkout`*

---

## How should you properly manage and copy pixel buffer data when passing worlds between effects using AEGP_EffectCallGeneric?

When passing world data between effects, you need to understand ownership. AE-owned worlds (input/output buffers during render) will be released after the render call returns, so you must copy the pixel data using PF_COPY(). For plugin-owned worlds, you can pass the AEGP_WorldH directly. To copy: (1) create a new AEGP_WorldH with WorldSuite3()->AEGP_New() at the same dimensions, (2) wrap it with WorldSuite3()->AEGP_FillOutPFEffectWorld(), (3) copy pixels using PF_COPY(). In the receiving plugin, assign the AEGP_WorldH to sequence data and call WorldSuite3()->AEGP_Dispose() when done. Only dispose once, and let wrapped PF_EffectWorld structures go out of scope naturally.

```cpp
// Create and copy a world
AEGP_WorldH newWorldH;
suites.WorldSuite3()->AEGP_New(inputP->width, inputP->height, &newWorldH);

PF_EffectWorld newWorld;
suites.WorldSuite3()->AEGP_FillOutPFEffectWorld(newWorldH, &newWorld);

PF_COPY(&inputWorld, &newWorld, NULL, NULL);

// Send to other plugin via AEGP_EffectCallGeneric
suites.EffectSuite2()->AEGP_EffectCallGeneric(pluginID, effectH, &timeT, (void*)newWorldH);

// In receiving plugin's sequence data
seqDataWorldH = receivedWorldH;
```

*Tags: `aegp`, `arb-data`, `memory`, `layer-checkout`*

---

## How can I get a list of the selected layers when my plugin is activated?

You can check the currently selected layers using the AEGP function AEGP_GetNewCollectionFromCompSelection(). This allows you to retrieve the user's current layer selection without modifying the scene.

*Tags: `aegp`, `layer-checkout`, `sdk`*

---

## How can I replace an image on a layer using the After Effects C API?

There is no direct C API function to replace a layer's source footage. However, you can work around this by using AEGP_ExecuteScript() to execute JavaScript code from within your plugin. Get the layer's comp item ID, then use the JavaScript API to change the layer source. This approach keeps the JavaScript call internal to your plugin, so users are unaware it's being used.

```cpp
AEGP_FootageH footageH = NULL;
ERR(suites.FootageSuite5()->AEGP_NewFootage(S_my_id,
  psd_path,
  &key1,
  NULL,
  FALSE,
  NULL,
  &footageH));
AEGP_ItemH layer_itemH = NULL;
ERR(suites.FootageSuite5()->AEGP_AddFootageToProject(footageH, new_folderH, &layer_itemH));
// Then use AEGP_ExecuteScript() to change the layer source via JavaScript API
```

*Tags: `aegp`, `layer-checkout`, `scripting`, `api`*

---

## How do you get the correct coordinate in layer coordinate system when using the iterate function?

When a layer is partially off the comp, After Effects may give you a reduced buffer. The effectWorld structure shows you the offset values needed to correctly map pixel coordinates. When iterating, you need to account for this offset by using outPixel=GetColor(x+offsetX,y+offsetY). Additionally, there is an "iterate offset function" available in the SDK that can help handle this automatically. The input and output buffers for non-collapsed layers are in layer coordinates, but AE diminishes the buffer at 20% increments based on what part of the layer is out of sight, not just individual pixels.

```cpp
outPixel=GetColor(x+offsetX,y+offsetY)
```

*Tags: `layer-checkout`, `iterate`, `output-rect`, `sdk`*

---

## How can I check out only a portion of a frame at a different time instead of the entire frame?

You can use checkout_layer_pixels() which takes a request info structure as an argument that allows you to request a smaller rectangle. Alternatively, use the render suite to request the source item directly and set an ROI (region of interest) to get only the part you need. The render suite method doesn't hint AE in advance about your intentions, so AE won't cache items in advance—test both methods to see which is faster for your use case, as getting a potentially cached full frame might sometimes be faster than requesting a partial frame without caching.

*Tags: `layer-checkout`, `aegp`, `caching`, `output-rect`, `memory`, `render-loop`*

---

## Why does AEGP_GetLayerSourceItem return NULL even with a valid composition handle?

Some layers don't have a source item, including cameras, lights, text layers, and vector shapes. Before using AEGP_GetLayerSourceItem, verify that the layer is one that can have a source item. If itemH is NULL after the call, the layer type doesn't support source items.

*Tags: `aegp`, `layer-checkout`, `sdk`*

---

## How can I hide layers in a composition using AEGP without getting internal verification errors?

Use AEGP_SetDynamicStreamFlag to hide layers, but suppress errors with AEGP_StartQuietErrors() and reset the error variable (err = NULL) after each call. The ERR() macro will skip subsequent commands if an error has occurred, so resetting err is necessary to continue loop execution. Additionally, check the stream type before calling AEGP_SetDynamicStreamFlag() to ensure you're only operating on valid layer streams (AEGP_StreamType_LAYER_ID).

```cpp
ERR(suites.DynamicStreamSuite4()->AEGP_SetDynamicStreamFlag(streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, TRUE));
err = NULL;
ERR2(suites.StreamSuite4()->AEGP_DisposeStream(streamH));
```

*Tags: `aegp`, `layer-checkout`, `debugging`*

---

## How can I disable plugin parameters based on the layer type they are applied to?

Use the UPDATE_PARAMS_UI command instead of USER_CHANGED_PARAM to dynamically hide/unhide parameters. First, get the effect layer reference using AEGP_GetEffectLayer from PFInterfaceSuite1, then check the layer type using AEGP_GetLayerObjectType from LayerSuite5. The layer type will be one of: AEGP_ObjectType_AV (footage/solid/adjustment), AEGP_ObjectType_LIGHT, AEGP_ObjectType_CAMERA, AEGP_ObjectType_TEXT, or AEGP_ObjectType_VECTOR. Base your parameter visibility decisions on this layer type. The Supervisor sample demonstrates this pattern.

```cpp
AEGP_LayerH layerH;
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH));
AEGP_ObjectType layerType = NULL;
ERR(suites.LayerSuite5()->AEGP_GetLayerObjectType(layerH, &layerType));
```

*Tags: `params`, `ui`, `aegp`, `layer-checkout`*

---

## Why is a newly added layer not visible in the composition, showing the source layer instead on the current frame?

Adding a layer during a PF_Cmd_RENDER call causes rendering issues because the layers involved in the render are listed and checked prior to the render call. When you add a new layer during rendering, you mess with pre-made lists of render items. The solution is to add the new layer to the composition during other calls, preferably during PF_Cmd_UPDATE_PARAMS_UI or during idle process, not during render.

```cpp
ERR(suites.LayerSuite7()->AEGP_AddLayer(res_item, cResCompH, &res_layer));
```

*Tags: `aegp`, `layer-checkout`, `render-loop`, `debugging`, `params`*

---

## How can I access pixel data from an AEGP plugin?

An AEGP plugin cannot directly access layer pixels with effects, masks, or transformations applied. Instead, you can access the source pixels of a layer by getting the source item and then using AEGP_RenderAndCheckoutFrame() to render and checkout the frame, followed by AEGP_GetReceiptWorld() to retrieve the pixel data. The source provides pixels without any masks, effects, or transformation applied.

*Tags: `aegp`, `pixel-data`, `layer-checkout`, `reference`*

---

## How do you get the position stream for a nested composition layer in an AEGP plugin?

When you nest a composition (compA) into another composition (compB), the nested composition becomes a layer in compB. To get the position stream for that layer, you need to: 1) Use AEGP_GetCompLayerByIndex() on compB to get the AEGP_LayerH for the layer containing the nested composition, or receive it directly when nesting. 2) Then use AEGP_GetNewLayerStream() with AEGP_LayerStream_POSITION on that layer handle to get the position stream. Note that compositions themselves don't have positions—only layers within compositions do. If you're trying to find which comp a nested composition is being used in, you must scan the project and check each layer's source, as there is no direct API to query nesting relationships.

*Tags: `aegp`, `nested_comp`, `layer-checkout`, `reference`*

---

## Why don't animations on a nested composition appear when querying the source composition in an AEGP plugin?

When you have a composition (compA) with animated text layers that is then nested as a layer in another composition (compB) with animation applied to the nested layer itself, you will only see the text layer animations if you query compA directly. The animation applied to the nested composition layer in compB is separate and stored with that layer in compB, not in compA. To see both animations combined, you need to query the layer in compB that contains the nested composition and retrieve its position stream using AEGP_GetNewLayerStream() with AEGP_LayerStream_POSITION.

*Tags: `aegp`, `nested_comp`, `layer-checkout`, `params`*

---

## What are the best approaches for accessing and sampling specific pixels from a buffer in After Effects?

There are two main approaches: (1) Use the Sample Suite to grab color values from a pixel, subpixel, or area within a buffer - see the Shifter sample; (2) Access pixel data directly in RAM and retrieve the data yourself - see the CCU sample. Choose based on whether you need convenience (Sample Suite) or direct memory access performance.

*Tags: `memory`, `layer-checkout`, `sdk`, `reference`*

---

## What is the correct way to create custom hidden streams and store data on layers using AEGP?

There are limited options for storing custom data on layers with AEGP. Dynamic streams (like strokes from the paint tool and text animators) exist but are very specific. For persistent data across sessions, the recommended approach is to keep the data within the AEGP itself rather than on the layer, storing each layer's ID and its comp's project item ID to identify and retrieve the associated data. For temporary data that can be regenerated, storing it in the AEGP is also preferred. Alternative approaches include using effect parameters, layer comments, or layer marker comments, though these have limitations with the SDK.

*Tags: `aegp`, `arb-data`, `layer-checkout`, `sdk`*

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

## How can I detect whether a layer is audio-only in After Effects?

There are multiple approaches to detect audio-only layers: (1) Use AEGP_GetLayerFlags() to check AEGP_LayerFlag_VIDEO_ACTIVE and AEGP_LayerFlag_AUDIO_ACTIVE flags, though this only tells if video is off, not if it's a true audio layer. (2) Use AEGP_GetLayerSourceItem() to get the source item, then call AEGP_GetItemFlags() and check AEGP_ItemFlag_HAS_VIDEO and AEGP_ItemFlag_HAS_AUDIO—if the source has audio but no video, it's definitely audio-only. (3) As a last resort, use AEGP_ExecuteScript() with JavaScript for rare checks. Additionally, audio layers typically have dimensions of 0x0 in the project panel, which can serve as another detection criterion.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How can an AEGP plugin check if there is an active camera in the current composition?

Use the AEGP_GetCameraType() function with a layer handle to determine the camera type. This function returns one of three values: AEGP_CameraType_NONE (value -1) if no camera, AEGP_CameraType_PERSPECTIVE, or AEGP_CameraType_ORTHOGRAPHIC. Alternatively, iterate through the composition's layers and identify the first camera layer that is not out of trim at the current time—this will be the active camera. If no camera layers are found, the composition has no active camera.

```cpp
AEGP_GetCameraType(layerH) returns:
AEGP_CameraType_NONE = -1
AEGP_CameraType_PERSPECTIVE
AEGP_CameraType_ORTHOGRAPHIC
```

*Tags: `aegp`, `camera`, `composition`, `layer-checkout`*

---

## How can I set a camera to One-Node type when creating it via AEGP plugin?

Use AEGP_SetLayerFlag() with the flags AEGP_LayerFlag_LOOK_AT_POI or AEGP_LayerFlag_AUTO_ORIENT_ROTATION to control camera node type. Disabling these flags will give you one-node camera behavior. Alternatively, you can execute a script using AEGP_ExecuteScript() to set autoOrient properties directly.

```cpp
AEGP_ExecuteScript("app.project.item(compItem).layer(camLayerIndex).autoOrient = AutoOrientType.NO_AUTO_ORIENT");
```

*Tags: `aegp`, `camera`, `layer-checkout`*

---

## How can I access the composited color of layers beneath the current layer in After Effects?

There is no direct API to access intermediate composition buffers during normal effect rendering, as After Effects does not render bottom-to-top and optimizes by skipping rendering of fully opaque regions. However, several workarounds exist: (1) Write an AEGP artisan-type plugin (see the 'arti' sample) which handles composition rendering and has access to intermediate buffers, though this is very difficult; (2) Create a duplicate composition with only needed layers and render it using AEGP_GetReceiptWorld(); (3) Apply your effect on an adjustment layer, which receives the composited buffer of underlying layers, then use checked-out layer params to access original sources; (4) Use sampleImage() expressions on hidden parameters to sample pixel data, though this is slow and limited; (5) Ask the user to provide the background as a layer parameter, a common practice in effects that need background information.

*Tags: `aegp`, `render-loop`, `layer-checkout`, `sdk`, `params`*

---

## How can an effect force full resolution rendering when the user has downsampled the composition?

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

*Tags: `aegp`, `smart-render`, `layer-checkout`, `memory`*

---

## How can you re-apply a mask after modifying the alpha channel in an After Effects plugin?

There are two main approaches: (1) Use PF_MaskWorldWithPath on each mask and composite them in the correct transfer mode to recreate the layer's mask matte. Note that PF_MaskWorldWithPath does not support mask expansion, so you must render expanded masks manually. Read feather, opacity, and other mask values using AEGP_GetNewMaskStream or AEGP_GetNewMaskOpacity, then access numeric values with AEGP_GetNewStreamValue. (2) A simpler alternative: checkout the layer's original pixels (without masks applied) using a hidden layer parameter and PF_CHECKOUT_PARAM (or checkout_layer for smartFX), then compare the alpha difference between the original and masked input pixels to derive the mask matte. The second method is easier but less precise than fully recreating masks.

*Tags: `aegp`, `params`, `layer-checkout`, `mfr`, `smartfx`*

---

## How can you hide programmatically created layers like cameras from the AE timeline panel?

Use AEGP_GetNewStreamRefForLayer to get a stream reference for the camera layer, then call AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to hide it from the timeline. However, this approach has risks—a better alternative is to keep the layer visible but locked, similar to how plugins like Particular handle added elements, to prevent accidental deletion by users.

```cpp
AEGP_StreamRefH streamPH = NULL;
err = suites.DynamicStreamSuite3()->AEGP_GetNewStreamRefForLayer(NULL, layerPH, &streamPH);
err = suites.DynamicStreamSuite3()->AEGP_SetDynamicStreamFlag(streamPH, AEGP_DynStreamFlag_HIDDEN, false, true);
```

*Tags: `aegp`, `ui`, `layer-checkout`*

---

## How do you access pixel data from multiple layers in an After Effects effect plugin?

To access pixel data from multiple layers in an effect plugin, you need to create a layer parameter (e.g., param number 3) that allows the user to select any layer from the composition. Then use PF_CHECKOUT_PARAM to checkout that parameter. The pixel data can be accessed from the resulting paramDef structure at paramDef->u.ld.data (or similar location in the structure). This approach allows an effect on one layer to access and process pixels from any other layer in the composition.

```cpp
paramDef->u.ld.data
```

*Tags: `params`, `layer-checkout`, `aegp`, `mfr`*

---

## What is the difference between using layer parameters for effects versus using AEGP for multi-layer access?

For effect-type plugins, use a layer parameter with PF_CHECKOUT_PARAM to access multiple layers selected by the user. For AEGP (After Effects General Plugin) development, the approach is more complex and requires using AEGP_RenderAndCheckoutFrame, which involves finding the project item you want to reference. The layer parameter approach is simpler for effects and allows users to dynamically select which layers to access.

*Tags: `aegp`, `params`, `layer-checkout`, `render-loop`*

---

## How can I control layer copy-paste operations in an After Effects Effect plugin to restrict pasting within the same composition?

Use an AEGP (After Effects General Plugin) rather than an Effect plugin to intercept copy-paste operations. Register a command hook using AEGP_RegisterCommandHook to be notified whenever the user copies or pastes. This allows you to check the collection being pasted and remove or delete effects as needed. You can use AEGP_Command_ALL with AEGP_RegisterCommandHook to discover the command numbers for copy and paste operations. Attempting this from within an Effect plugin alone is not practical.

*Tags: `aegp`, `pipl`, `layer-checkout`, `scripting`*

---
