# Q&A: params

**412 entries** tagged with `params`.

---

## How do you handle backward compatibility when adding new parameters to existing plugins?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on new parameters. This flag tells AE to use the parameter's 'value' field (not 'dephault') when loading projects saved before this parameter existed. For more complex migration scenarios, add a hidden parameter storing a version number, and in sequence data setup, parse that version to reset values accordingly.

*Contributors: [**rowbyte**](../contributors/rowbyte/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2025-01-16 · Tags: `backward-compatibility`, `params`, `versioning`, `pf-param-flag`, `migration`*

---

## How do you detect when to invalidate a cached frame and prevent using stale results?

Create a hash string from all the important parameter values and settings that affect the output (time_step, time_scale, quality, downsample_x, downsample_y, and any non-keyframable settings like LOD). On each render, compare the current hash with the one from the last frame. If they differ or if the current time is not exactly one frame after the last cached time, reset the result_loaded flag to false.

```cpp
A_char result_hash[260];
sprintf(result_hash, "%d%d%d%d%d%i",
    in_data->time_step,
    in_data->time_scale,
    in_data->quality,
    in_data->downsample_x,
    in_data->downsample_y,
    lod
);

if (strcmp(result_hash, seqdataP->result_hash) != 0 || current_time != (seqdataP->result_last_time + in_data->time_step)) {
  seqdataP->result_loaded = false;
}
seqdataP->result_last_time = current_time;
strcpy(seqdataP->result_hash, result_hash);
```

*Tags: `caching`, `sequence-data`, `params`*

---

## How can sequence_data be used to set values in the output data?

The out_data parameter has sequence_data as a property that can be used to set the values of sequence data. This functionality works in After Effects 2021 and will also work in 2022 if you don't set the MFR flag, though it may show a warning in 2022.

*Tags: `sequence-data`, `mfr`, `params`*

---

## How do you prevent an After Effects plugin from appearing in Premiere Pro?

Set the PF_OutFlag_I_AM_OBSOLETE flag in out_data->out_flags when the application ID is 'PrMr' (Premiere Pro). This is done by checking if(in_data->appl_id == 'PrMr') and then setting out_data->out_flags = PF_OutFlag_I_AM_OBSOLETE;

```cpp
if(in_data->appl_id == 'PrMr')
{
    out_data->out_flags = PF_OutFlag_I_AM_OBSOLETE;
}
```

*Tags: `premiere`, `pipl`, `params`*

---

## How do you hide a banner in the timeline while showing it in the Effect Control Window?

The banner needs to be associated with a keyframeable parameter to show in both places like the Color Grid plugin does. For a static banner without keyframing, you need to configure it to only display in the ECW and not in the timeline.

*Tags: `ui`, `params`, `aegp`*

---

## What causes an error when working with arbitrary parameters?

The error occurs if you don't give the arbitrary (arb) parameter a name.

*Tags: `arb-data`, `params`, `debugging`*

---

## Is it illegal to checkout a parameter multiple times?

You cannot checkout parameters during sequence setup, resetup (when AE is loading the project), or sequence setdown. The world checkout can sometimes be empty in these contexts. Additionally, you must handle checkout errors and check in the parameter immediately after using it in loops.

*Tags: `params`, `layer-checkout`, `aegp`, `debugging`*

---

## When should parameter check-in be performed relative to parameter usage?

The check-in must be done immediately after getting the parameter value, especially in loop examples where the parameter is accessed multiple times.

*Tags: `params`, `layer-checkout`, `aegp`*

---

## Is it possible to set or change the value of a LAYER_PARAM in the plugin UI?

Yes, you can set the layer ID value in a streamVal2 using SetStreamValue.

*Tags: `params`, `ui`, `aegp`*

---

## Can you write to sequence data during PF_update_param_ui with MFR?

No, you cannot write to sequence data during PF_update_param_ui with MFR. You can only do it in the user changed param callback. If you need to modify layer defaults, you may need to use AEGP instead.

*Tags: `mfr`, `sequence-data`, `params`, `aegp`*

---

## How can you set a layer default parameter value?

Layer defaults do not have a direct 'value' property. You can try using PF_LayerDefault_NONE or use the AEGP suite to modify layer defaults.

```cpp
param.u.ld = PF_LayerDefault_NONE
```

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How do you extract scale values from a decomposed transformation matrix?

Use AEGP_MatrixDecompose4() on your result matrix to decompose it into position (posVP), scale (scaleVP), shear (shearVP), and rotation (rotVP) components. The scale values will be in the scaleVP output parameter.

```cpp
A_FloatPoint3 posVP, scaleVP, shearVP, rotVP;
A_Matrix4 resultMat4;
ERR(suites.MathSuite1()->AEGP_MatrixDecompose4(&resultMat4, &posVP, &scaleVP, &shearVP, &rotVP));
```

*Tags: `mfr`, `params`, `aegp`*

---

## How can I create a parameter slider with a percentage display and negative range (-100% to 100%) instead of the default 0% to 100%?

Use PF_ADD_FLOAT_SLIDERX instead of PF_ADD_FIXED, specifying the full range including negative values. The slider_min, slider_max, value_min, and value_max parameters should all support negative values. The PF_ValueDisplayFlag_PERCENT flag will display the values as percentages regardless of the range.

```cpp
PF_ADD_FLOAT_SLIDERX("Length", -100, 100, -100, 100, 100, PF_Precision_TENTHS, PF_ValueDisplayFlag_PERCENT, NULL, PARAM_LENGTH_ID);
```

*Tags: `params`, `ui`, `pipl`*

---

## How do you toggle a UI element's visibility on and off in an After Effects plugin?

To toggle UI element visibility, use the AEGP ParamUI thread approach instead of directly modifying ui_flags. Get the effect and stream references using AEGP_GetNewEffectForEffect and AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag with the AEGP_DynStreamFlag_HIDDEN flag. The issue with the direct ui_flags approach is that the bitwise NOT operator (!PF_PUI_INVISIBLE) doesn't properly clear the flag—you need to use the AEGP suite methods instead. Note that Premiere has a different method for accomplishing this.

```cpp
static PF_Err
changeParamVisibility(PF_InData            *in_data,
                      PF_OutData            *out_data,
                      PF_ParamDef          *paramsDef,
                      PF_ParamIndex        paramIndex,
                      PF_Boolean           paramVisibleB)
{
    PF_Err                err                    = PF_Err_NONE,
    err2                = PF_Err_NONE;
    global_dataP        globP                = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler        suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr')
    {
        AEGP_EffectRefH            meH                = NULL;
        AEGP_StreamRefH      currStreamH        = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex,     &currStreamH));
        if (meH && currStreamH)
        {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
        if (meH){
            ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
        }
        if ( currStreamH ){
            ERR2(suites.StreamSuite2()->AEGP_DisposeStream( currStreamH ));
        }
    }
    return err;
}
```

*Tags: `ui`, `params`, `aegp`, `premiere`*

---

## Is it possible to offset an Angle parameter by 90 degrees so that 0 degrees points to 3 o'clock instead of 12 o'clock?

You have to draw it as a custom parameter rather than using the stock angle parameter.

*Tags: `params`, `ui`*

---

## Can you reuse most of the drawing code from the stock angle parameter when creating a custom angle parameter with a different offset?

No, you have to draw it from scratch rather than reusing the stock parameter's drawing code.

*Tags: `params`, `ui`*

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

## How can we report progress while a blocking rendering computation is happening in the main thread?

The standard PF_PROGRESS approach won't work if the main thread is blocked. Instead, consider baking the computation in the backend (like in userchangedParam), updating an arb param during processing, storing the result in sequence data, and applying it in the render function (similar to how warp stabilizer works). This avoids blocking the main thread and allows proper progress reporting.

*Tags: `render-loop`, `params`, `arb-data`, `threading`*

---

## Can PF_ADD_ARBITRARY2 be used with PF_PUI_INVISIBLE flag?

PF_ADD_ARBITRARY2 can be used with PF_PUI_INVISIBLE, but there are specific requirements for the ARB param setup. The ARB needs special functions implemented correctly, similar to the ColorGrid example in the SDK.

*Tags: `arb-data`, `params`, `ui`*

---

## Should arbitrary data be stored in a separate hidden param or saved in the popup param itself?

An ARB is an independent param, so it should be a separate hidden parameter, not saved within the popup param.

*Tags: `arb-data`, `params`*

---

## Why does an arbitrary data struct work with int but fail with std::string?

std::string cannot be used in ARB data because it cannot be flattened or serialized to be saved on disk. Use char arrays (char[xxx] where xxx is the max size) instead for storing strings in arbitrary data.

*Tags: `arb-data`, `params`, `memory`*

---

## Can a CEP panel change parameter values of a C++ plugin?

CEP cannot directly change plugin parameter values. Parameter changes must be made through the plugin's own UI or other indirect mechanisms.

*Tags: `cep`, `params`, `ui`*

---

## How can a slider value be made to slide slower or with more precision without holding cmd/ctrl?

You can click on the slider and enter the value directly. If you control the code, you can use a UI flag such as PF_Precision_HUNDREDTHS or similar precision flags to control slider precision.

*Tags: `ui`, `params`*

---

## How can C++ plugin code communicate with a CEP panel?

C++ can open a CEP by executing scripting that calls the CEP. For CEP to plugin communication, store values to transfer in JavaScript memory using global variables. For large data sizes that are expensive in memory, use a temp file instead. Alternatively, set a hidden checkbox that is activated by script to catch the value. Another solution is to run a hidden panel as a server that receives requests from the plugin and adjusts values in CEP.

*Tags: `cep`, `scripting`, `params`, `memory`*

---

## Can C++ PersistentDataSuite3 and ExtendScript app.preferences share the same settings?

The question was asked but not answered in the conversation. The user clarified they meant app.preferences rather than app.settings, but no definitive answer was provided about whether these two APIs can share the same underlying settings storage.

*Tags: `scripting`, `aegp`, `params`*

---

## How can I make a slider value change more slowly or with finer precision without requiring modifier keys?

You can hold cmd/ctrl while dragging to slide slower, but there may be other approaches depending on your specific UI implementation. Consider implementing custom slider behavior with adjustable sensitivity or step values in your plugin's parameter interface.

*Tags: `ui`, `params`*

---

## What is the difference between in_data->sequence_data and out_data->sequence_data?

in_data->sequence_data is where you read sequence data. out_data->sequence_data is where you write to sequence data. In CC2022 and later, there are special functions for reading/writing. You can only write to sequence data in specific contexts: sequence setup, resetup, user changed param, do dialog, or external_dependencies callbacks. Use a static variable instead if you need to modify data more freely.

*Tags: `sequence-data`, `params`, `aegp`*

---

## What should you avoid doing with globalData in After Effects 2022 and later?

Do not write to globalData outside of globalsetup/setdown in AE 2022 and later, even for non-MFR plugins. After Effects has stricter requirements about globalData immutability. Instead, store mutable data in sequence data (with appropriate callbacks) or static variables.

*Tags: `mfr`, `params`, `memory`*

---

## How can you pass data between a button parameter and a script execution in an After Effects plugin?

Store the script text in an arbitrary data parameter. When a button is clicked, use AEGP_ExecuteScript() to execute the script string. You can pass data by find-and-replacing values in the script string before execution, and the script can return data back to After Effects.

```cpp
AEGP_ExecuteScript()
```

*Tags: `arb-data`, `params`, `ui`, `scripting`, `aegp`*

---

## Should script text be stored in an arbitrary parameter when implementing a button that triggers script execution?

Yes, the script text should be stored in an arbitrary parameter. A button parameter triggers the AEGP_ExecuteScript() function to execute the script string stored in the arb data.

*Tags: `arb-data`, `params`, `ui`, `aegp`*

---

## Is PF_Arbitrary_NEW_FUNC ever called in post-CS6 AE architecture?

No, PF_Arbitrary_NEW_FUNC is not called in post-CS6 architecture. Instead, you allocate a default Arb handle during ParamSetup, and for every new instance AE sends a PF_Arbitrary_COPY_FUNC selector with your default arb handle. The arb data won't be zero length.

*Tags: `arb-data`, `params`, `aegp`*

---

## What is the correct way to initialize arbitrary data in ParamSetup?

In ParamSetup, you fill the default arb data by calling a custom function to create the default arb. You allocate the default Arb handle during ParamSetup, and AE will send PF_Arbitrary_COPY_FUNC selectors with your default arb handle for each new instance.

*Tags: `arb-data`, `params`, `aegp`*

---

## How can you enable or toggle a slider parameter to be collapsed in After Effects?

Slider parameters cannot be collapsed individually. The PF_ParamFlag_COLLAPSE_TWIRLY flag only works with parameter groups (topics), not individual slider parameters. To collapse parameters, you need to use PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG on a group/topic, not on individual sliders.

```cpp
def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;
```

*Tags: `params`, `ui`, `pipl`*

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

## What is the best approach to restrict two float sliders so that Start cannot be equal to or greater than End?

Use the UserChangedParams callback to check which slider changed, then validate and update both parameter values accordingly. Avoid clamping min/max values dynamically during sliding, as this causes issues with keyframing where previously valid keyframes can become locked and unmutable. Instead, treat both parameters as unordered endpoints and use min() and max() of the two values in your code to determine Start and End.

```cpp
If sliderA has changed
    Check value of sliderA
    Check value of sliderB
    Apply value for both
If sliderB has changed
    Check value of sliderB
    Check value of sliderA
    Apply value for both
```

*Tags: `params`, `ui`, `debugging`*

---

## Why does changing the match name cause the CUI and parameters to become invisible in the ECW?

This was caused by an underlying C++ syntax error where a PF_Err variable was declared but not initialized, causing AEGP_GetNewEffectStreamByIndex to return 0x0. The real issue was that error checking with if(!err) statements was failing because the uninitialized error variable had a garbage value (usually 1) instead of PF_Err_NONE (0). The solution is to properly initialize all error variables when declaring multiple variables of the same type.

```cpp
// Wrong - err is undefined
PF_Err err, err2 = PF_Err_NONE;

// Correct - both variables initialized
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `pipl`, `aegp`, `debugging`, `params`*

---

## How can I check out the current layer (INPUT) with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT on the INPUT layer at different times, it returns the effectworld before all effects are applied, even those that precede the current effect in the chain. To get the current layer at different times with all effects applied, you need to use a different approach than simple PF_CHECKOUT. The Echo effect demonstrates that this is possible, likely through using layer composition or render callbacks that account for the full effect stack.

*Tags: `layer-checkout`, `render-loop`, `smartfx`, `params`*

---

## How can you ensure that PiPL flags are automatically updated when compiling without manual definition?

Define the flags in a header file using #define (e.g., FX_OUT_FLAGS) and reference that macro in the PiPL file. This way the PiPL auto-updates when compiling. However, this technique has limitations with higher bits/newest flags as it can cause overflow in Adobe's pipl compiler, in which case you must revert to manually defining flags as bits.

```cpp
#define FX_OUT_FLAGS (  PF_OutFlag_FORCE_RERENDER + \
                         PF_OutFlag_USE_OUTPUT_EXTENT + \
                         PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                         PF_OutFlag_DEEP_COLOR_AWARE + \
                         PF_OutFlag_CUSTOM_UI + \
                         PF_OutFlag_I_DO_DIALOG )

AE_Effect_Global_OutFlags {
    FX_OUT_FLAGS
}
```

*Tags: `pipl`, `build`, `params`*

---

## How can you force After Effects to de-cache frames when changing a plugin's globaldata variable?

Setting PF_OutFlag_FORCE_RERENDER alone may not work reliably when mixing in a globaldata variable via extra->cb->GuidMixInPtr. One workaround is to "kick" AE by triggering an internal frame cache check, such as changing the composition background color and then changing it back. However, this is not ideal; using GuidMixInPtr for forcing rerenders can work properly if implemented correctly.

*Tags: `caching`, `compute-cache`, `params`*

---

## In what scenarios would the nullptr checks for extraP and its members fail?

The checks for extraP, extraP->cb, and extraP->cb->GuidMixInPtr can fail in older versions of After Effects (before PF_AE130_PLUG_IN_VERSION) where PreRenderExtra or the callback structure are not available, or when the GuidMixInPtr function pointer is not implemented or initialized. The code also checks for a sentinel value (0xabababababababab) to detect uninitialized function pointers. These safeguards ensure compatibility across different AE versions and configurations where these features may not be present.

```cpp
if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION && extraP && extraP->cb && extraP->cb->GuidMixInPtr && (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab)
```

*Tags: `pipl`, `params`, `cross-platform`, `debugging`*

---

## What error occurs when trying to adjust effect parameters using AEGP during smartrender?

After Effects gives an error box saying 'error: effect attempting to modify a locked project' when trying to adjust the parameter of an effect using AEGP during a smartrender command.

*Tags: `smartrender`, `aegp`, `params`, `debugging`*

---

## What is a pseudoeffect and how is it used?

A pseudoeffect is a nicer alternative to using standard expression control effects like Slider Control or Point Control. It is written in XML and creates a regular-looking effect parameter UI but doesn't actually do anything—it just acts as controls that can be read from expressions, making the interface look more professional than having multiple Slider Controls.

*Tags: `params`, `ui`, `scripting`*

---

## Can compute cache data be accessed in globalsetup or paramssetup?

Compute cache must be defined during global setup, and the first access possible is during param setup. However, compute cache cannot be flattened. For persisting data in project files, sequence data or arb data should be used instead.

*Tags: `compute-cache`, `globalsetup`, `params`*

---

## How can instance-specific data be stored in a project file and accessed in global or paramssetup?

Arb data can be used for this purpose. Arb data can be accessed using special group functions for access and save operations, allowing you to store and retrieve instance-specific data that persists in the project file.

*Tags: `arb-data`, `globalsetup`, `params`, `sequence-data`*

---

## Can arb data parameter values be accessed in paramssetup?

Arb data can be accessed in paramssetup using the standard access functions (similar to how it's accessed in other functions like smartrender), allowing you to read arbP member variables.

*Tags: `arb-data`, `params`, `smartfx`*

---

## Is it possible to make dropdown options bold in an After Effects effect?

No, standard text formatting syntax like **bold**, *bold*, and <bold>bold</bold> do not work for dropdown options in After Effects effects.

*Tags: `ui`, `params`*

---

## What is required to support 32 bits per channel (32 bpc) in an After Effects effect?

You must use SmartFX if you want your effect to support 32 bpc.

*Tags: `smartfx`, `params`*

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

## How can a plugin maintain a unique ID per instance when duplicating, and why does sequence data become out of sync between the UI thread and render threads?

Sequence data is the appropriate mechanism for storing per-instance unique IDs. When duplicating, catch the duplication event and assign a new unique ID to the sequence data. However, sequence data may become out of sync on render threads even though it updates correctly in UpdateUI and UserChangeParams. Two potential solutions: (1) Check if the sequence is flattened in the project, as this can force render threads to use the correct sequence value, and (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of directly accessing in_data->seq_data, as the latter has not been updated correctly since CC 2021/2022.

*Tags: `sequence-data`, `render-loop`, `threading`, `params`*

---

## What is the purpose of the changedB parameter in PathOutline?

The changedB parameter appears to be a flag that indicates whether path data has been modified, though the exact public API for modifying path data is not exposed in the SDK. Setting it to TRUE may signal to After Effects that path data needs to be updated in the UI, but the internal APIs used by After Effects to modify path data are not publicly available.

*Tags: `params`, `ui`, `aegp`*

---

## Can mask outline modifications be done as a regular effect that procedurally modifies path vertices each frame rather than as a one-time destructive operation?

You cannot modify any project state in the render thread (like adding layers, changing layer properties, or masks). However, procedurally changing mask vertices might be possible using expressions. Baking the paths is also an effective alternative approach, though not as magical.

*Tags: `aegp`, `scripting`, `render-loop`, `params`*

---

## If mask path APIs are available in the JavaScript API, why couldn't they be used in C++ plug-ins by running the script within the plug-in?

The conversation suggests this is theoretically possible - you can run scripts within a plug-in to access JavaScript API functionality. However, this approach has limitations compared to native C++ implementation, particularly regarding per-character path separation which is still being lobbied for in the C++ plugin API.

*Tags: `scripting`, `aegp`, `params`, `debugging`*

---

## Can mask outlines be modified using AEGP_MaskOutlineSuite?

Yes, AEGP_MaskOutlineSuite can be used to modify mask outlines, but this approach would be destructive and one-time only, similar to running a script that modifies path vertices. For a procedural effect that modifies path vertices each frame before rasterization, a regular effect applied to a layer would be more appropriate.

*Tags: `aegp`, `params`, `render-loop`*

---

## What does 'connected instance to instance' mean in the context of multiple plugin instances?

It refers to having multiple instances of the plugin on the same or different layers communicating with each other.

*Tags: `aegp`, `params`, `threading`*

---

## How can you get the bit depth of a PF_LayerDef without using the world suite?

You can check the world_flags field of the PF_LayerDef. If PF_WorldFlag_DEEP is set, it's 16-bit. If PF_WorldFlag_RESERVED1 is set, it's 32-bit. Otherwise, it defaults to 8-bit.

```cpp
int get_bitdepth(const PF_LayerDef& layer) {
    if (layer.world_flags & PF_WorldFlag_DEEP) {
        return 16;
    } if (layer.world_flags & PF_WorldFlag_RESERVED1) {
        return 32;
    } else {
        return 8;
    }
}
```

*Tags: `params`, `debugging`*

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

## What is the official way to determine pixel format in Premiere using the AE SDK?

The official way to determine pixel format in Premiere using the AE SDK is to call the PF_PixelFormatSuite1, as seen in the SDK_Noise sample in the AE SDK since AE CS5. Using PF_WorldFlag_RESERVED1 was an unofficial/internal workaround by Adobe developers that is no longer recommended.

*Tags: `premiere`, `pipl`, `params`*

---

## Can the width-to-rowbytes ratio reliably determine bit depth?

While theoretically rowbytes can be anything, in reality padding is usually minimal, so the width-to-rowbytes ratio should reliably tell you the bit depth with certainty in practice. However, edge cases may exist and Adobe could change this behavior in the future.

*Tags: `params`, `memory`*

---

## How can a plugin with a new mode parameter maintain backward compatibility with older projects that lack this parameter?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on the parameter. This flag ensures that when opening projects created with older versions of the plugin that don't have the new parameter, After Effects will use the default value specified for that parameter rather than causing visual inconsistencies.

*Tags: `params`, `backward-compatibility`, `aegp`*

---

## How does After Effects store arbitrary data of a plugin to disk when that plugin is not installed?

Arbitrary data is serialized via PF_Cmd_Arbitrary_Callback using the flatten function, similar to sequence data. However, the flatten/unflatten versions are not accessible via AEGP functions. For plugins like 'curves' with stack variable handles, the data can be reinterpreted cast as char* and serialized. However, for plugins like 'liquify' that have pointers in their struct, serialization is problematic because deserialized pointer addresses are invalid on reload and will cause crashes.

*Tags: `arb-data`, `aegp`, `sequence-data`, `params`*

---

## Is there a way to serialize all plugin data including arbitrary data for template/capsule creation?

For first-party plugins where you know the handle, you can serialize the data yourself. However, for third-party and Adobe stock plugins, it is very tricky. The main challenge is accessing the flatten/unflatten versions of arbitrary data which are not exposed via AEGP functions. Some plugins have serializable stack variable handles (like 'curves') while others contain pointers (like 'liquify') that cannot be reliably serialized and deserialized. Extended script investigation may be a possible avenue to explore.

*Tags: `arb-data`, `aegp`, `scripting`, `params`*

---

## Is it possible to determine which stream was selected in the effect control window using AEGP API?

Based on the AESDK, there does not seem to be a way to know which stream was selected in the effect control window, unlike the timeline where you can use the selection collection to determine which streams are selected. Additionally, you cannot register a command inside the right-click context menu of the effect control window; the best alternative is to place the command in the Keyframe Assist menu.

*Tags: `aegp`, `ui`, `params`*

---

## Why is the PF_ParamDef* params[] pointer always nullptr when handling PF_Cmd_EVENT?

When handling PF_Cmd_EVENT, you need to perform a switch on extra->e_type. Parameter modification is only possible for specific event types, particularly click and keyboard events. The params pointer may not be available in all event types, and you may need to use AEGP functions instead of directly accessing the params pointer depending on the event type.

```cpp
case PF_Cmd_EVENT:
    err = HandleEvent(in_data, out_data, params);
    // Switch on extra->e_type
    // Check if event is click or keyboard type
```

*Tags: `params`, `aegp`, `ui`, `debugging`*

---

## How can you set a flag for the entire After Effects session across all plugins that resets on the next launch?

You can use the aegp suite with stream params, or implement a simpler solution using global variables in one of your plugins and dynamic symbol loading with dlopen(). This allows other plugins to dynamically load a function from that plugin to access and read the state of the shared global variable, avoiding the need for a separate companion aegp plugin.

*Tags: `aegp`, `params`, `cross-platform`, `scripting`*

---

## Should sequence data be flattened when using the sequence data suite?

Flattening sequence data is recommended and should work in most cases. However, it's not strictly required - some developers report success without flattening it. The flattening approach is more commonly used and tested.

*Tags: `sequence-data`, `params`*

---

## How should dynamic dropdown lists be updated in After Effects plugins as of CC2025.2 to avoid crashes and empty display?

Use the AEGP StreamSuite to update dropdown parameters via AEGP_SetStreamValue instead of directly modifying the param union during param_ui thread. Call AEGP_GetNewEffectStreamByIndex to get the stream reference, create an AEGP_StreamValue with the new value, use AEGP_SetStreamValue to apply it, and dispose the stream with AEGP_DisposeStream. This approach is more reliable than strncpy_s manipulation of namesptr, though it may have undo history implications that should be tested.

```cpp
PF_Err setFloatOrBoolOrDropdownParamViaAEGP(uint16_t i_param_to_change, float val,
                                     AEGP_PluginID my_id, AEGP_EffectRefH *effect_refHP, AEGP_SuiteHandler *suitesP, GlobalData* global_data)
{
    PF_Err err = PF_Err_NONE;
    AEGP_StreamValue streamvalue;
    streamvalue.val.one_d = val;
    AEGP_StreamRefH param_refH = NULL;
    ERR(suitesP->StreamSuite2()->AEGP_GetNewEffectStreamByIndex(my_id, *effect_refHP, i_param_to_change, &param_refH));
    streamvalue.streamH = param_refH;
    ERR(suitesP->StreamSuite2()->AEGP_SetStreamValue(my_id, param_refH, &streamvalue));
    ERR(suitesP->StreamSuite5()->AEGP_DisposeStream(param_refH));
    return err;
}
```

*Tags: `popup`, `aegp`, `params`, `ui`, `windows`*

---

## Does using AEGP_SetStreamValue to update dropdown parameters break the undo history in After Effects?

This is uncertain. The implementation includes a debug warning stating that using AEGP_SetStreamValue causes weird bugs in the undo history, but the actual extent and nature of these bugs is not definitively documented. Users should test this functionality in their specific implementation to understand the undo behavior.

*Tags: `aegp`, `params`, `debugging`*

---

## What is the exact behavior of the GuidMixInPtr callback from extra->cb in PreRender?

The GuidMixInPtr callback in PreRender can return a result that indicates whether a new render is needed or not. However, using or disabling this callback alone may not resolve SmartRender not being called if there are other underlying issues like SDK version mismatches.

```cpp
extra->cb->GuidMixInPtr
```

*Tags: `smartfx`, `smart-render`, `params`*

---

## Can global_outflags be set differently for specific host applications in PIPL?

Yes, you can set flags per host in global setup by checking the host application ID. Use a conditional check on in_data->appl_id to determine which host is running, then set different out_flags accordingly. For example, you can check if the application is not Premiere Pro (appl_id != 'PrMr') and apply different flags for After Effects versus other hosts.

```cpp
if (in_data->appl_id != 'PrMr') {
  out_data->out_flags = PR_OUT_FLAGS;
}
```

*Tags: `pipl`, `premiere`, `cross-platform`, `params`*

---

## Is it possible in PiPL to set global_outflags specific to a host app, such as removing a flag for Premiere while keeping it in After Effects?

Yes, it is possible to set global_outflags specific to host apps in PiPL files. You can use conditional logic or host-specific PiPL configurations to remove flags like no_params_vary for Premiere while maintaining them in After Effects.

*Tags: `pipl`, `premiere`, `params`*

---

## How can you unhover/highlight a button once the cursor goes outside the custom UI parameter area?

Use a debounce function to delay the unhover callback. Create a debounced callback that triggers after a delay (e.g., 2 seconds) when the mouse leaves the button area. Call this debounced callback every time you detect a mouse over event on the button. The callback should trigger an updateUI on the main thread so After Effects renders the custom UI and forces it to exit the hover state.

```cpp
template <typename Func, typename... Args>
std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {
    std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =
        std::make_shared<std::chrono::steady_clock::time_point>();
    return [=](Args... args) {
        *lastCallTime = std::chrono::steady_clock::now();
        std::thread([=]() {
            std::this_thread::sleep_for(delay);
            if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {
                func(args...);
            }
        }).detach();
    };
}
// Usage:
auto mouse_over_button = debounce(unhover_after_2s, std::chrono::milliseconds(2000));
// Call mouse_over_button() on each mouse over event
```

*Tags: `ui`, `custom-ui`, `params`*

---

## What is the correct way to read kerning property in After Effects 2024.3 character range API?

The autoKernType property needs to be set to 'none' or 'undefined' in order to read the kerning property. However, there appears to be a bug where it doesn't actually read that value correctly, particularly when keyframes are involved - it often stays as 'metric' even when kerning is manually set to a high value.

*Tags: `scripting`, `params`*

---

## Is the path value converted properly for 16/32 bit processing?

Eric was investigating whether path values are being converted correctly between 8-bit and 16/32-bit processing paths. The issue was occurring during a blur function and was resolved by precomposing the layer with the effect first, suggesting it may be a bit-depth conversion detail.

*Tags: `debugging`, `params`, `memory`*

---

## Is there a method for forcing masks on the layer to be applied after the plugin processes?

There is no built-in method to force masks to be applied after plugin processing. As a workaround, you can have the user select the mask type to None and compute the alpha yourself within your plugin, though this requires additional implementation effort.

*Tags: `params`, `ui`, `render-loop`*

---

## How can internally computed position values from a plugin render be exposed to users for use in expressions or as parameters?

The recommended approach is to output the internally computed values through a point control parameter. This allows users to access the generated positions either in expressions or by exporting them as nulls. The answer suggests this is a viable solution, with a reference to similar functionality being planned for the Vision plugin on aescripts.com.

*Tags: `params`, `ui`, `scripting`, `arb-data`*

---

## Is it possible to create a multi-layer 2D global illumination plugin where a renderer effect discovers and uses material properties from dummy effects on other layers?

This approach is theoretically possible with the After Effects SDK. You can use the AEGP suite to enumerate layers in a composition and checkout their properties. However, the key challenge is properly communicating dependencies to AE's internal dependency tracking system. You need to explicitly declare all layer and effect dependencies your renderer effect relies on, so that changes to materials or other layers trigger re-renders. This ensures the disk cache is used correctly—cached results are only used when all dependencies remain unchanged. The AEGP layer enumeration and property checkout APIs support this pattern, though it requires careful dependency management to avoid either missing updates or unnecessary re-renders.

*Tags: `aegp`, `layer-checkout`, `caching`, `params`, `compute-cache`*

---

## Is it possible to create a multi-layer effect that discovers material properties from per-layer dummy effects and performs global illumination rendering while properly communicating dependencies to After Effects' caching system?

Tobias Fleischer shared that he attempted a plugin-per-layer approach years ago but abandoned it due to render order troubles. Instead, he recommended using a single plugin with multiple layer parameters that pull in pixels from different layers, apply material/lighting options, and composite everything within that single plugin. This approach worked well despite AE's rigid parameter interface being somewhat clunky. The viability of the frankensteined multi-layer approach with proper dependency tracking depends on whether the SDK allows explicit communication of implicit dependencies to AE's caching system, though existing solutions tend to consolidate the logic into a single effect rather than relying on scattered per-layer effects.

*Tags: `params`, `layer-checkout`, `caching`, `smart-render`, `output-rect`*

---

## How can Plugin V2 detect when it's loaded in a project saved with V1 and default to Advanced Mode instead of Simple Mode?

Check the After Effects documentation for project compatibility and parameter versioning. The SDK provides mechanisms to detect legacy project data and conditionally set parameter visibility based on whether old parameters are present in the loaded project.

*Tags: `params`, `ui`, `scripting`*

---

## How can you hide parameters in Premiere within UpdateParamsUI?

To hide parameters in Premiere, use PF_UpdateParamUI with the PF_PUI_INVISIBLE flag set on the param's ui_flags. It's recommended to also set PF_PUI_DISABLED to avoid backend conflicts. The key is to ensure no error code (err) is set before calling PF_UpdateParamUI, as the function only executes when err is 0. Use bitwise OR to set flags (paramsDef[paramIndex].ui_flags |= PF_PUI_INVISIBLE) and bitwise AND with negation to unset them (paramsDef[paramIndex].ui_flags &= ~PF_PUI_INVISIBLE).

```cpp
if (!err && in_data->appl_id == kAppID_Premiere) {
    paramsDef[paramIndex].ui_flags |= PF_PUI_INVISIBLE;
    paramsDef[paramIndex].ui_flags |= PF_PUI_DISABLED;
    ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref,
        paramIndex,
        &paramsDef[paramIndex]));
}
```

*Tags: `params`, `ui`, `premiere`*

---

## How can you keep layer parameters synced between multiple locations when users apply keyframes and expressions to one?

This is acknowledged as a difficult problem. One approach mentioned is using AEGP or a script in userchangedparam to keep them synced, but keeping them synchronized when keyframes and expressions are applied to one location and mirroring it constantly is noted as being challenging.

*Tags: `aegp`, `scripting`, `params`, `ui`*

---

## How should material controls be organized when using child plugins on layers?

It is more convenient to have each set of material controls directly on the layer it is affecting rather than having indirection through the main effect's parameter list. This allows users to access layer-specific properties without having to search through a potentially long list of parameters in the main effect.

*Tags: `params`, `ui`, `layer-checkout`*

---

## How can dummy effects be used to specify layer properties without manual renaming?

The main effect loops over each layer's effects and looks for specifically-named instances of slider/checkbox controls. A more polished dummy effect can be created for specifying layer properties so users don't have to manually add slider controls and rename them, though the principle remains the same: these dummy effects wouldn't do anything themselves, just serve as property containers that the main effect reads from.

*Tags: `params`, `ui`, `layer-checkout`*

---

## How can you pass parameters between C and scripting languages in After Effects plugins?

Parameters can be passed as text using sprintf formatting. For example, parameters are formatted as text strings like "paramname=value" and then parsed by the scripting layer. Additionally, some bindings libraries (like PyCairo for Python or custom XML-based C binders for Lua) provide C APIs for assigning bitmap data and layer parameters directly between the compiled C code and the scripting layer.

```cpp
sprintf("%s=%s\n", paramname, value)
```

*Tags: `scripting`, `params`, `aegp`*

---

## What alignment guarantees does PF_LayerDef::rowbytes have in After Effects?

rowbytes will always be a multiple of sizeof(PF_Pixel8), which is 4 bytes, but 16-byte alignment is not guaranteed. The minimum guarantee is that rowbytes must be a multiple of the alignment requirement of the pixel type being used. Since After Effects effects don't have pixel types smaller than ARGB8, rowbytes will always be at least 4-byte aligned in practice. Misaligned pointers would result in undefined behavior according to the C specification.

*Tags: `memory`, `params`, `aegp`, `reference`*

---

## How can you trigger parameter updates in Premiere from CEP script interactions when PF_UserChangedParam doesn't work?

When working with AE/Premiere plugins updated by CEP backend data, using a hidden parameter updated via script that calls PF_UserChangedParam works well in After Effects, but Premiere does not recognize script interactions as user actions. Alternative approaches include monitoring external file dependencies for reactivity (though this can be slow), or implementing a manual button to force refresh as a workaround.

*Tags: `premiere`, `cep`, `params`, `scripting`, `cross-platform`*

---

## How should popup menu items be formatted in After Effects plugin parameters?

Popup menu items should be separated by pipe characters (|) with the separator placed at the end of each line, including the last item. For example: "choice1|" "choice2|" "Choice3". When using PF_ADD_POPUPX, ensure the parameter definition is properly initialized with AEFX_CLR_STRUCT(def) before adding the popup, and that there are no missing parameter definitions between the popup and any transition function that follows it.

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_POPUPX("Color", 3, 2,
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Tags: `params`, `ui`, `reference`*

---

## How do you define ARGB_8u color space in globalSetup for After Effects plugins?

When using ARGB_8u as your color space, you need to properly define it in the globalSetup function. The specific definition depends on your plugin architecture and color space requirements.

*Tags: `mfr`, `params`, `build`*

---

## Why does a plugin on a Premiere adjustment layer only receive the first keyframe value regardless of current time when checking out parameters?

This is a known issue in Premiere where plugins applied to adjustment layers with multiple keyframes on parameters receive only the value at the first keyframe, regardless of the current playhead time during parameter checkout. The issue affects all parameter types tested including sliders, angles, checkboxes, and 2D points. This behavior does not occur in After Effects, suggesting it is specific to Premiere's adjustment layer implementation.

*Tags: `premiere`, `params`, `layer-checkout`, `debugging`*

---

## What causes a plugin to not work properly in Premiere when it works in After Effects?

The issue can stem from the no_params_vary flag. In After Effects, this can be replaced using MIX_GUI during the pre_render thread. However, this flag has no effect in Premiere, so alternative approaches are needed for cross-application compatibility.

*Tags: `premiere`, `params`, `render-loop`, `cross-platform`, `aegp`*

---

## Why must X and Y positions be cast to FIX when using subpixel_sample_float for displacement sampling?

When using the subpixel_sample_float function from the Sampling Float Suite, coordinates must be converted to FIX format (fixed-point notation) as required by the AE API, even though you're working with float values. This is a requirement of the sampling function signature, not a rounding operation that degrades quality. The issue with displacement quality may relate to the sampling method chosen rather than the FIX conversion itself. Consulting displacement plugin implementations and tutorials can help identify if a different sampling approach would yield better results.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref, INT2FIX(newX), INT2FIX(newY), &giP->inSamp_pb, &samplePixel));
```

*Tags: `sampling`, `displacement`, `params`, `reference`*

---

## How do you hide an arbitrary parameter banner in the timeline while keeping it visible in the Effect Control Window?

When working with arbitrary parameters in After Effects, the visibility in the timeline versus the Effect Control Window (ECW) is tied to whether the parameter is keyframeable. Static banners that don't need keyframing can be hidden from the timeline while remaining visible in the ECW by ensuring they are not set up as keyframeable streams. The Color Grid effect demonstrates having parameters visible in both locations because they are keyframeable, but for static-only parameters, the visibility behavior differs.

*Tags: `arb-data`, `ui`, `params`, `debugging`*

---

## What causes an error when working with arbitrary parameters in After Effects plugins?

An error occurs when you don't provide a name for an arbitrary parameter (arb param). Always ensure that arbitrary parameters are given explicit names to avoid this error.

*Tags: `arb-data`, `params`, `debugging`*

---

## Is it illegal to checkout a parameter multiple times in After Effects plugins?

Checkout of parameters can be problematic in certain lifecycle stages. You cannot perform parameter checkout during sequence setup, resetup (when AE is loading the project), or sequence setdown. It's important to handle checkout errors properly and ensure that checkin is called immediately after using the parameter value, rather than holding the checkout open.

*Tags: `params`, `layer-checkout`, `debugging`, `aegp`*

---

## What should be done after checking out a parameter value in the render loop?

After getting a parameter value through checkout in a loop, you must call checkin immediately after using the value. Holding the checkout open or delaying checkin can cause errors like 'bad parameter passed to effect callback'. Always pair checkout with checkin as soon as the parameter data is no longer needed.

*Tags: `params`, `layer-checkout`, `render-loop`, `debugging`*

---

## Is it possible to set or change the value of a LAYER_PARAM in a plugin UI?

Yes, you can set/change the value of a LAYER_PARAM in your plugin UI by setting the layer ID value in a streamVal2 using SetStreamValue.

*Tags: `params`, `ui`, `aegp`*

---

## Can you write to sequence data during PF_update_param_ui callback?

No, you cannot write to sequence data during PF_update_param_ui. Writing to sequence data is only possible in the user changed param callback. With MFR, you also cannot write to sequence data during PF_update_param_ui. However, in certain cases you may be able to accomplish similar goals without AEGP by using PF_LayerDefault_NONE or similar approaches, though AEGP is often the recommended solution.

*Tags: `mfr`, `params`, `sequence-data`, `aegp`*

---

## How do you set a layer default parameter value in After Effects plugins?

Layer definitions do not have a direct 'value' property. Instead, you can use PF_LayerDefault_NONE or similar PF_LayerDefaults constants to set layer default parameters. For more complex parameter manipulation, using the AEGP API is recommended as an alternative approach.

```cpp
param.u.ld = PF_LayerDefault_NONE;
```

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How can I create a parameter slider with a negative range and percentage display in After Effects?

Use PF_ADD_FLOAT_SLIDERX (or PF_ADD_FIXED for integer values) with negative minimum and maximum values, combined with the PF_ValueDisplayFlag_PERCENT flag. Example: PF_ADD_FLOAT_SLIDERX("Length", -100, 100, -100, 100, 100, PF_Precision_TENTHS, PF_ValueDisplayFlag_PERCENT, NULL, PARAM_LENGTH_ID); This allows creating sliders with ranges like -100% to 100%, similar to Element 3D sliders.

```cpp
PF_ADD_FLOAT_SLIDERX("Length", -100, 100, -100, 100, 100, PF_Precision_TENTHS, PF_ValueDisplayFlag_PERCENT, NULL, PARAM_LENGTH_ID);
```

*Tags: `params`, `ui`, `slider`*

---

## How do you toggle the visibility of a UI parameter in After Effects?

To toggle parameter visibility, use the DynamicStreamSuite2 in the ParamUI thread rather than directly manipulating ui_flags. The issue with toggling ui_flags is that the bitwise NOT operator (!) doesn't work as expected for flag manipulation. Instead, get the effect and stream references, then use AEGP_SetDynamicStreamFlag with the AEGP_DynStreamFlag_HIDDEN flag. Set the last parameter to !paramVisibleB where paramVisibleB is true to show and false to hide the parameter. Note that Premiere Pro requires a different approach.

```cpp
static PF_Err
changeParamVisibility(PF_InData            *in_data,
                      PF_OutData            *out_data,
                      PF_ParamDef          *paramsDef,
                      PF_ParamIndex        paramIndex,
                      PF_Boolean           paramVisibleB)
{
    PF_Err                err                    = PF_Err_NONE,
    err2                = PF_Err_NONE;
    global_dataP        globP                = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler        suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr')
    {
        AEGP_EffectRefH            meH                = NULL;
        AEGP_StreamRefH      currStreamH        = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex,     &currStreamH));
        if (meH && currStreamH)
        {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
        if (meH){
            ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
        }
        if ( currStreamH ){
            ERR2(suites.StreamSuite2()->AEGP_DisposeStream( currStreamH ));
        }
    }
    return err;
}
```

*Tags: `ui`, `params`, `aegp`, `premiere`*

---

## How can I offset an Angle parameter by 90 degrees so that 0° points to 3 o'clock instead of 12 o'clock?

You need to implement a custom parameter to achieve this offset. While you may want to reuse drawing code from the stock angle parameter, you'll likely need to create a custom parameter implementation from scratch rather than simply modifying the existing one.

*Tags: `params`, `ui`, `custom-param`*

---

## How can I safely detect if a layer is available for checkout in a userChangedParam callback?

A workaround is to validate layer data by checking width/height and Lrect values—if they are random, over 100,000, or negative, the layer is not properly available for checkout. Note that Param[0].u.ld.data will be empty if checkout hasn't occurred in that thread. However, this is not a fully reliable solution, and the proper approach may require additional checks during effect duplication and shutdown phases.

*Tags: `layer-checkout`, `params`, `arb-data`, `shutdown`, `threading`*

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

## Can After Effects plugins currently access the Param source type for effects, masks, or sources through the dropdown list?

As of February 2023, there is no direct access to the Param source type effect/mask/source dropdown list in the After Effects plugin API. This limitation prevents plugins from programmatically controlling or reading which layer or effect is selected as a source parameter.

*Tags: `params`, `aegp`, `limitation`*

---

## Why is the paramUI suite slower when changing a parameter name in After Effects 2023?

The paramUI suite has been observed to be slower when changing a parameter name in After Effects 2023 (version 23.2.1). An alternative approach using the streamSuite has been found to work well as a workaround for this performance issue.

*Tags: `params`, `ui`, `performance`, `aegp`*

---

## Can you call PF_PROGRESS and PF_ABORT from a separate thread spawned during the render function?

No, you cannot call PF_PROGRESS and PF_ABORT from another thread. These functions must be called from the main thread. If you need to report progress during a blocking computation, you should consider alternative approaches such as baking the compute in a backend operation (like in userchangedParam), updating an arbitrary parameter during the update phase, storing results in sequence data, and applying them later (similar to how warp stabilizer works). This avoids blocking the main thread and allows proper progress reporting through the UI.

*Tags: `threading`, `render-loop`, `params`, `arb-data`*

---

## Can arbitrary parameters with PF_PUI_INVISIBLE flag be used to store custom data in After Effects plugins?

Yes, arbitrary (Arb) parameters can be used to store custom data invisibly in plugin parameters. However, there are important constraints: the data structure must be serializable and cannot contain C++ std::string types. Instead, use fixed-size char arrays (char[size]) for string data, as std::string cannot be flattened or saved to disk. Refer to the ColorGrid example in the SDK for proper Arb parameter setup including special functions for initialization and updates.

*Tags: `arb-data`, `params`, `ui`, `reference`*

---

## Why does using std::string in an arbitrary parameter data structure cause errors in After Effects plugins?

std::string cannot be flattened or serialized to disk, which is required for After Effects to save and restore arbitrary parameter data when projects are saved and reopened. Use fixed-size char arrays (char[max_size]) instead to store string data in Arb parameter structures.

*Tags: `arb-data`, `params`, `memory`*

---

## How can you make a slider value change more slowly for greater precision in After Effects plugins?

You can click on the slider value field and manually enter the desired value. Additionally, if you control the plugin code, you can use UI precision flags such as PF_Precision_HUNDREDTHS and similar options to control the slider's precision behavior.

*Tags: `ui`, `params`, `precision`*

---

## How can C++ plugins communicate with CEP panels to exchange parameter values?

There are several approaches: (1) CEP can change parameter values via scripting, and C++ can open a CEP by executing a script that calls it; (2) C++ can check if CEP is open using TCP protocol or a hidden checkbox that CEP activates via scripting; (3) Use a hidden panel running as a server that receives requests from the plugin and adjusts values in CEP; (4) For CEP-to-plugin communication, store values in JavaScript memory using global variables, or use temporary files if memory size is a concern; (5) In AE-only scenarios, set a hidden checkbox activated by script to catch the value.

*Tags: `cep`, `scripting`, `params`, `ui`, `communication`*

---

## How can you make a slider value change more slowly or with higher precision without holding modifier keys?

The user asked if there's a way to make slider values adjust more slowly or with greater precision. While they noted that holding cmd/ctrl provides this functionality, they were looking for a way to achieve slower sliding without requiring modifier key interaction. No complete solution was provided in the conversation.

*Tags: `ui`, `params`*

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

## What are the constraints on writing to sequence_data in After Effects CC2022 and later?

In CC2022 and later, you cannot write to sequence_data whenever you want. Writing is only allowed during specific events: sequence setup, sequence resetup, user changed param, do dialog, and external dependencies. For unrestricted modifications, use a global static variable initialized once instead. Note that in_data->sequence_data is for reading, while out_data->sequence_data is for writing.

*Tags: `sequence-data`, `params`, `memory`*

---

## How can you trigger frame invalidation from external background threads in After Effects plugins?

GuidMixInPtr can be problematic for triggering invalidation from background threads when using global or sequence data variables. A reliable workaround is to change a composition property (like background color) via AEGP calls or scripting, which forces After Effects to check if frames need invalidation. Another approach is to use a hidden parameter that gets modified through AEGP or scripting calls, though this has limitations when modal dialogs are open. The method must ultimately signal to After Effects through a mechanism it actively monitors.

*Tags: `smart-render`, `threading`, `sequence-data`, `params`, `aegp`*

---

## What is the recommended way to store plugin scripts as part of the project rather than in external folders?

Store scripts in arbitrary data parameters within the plugin rather than maintaining separate script files in external folders. This keeps scripts bundled with the project/plugin and avoids the clunky workflow of managing scripts outside the After Effects project structure.

*Tags: `arb-data`, `scripting`, `params`*

---

## How can you retrieve custom data computed from a previous frame in After Effects plugins?

You can use checkout Param during compute cache to access data from previous frames. However, this approach has limitations as it doesn't account for previous effects in the chain. For frame-sized custom data, you need to handle the data flow through the compute cache mechanism while being aware that parameter checkout alone may not fully solve the problem if previous effects need to be considered.

*Tags: `compute-cache`, `params`, `render-loop`, `arb-data`*

---

## Why is PF_Arbitrary_NEW_FUNC selector never received, causing arbitrary data to always be zero length initially?

This is a known behavior where the PF_Arbitrary_NEW_FUNC selector is not consistently triggered during arbitrary data initialization. The user reported that their arbitrary data always starts with zero length despite expecting the NEW_FUNC selector to be called. This appears to be an obscure aspect of the arbitrary data lifecycle that requires alternative initialization strategies.

*Tags: `arb-data`, `aegp`, `debugging`, `params`*

---

## Is the PF_Arbitrary_NEW_FUNC selector called in After Effects post-CS6 architecture?

No, PF_Arbitrary_NEW_FUNC is not called in the post-CS6 architecture. Instead, arbitrary data is initialized by allocating a default Arb handle during ParamSetup by calling a custom function to create the default arb data. For every new instance, After Effects sends a PF_Arbitrary_COPY_FUNC selector with your default arb handle. While the NEW_FUNC code can be left in place just in case, it will not be triggered in practice.

*Tags: `arb-data`, `params`, `aegp`, `reference`*

---

## How can you collapse a slider parameter in After Effects plugins?

To collapse a slider parameter, you need to set the global flag PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG and then set the param flag with 'def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;'. However, this approach may not work for individual sliders in the same way it works for topic groups.

```cpp
def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

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

## What is the best approach to restrict two float sliders so that one cannot be equal to or greater than the other?

Handle parameter changes in the UserChangedParams callback. When one slider changes, check the value of both sliders and apply corrective values to maintain the constraint. However, avoid updating parameter min/max values dynamically while sliding, as this can cause issues with keyframing—keyframes with previously acceptable values can become unmutable if they later violate the constraint. A better approach is to treat both parameters as interchangeable "endpoints" and use min() and max() in your code to determine which is Start and which is End, rather than enforcing strict ordering on the parameters themselves.

*Tags: `params`, `ui`, `scripting`*

---

## How can I get text layer path vertices in true layer space that match expression path.points() behavior, especially when the layer is parented?

PF_PathDataSuite does not return vertices in true layer space as documented in the SDK. Instead, it returns vertices in a hybrid space that is affected by 2D transforms (xy-translation, shear from parented transforms, Z Rotation) but not by z-translation, Orientation, or X/Y Rotation properties. To convert PF_PathDataSuite output to true layer space (matching what path.points() returns in expressions), you need to reverse the 2D transforms while preserving the unaffected 3D properties. Use AEGP_GetLayerToWorldXform() to convert to world space after obtaining layer-space vertices. The discrepancy exists because PF_PathDataSuite appears to return vertices in a space between composition and layer space, unlike expressions which correctly return layer-space coordinates independent of layer transform properties.

*Tags: `aegp`, `layer-checkout`, `params`, `reference`, `cross-platform`*

---

## Is there a way for an effect to programmatically reset itself using a command ID instead of manually setting all parameters to defaults via streams?

James Whiffin asked about programmatically resetting an effect using a command ID rather than manually resetting each parameter through streams. This question was not answered in the conversation.

*Tags: `params`, `aegp`, `pipl`*

---

## Is there a way for an effect to programmatically reset itself instead of manually setting all parameters to their defaults?

James Whiffin asked whether effects can use a command ID or similar mechanism to reset themselves programmatically rather than manually setting each parameter to default values via streams. This question was asked but no answer was provided in the conversation.

*Tags: `params`, `aegp`, `pipl`*

---

## Can compute cache data be transferred to arbdata for project persistence?

Yes, data from the compute cache can be transferred to arbdata if needed so it gets saved in the project. Additionally, you can read the compute cache from the arb or sequence thread if required.

*Tags: `compute-cache`, `arb-data`, `sequence-data`, `params`*

---

## How do you generate a key and create a pointer for computeIfNeeded or reading a function in After Effects plugins?

When you call computeIfNeeded or want to read a function, you create a pointer and must set in_data->pica_basicP in this pointer along with the other necessary values. The compute cache operates as something independent where you must send everything needed for the computation.

*Tags: `aegp`, `compute-cache`, `smart-render`, `params`*

---

## What is a best practice for managing PiPL outflags in plugin source code?

Define outflags as preprocessor macros in your plugin header file, then reference these macros in your PiPL resource. This way the flags are centrally defined and automatically updated during compilation, eliminating the need to manually touch the PiPL values. Example: define FX_OUT_FLAGS with all necessary flag constants (like PF_OutFlag_FORCE_RERENDER, PF_OutFlag_DEEP_COLOR_AWARE, etc.) and use them in AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 sections.

```cpp
#define FX_OUT_FLAGS (  PF_OutFlag_FORCE_RERENDER + \
                        PF_OutFlag_USE_OUTPUT_EXTENT + \
                        PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                        PF_OutFlag_DEEP_COLOR_AWARE + \
                        PF_OutFlag_CUSTOM_UI + \
                        PF_OutFlag_I_DO_DIALOG )

AE_Effect_Global_OutFlags {
    FX_OUT_FLAGS
},
AE_Effect_Global_OutFlags_2 {
    FX_OUT_FLAGS2
}
```

*Tags: `pipl`, `build`, `params`, `best-practice`*

---

## How should output flags be defined in an After Effects plugin to ensure they auto-update during compilation?

Define output flags using a macro in your plugin header file, then reference that macro in both the PiPL AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 entries. This ensures the PiPL automatically updates when you recompile. However, be aware that using higher bits/newest flags with this approach can cause overflow in Adobe's PiPL compiler, so for newer flags you may need to revert to manually defining flags as individual bits.

```cpp
#define FX_OUT_FLAGS (  PF_OutFlag_FORCE_RERENDER + \
                         PF_OutFlag_USE_OUTPUT_EXTENT + \
                         PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                         PF_OutFlag_DEEP_COLOR_AWARE + \
                         PF_OutFlag_CUSTOM_UI + \
                         PF_OutFlag_I_DO_DIALOG )

// In PiPL:
AE_Effect_Global_OutFlags {
    FX_OUT_FLAGS
},
AE_Effect_Global_OutFlags_2 {
    FX_OUT_FLAGS2
}
```

*Tags: `pipl`, `params`, `build`, `reference`*

---

## How can I force After Effects to invalidate cached frames when I modify global data in my plugin?

Simply setting PF_OutFlag_FORCE_RERENDER in outflags while changing a GuidMixInPtr value in extra->cb may not always work reliably. One workaround is to "kick" AE by triggering an internal frame cache check, such as changing the composition background color and changing it back. However, if you properly use GuidMixInPtr for forcing rerenders, it should work without this workaround—if you're experiencing issues, it may be worth reviewing source code examples from developers who have successfully implemented this.

*Tags: `compute-cache`, `smartfx`, `aegp`, `params`*

---

## What is a pseudoeffect in After Effects plugin development?

A pseudoeffect is a nicer alternative to using standard expression control effects like Slider Control or Point Control. Pseudoeffects are written in XML and create a regular-looking effect parameter UI without actually performing any rendering—they function purely as control elements that can be read from expressions. They provide a more professional appearance than exposing multiple Slider Controls directly to users.

*Tags: `ui`, `params`, `scripting`*

---

## Why does After Effects throw an 'effect attempting to modify a locked project' error when adjusting effect parameters during smartrender?

During a smartrender command, the project becomes locked to prevent modifications. Attempting to use AEGP calls to adjust effect parameters while smartrender is in progress results in this error because AEGP operations that modify the project cannot execute on the render thread. Such modifications must be deferred to the UI thread after the smartrender completes.

*Tags: `smart-render`, `aegp`, `threading`, `params`*

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

## Is SmartFX required for 32-bit per channel color depth support?

Yes, you must use SmartFX if you want your effect to support 32 bpc (32-bit per channel) color depth.

*Tags: `smartfx`, `params`*

---

## How can I detect if an input frame has actually changed in an After Effects plugin?

You can use PF_GetCurrentState to tell if an input has changed. Additionally, if an input frame changes you should certainly get a re-render. If you need to force another prerender, you can mix it into the guid mix in ptr.

*Tags: `params`, `render-loop`, `smartfx`, `layer-checkout`*

---

## How can you access layer pixels in Premiere Pro during parameter changes if checkout_layer_pixels is not available in Cmd_RENDER?

checkout_layer_pixels is available in PF_SmartRenderCallbacks for After Effects, but for Premiere Pro compatibility, you need to access layer pixels during the user changed parameter callback instead of the render command. The approach involves handling pixel checkout in the parameter change event rather than the standard render path.

*Tags: `premiere`, `aegp`, `params`, `layer-checkout`, `cross-platform`*

---

## How do you check out a frame of the current layer after prior effects have been applied in After Effects plugins?

To check out a frame after prior effects are applied (rather than before), you need to use a different approach than PF_CHECKOUT_PARAM, which gets the frame before all prior effects. The conversation indicates this is possible but the specific method was not detailed in the exchange. The questioner was trying to retrieve a frame after the Exposure effect but before the AI Color Match effect in the effect stack.

*Tags: `aegp`, `layer-checkout`, `params`, `render-loop`*

---

## How do you store a struct in global_data in an After Effects plugin?

You can store a struct in global_data by creating a pointer to it in the global_data structure, then in PF_GlobalSetup allocate it with `globaldata->structPointer = new StructName();`. While you can optionally delete it in PF_GlobalSetdown, this is not strictly necessary since the OS will clear the memory when After Effects closes.

```cpp
// In globaldata definition:
StructName* structPointer;

// In PF_GlobalSetup:
globaldata->structPointer = new StructName();

// Optional in PF_GlobalSetdown:
delete globaldata->structPointer;
```

*Tags: `memory`, `aegp`, `params`*

---

## How can you ensure sequence data stays synchronized across render threads when duplicating a plugin instance?

When duplicating a plugin and assigning a new unique ID to sequence data, the sequence data may become out of sync on render threads even though UpdateUI and UserChangeParams receive the correct values. Two potential solutions: (1) Check if the sequence is flattened in the project, as flattening can force render threads to get the correct sequence value. (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of relying on in_data->seq_data directly, as the latter is not updated correctly since CC2021/2022.

*Tags: `sequence-data`, `threading`, `render-loop`, `params`, `debugging`*

---

## How can you reliably assign a persistent ID to an effect instance that survives effect reordering?

One approach is to hash the layer ID, composition ID, and index of the effect together. However, this has the limitation that if the effect's index changes (e.g., when effects are reordered), the ID must be recalculated. An alternative suggestion from Adobe forums is to store a copy of the ID in a hidden arbitrary parameter and use the UI/ARB thread to update the value from sequence data, though this approach has not been thoroughly tested and may not be intuitive to implement.

*Tags: `arb-data`, `params`, `ui`, `aegp`*

---

## What is the purpose of the changedB parameter in PathOutline, and can paths be modified through the SDK?

The changedB parameter appears to signal whether path data has been modified, though the exact mechanism is unclear. The public SDK does not expose functions for modifying paths—they are read-only. It's theorized that After Effects internally uses an undocumented API for path modification that mirrors the PathOutline disposal API, but this functionality is not available to plugin developers.

*Tags: `aegp`, `params`, `sdk`, `reference`*

---

## Can you procedurally modify mask vertices each frame in an After Effects plugin effect?

You cannot modify project state (including masks) in the render thread of a plugin. However, procedurally changing mask vertices might be possible using expressions. Alternatively, 'baking' the paths is an effective workaround, though not as dynamic. If the functionality is available in the JavaScript API, you could potentially run scripts from within your plugin to achieve this.

*Tags: `aegp`, `render-loop`, `params`, `scripting`*

---

## What is the AEGP_MaskOutlineSuite and how does it differ from procedural mask modification?

The AEGP_MaskOutlineSuite allows you to work with mask outlines, but modifications made through it would be destructive and one-time only (similar to running a script that modifies path vertices). This differs from a regular effect plugin that could procedurally modify path vertices each frame before rasterization.

*Tags: `aegp`, `params`, `scripting`*

---

## Can AEGP_MaskOutlineSuite be used to procedurally modify mask path vertices each frame as a regular effect?

AEGP_MaskOutlineSuite can modify mask outlines, but the distinction is between destructive/one-time modifications (like running a script that permanently changes path vertices) versus procedural, non-destructive modifications applied each frame before rasterization. A regular effect that procedurally modifies path vertices each frame would be the latter approach, allowing the modifications to be recalculated and applied dynamically during rendering rather than permanently altering the mask data.

*Tags: `aegp`, `mask`, `render-loop`, `params`*

---

## What does 'connected instance to instance' mean in the context of After Effects plugins?

It refers to having multiple instances of the plugin on the same or different layers communicating with each other.

*Tags: `aegp`, `params`, `architecture`*

---

## How do you handle backward compatibility when adding new parameters with default values to an After Effects plugin?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag. This mechanism ensures that when opening older projects that don't have a new parameter, the plugin will use the specified default value without breaking the appearance of those projects. This is the standard way to handle parameter additions across plugin versions.

*Tags: `params`, `backward-compatibility`, `aegp`*

---

## How can you handle migration of sequence data when the plugin format changes over versions?

Add an extra hidden parameter that stores the version number when sequence data is saved. When loading, parse this version information and reset values accordingly. This approach becomes especially useful when you have complex migration logic to handle multiple format versions.

*Tags: `sequence-data`, `params`, `arb-data`, `versioning`*

---

## How does After Effects serialize arbitrary data from plugins to disk?

Arbitrary data is serialized using PF_Cmd_Arbitrary_Callback with the flatten function, similar to sequence data. However, the flatten/unflatten versions are not accessible via AEGP functions. For plugins with stack-variable handles (like "curves"), the handle can be reinterpreted cast as char* and serialized. Plugins with pointers in their structures (like "liquify") cannot be easily serialized this way because deserialized pointer addresses are invalid on reload, causing crashes.

*Tags: `arb-data`, `serialization`, `aegp`, `params`*

---

## How can an AEGP detect which parameter stream is selected in the effect control window?

There is currently no documented way in the After Effects SDK to determine which stream was selected in the effect control window (ECW). Unlike the timeline where you can use the selection collection to identify selected streams, the ECW does not provide equivalent selection APIs. As a workaround, developers typically place custom commands in the Keyframe Assist menu rather than registering them in the right-click context menu, which does not support command registration.

*Tags: `aegp`, `params`, `ui`, `selection`*

---

## Why is the params pointer always nullptr when handling PF_Cmd_EVENT?

When handling PF_Cmd_EVENT, you need to check the event type via extra->e_type first. You can only modify param values for certain event types, specifically click and keyboard events. The params pointer may not be available in all event contexts, and you may need to use AEGP functions instead to modify parameters depending on the event type.

```cpp
case PF_Cmd_EVENT:
    err = HandleEvent(in_data, out_data, params);
    // Check extra->e_type for event type (click, keyboard, etc.)
    switch(extra->e_type) {
        case PF_Event_CLICK:
        case PF_Event_KEYDOWN:
            // Can modify params here
            break;
    }
```

*Tags: `params`, `aegp`, `ui`, `debugging`*

---

## How should dynamic dropdown lists be updated in After Effects plugins to avoid crashes in CC 2025.2?

Use the AEGP StreamSuite to update dropdown parameters instead of directly modifying the param_union. Create a function like setFloatOrBoolOrDropdownParamViaAEGP that uses AEGP_GetNewEffectStreamByIndex and AEGP_SetStreamValue to properly update the parameter value. Call this during UpdateParamsUI. Note: There is a known warning that this approach may cause issues with undo history, though the exact impact is unclear.

```cpp
PF_Err setFloatOrBoolOrDropdownParamViaAEGP(uint16_t i_param_to_change, float val,
                                     AEGP_PluginID my_id, AEGP_EffectRefH *effect_refHP, AEGP_SuiteHandler *suitesP, GlobalData* global_data)
{
    PF_Err err = PF_Err_NONE;
    AEGP_StreamValue streamvalue;
    streamvalue.val.one_d = val;
    AEGP_StreamRefH param_refH = NULL;
    ERR(suitesP->StreamSuite2()->AEGP_GetNewEffectStreamByIndex(my_id, *effect_refHP, i_param_to_change, &param_refH));
    streamvalue.streamH = param_refH;
    ERR(suitesP->StreamSuite2()->AEGP_SetStreamValue(my_id, param_refH, &streamvalue));
    ERR(suitesP->StreamSuite5()->AEGP_DisposeStream(param_refH));
    return err;
}
```

*Tags: `params`, `ui`, `aegp`, `popup`, `windows`*

---

## Is there a reference implementation of dynamic dropdown list handling in After Effects plugins?

Diffusae (an AE plugin project) has a working implementation for updating dropdown parameters like a models list. It uses the AEGP StreamSuite approach with AEGP_GetNewEffectStreamByIndex and AEGP_SetStreamValue called from UpdateParamsUI. This approach is more reliable than directly manipulating param_union.pd.u.namesptr.

*Tags: `params`, `ui`, `aegp`, `open-source`, `reference`*

---

## What does the GuidMixInPtr callback in PreRender do?

The extra->cb->GuidMixInPtr callback in PreRender can indicate whether a new render is needed. If this callback suggests no new render is needed, SmartRender may not be called, which is expected behavior.

```cpp
extra->cb->GuidMixInPtr
```

*Tags: `smart-render`, `params`, `aegp`*

---

## Can you set global_outflags in a pipl file to be host-application specific?

The user asked whether pipl files support setting global_outflags with host-app-specific conditions. They wanted to disable the no_params_vary flag for Premiere due to a bug while keeping it enabled in After Effects. No definitive answer was provided in the conversation, so this question remains unresolved.

*Tags: `pipl`, `premiere`, `aegp`, `params`*

---

## Is it possible in PiPL to set global_outflags specific to a host application like Premiere?

Yes, you can set global_outflags with host-specific conditions in PiPL files. One approach is to use GUID-mix with time values to conditionally apply flags like no_params_vary for different host applications, allowing you to remove a flag for Premiere while keeping it enabled in After Effects.

*Tags: `pipl`, `premiere`, `params`, `aegp`*

---

## How can you unhover/remove highlight from a custom UI button when the cursor leaves the parameter UI area?

Use a debounce pattern with a timer to detect when the mouse has left the button area. Create a debounce function that wraps a callback (like unhover_after_2s) with a delay. Call this debounced callback every time a mouse-over event occurs on the button. The callback should trigger an updateUI on the main thread, which forces After Effects to re-render the custom UI and remove the hover state. This prevents the button from staying highlighted when the cursor moves outside the UI area.

```cpp
template <typename Func, typename... Args>
std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {
    std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =
        std::make_shared<std::chrono::steady_clock::time_point>();
    return [=](Args... args) {
        *lastCallTime = std::chrono::steady_clock::now();
        std::thread([=]() {
            std::this_thread::sleep_for(delay);
            if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {
                func(args...);
            }
        }).detach();
    };
}
// Usage: auto mouse_over_button = debounce(unhover_after_2s, std::chrono::milliseconds(2000));
```

*Tags: `ui`, `custom-ui`, `threading`, `params`*

---

## How should autoKernType be set to read kerning property values in After Effects text layers?

The autoKernType property needs to be set to either 'none' or 'undefined' in order to read the kerning property. When autoKernType is set to 'metric' or 'optical' (which means automatic by the host), the kerning property may not read or write as expected. Manual kerning is indicated by setting autoKernType to 'undefined', but this behavior appears to have issues when keyframes are involved in AE 2024.3.

*Tags: `scripting`, `params`, `debugging`*

---

## What flag should be set in global setup to handle premultiplied alpha correctly?

Set the flag PF_outflags2_REVEALS_ZERO_ALPHA in global setup to properly handle cases where the plugin reveals zero alpha, which is related to the premultiplied alpha of the input layer.

*Tags: `pipl`, `params`, `alpha`, `mfr`*

---

## How do you create collapsible parameter groups in After Effects plugins?

To create collapsible parameter groups, set the global PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG flag. Then for each individual parameter, set PF_ParamFlag_START_COLLAPSED if that parameter should be collapsed by default, or omit the flag if it should not be collapsed.

*Tags: `params`, `ui`, `pipl`*

---

## How can you create a render-only version of an After Effects plugin that doesn't require a GUI license on render farms?

One approach is to make all parameters invisible while still allowing the plugin to read them, effectively creating a read-only version of the plugin. This way, the plugin can process parameters set during the authoring phase without requiring interactive UI on render farm machines.

*Tags: `params`, `ui`, `deployment`, `render-loop`*

---

## Can a plugin expose generated data back to the user for use in expressions?

The question was asked but not answered in the conversation. No response was provided about whether plugins can expose data to expressions or other user-accessible formats.

*Tags: `aegp`, `params`, `scripting`*

---

## How can internally computed positions from a plugin be exposed to users in After Effects?

Internally computed positions can be exposed to users by outputting them through a point control parameter. This allows users to index the values in expressions or access them programmatically. One practical example is exporting bounding boxes as nulls, which is a technique used in projects like Vision (https://aescripts.com/vision).

*Tags: `params`, `ui`, `scripting`, `expressions`, `arb-data`*

---

## Can an After Effects effect plugin compute heavy calculations and update parameter values from the render function?

This appears to be an unsolved use case. The developer wants to perform computationally expensive operations in an effect plugin's render function and have those results update a parameter slider, but current After Effects plugin architecture doesn't provide a straightforward mechanism for plugins to push parameter updates back to the host during rendering. Parameters are typically read during render, not written to.

*Tags: `params`, `render-loop`, `aegp`, `threading`*

---

## How can internally computed positions generated during plugin rendering be exposed to users for use in expressions or parameters?

When a plugin generates its own set of positions as part of the render process (rather than reading from an existing AE path), those values can potentially be exposed to users in two ways: (1) by indexing them in expressions, allowing users to reference computed values dynamically, or (2) by outputting them through a point control parameter, which would make the values available as a controllable/readable parameter in the After Effects UI.

*Tags: `params`, `ui`, `scripting`, `aegp`*

---

## Can you pass text output from an effect to a text layer instead of rendering it yourself?

Yes, you can encode text data into pixels and pass it to a text layer. One approach is to use dead pixels (out-of-frame pixels or pixels with alpha of 0) to encode the string data in RGB color values. This allows an effect that produces both pixels and text to delegate text rendering to a native text layer rather than rendering it directly.

*Tags: `ui`, `params`, `aegp`, `output-rect`*

---

## Is it feasible to build a 2D global illumination plugin using material property metadata effects on individual layers with a separate renderer effect?

Yes, this architecture is theoretically viable with the AE SDK. The pattern involves creating dummy effects on individual layers that only expose material parameters (emissive, diffuse, glass properties, etc.) without rendering output, while a separate renderer effect on an adjustment layer or solid discovers and interprets these metadata effects at render time. The key challenge is properly communicating cross-layer dependencies to AE's caching system so changes to materials trigger re-renders of the expensive global illumination computation while still benefiting from disk caching when unrelated elements change.

*Tags: `aegp`, `params`, `layer-checkout`, `caching`, `smart-render`, `rendering`*

---

## What are the practical challenges and solutions when designing multi-layer effects in After Effects plugins?

According to Tobias Fleischer (reduxFX), a plugin-per-layer approach was abandoned in favor of a single plugin with multiple layer parameters due to render order troubles. The successful approach involved creating a single plugin that accepts 10 layer parameters to pull in pixels from multiple layers, applies lighting/material options, and composites everything together within that single plugin. While this worked well, AE's rigid parameter interface made it somewhat clunky to use. The key insight is that centralizing the multi-layer logic into a single plugin avoids render order dependency issues that arise from distributing logic across multiple per-layer plugins.

*Tags: `render-loop`, `params`, `ui`, `layer-checkout`, `smartfx`*

---

## What approach did reduxFX use to handle multi-layer compositing in After Effects plugins?

Instead of using a plugin-per-layer approach (which caused render order issues), reduxFX implemented a single plugin with 10 layer parameters that could pull in pixels from multiple layers, apply lighting/material options, and composite everything together within that single plugin. While AE's rigid parameter interface made it somewhat clunky to use, this approach worked surprisingly well and was later ported to node-based hosts like Nuke and Natron where the multi-input approach worked more naturally.

*Tags: `params`, `render-loop`, `mfr`, `ui`*

---

## How can I detect when a plugin loads an old project to change default UI visibility between versions?

You can use the After Effects documentation to detect project compatibility. When loading an old V1 project in V2, check if legacy parameters exist or have non-default values during initialization, then programmatically set your UI mode (Simple vs Advanced) accordingly. The actual implementation details are found in the official AE SDK documentation.

*Tags: `params`, `ui`, `scripting`, `reference`*

---

## Can I use checkout_layer() during PF_Cmd_SMART_PRE_RENDER to access arbitrary composition layers not exposed as effect parameters?

According to maxon developer Alex Bizeau, you can only grab layer params directly during SMART_PRE_RENDER. However, you can use AEGP_RenderAndCheckoutLayerFrame with any layer IDs, and both sync and async versions exist. For multiple layers, a workaround is to use hidden layer parameters (up to 999) assigned via a button script that updates the comp, so checkout_layer() can access them. You may also handle dynamic layer order/removal by looping through the comp in SmartPreRender and mixing layer order into GuidMixInPtr() for dependency tracking.

*Tags: `smart-render`, `layer-checkout`, `aegp`, `params`, `caching`*

---

## How can I make a multi-layer effect that depends on material properties defined by effects on other layers while maintaining smart render efficiency?

The most robust approach is to use hidden layer parameters (you can have up to 999) and a button parameter that triggers a script to assign layers from the composition to those parameters one by one. Alternatively, loop through the comp in SmartPreRender and mix layer order into GuidMixInPtr() for dependency tracking. This allows the renderer effect to check out all layers efficiently while maintaining AE's caching and smart render system. You only need to click an 'Update Comp' button when you add layers; layer order changes or removals can be handled automatically by the SmartPreRender logic.

*Tags: `smart-render`, `layer-checkout`, `params`, `caching`, `compute-cache`*

---

## How do you hide parameters in Premiere Pro using PF_UpdateParamUI?

To hide parameters in Premiere Pro, check if the application is Premiere using `in_data->appl_id == kAppID_Premiere`, then set the `PF_PUI_INVISIBLE` flag on the parameter's `ui_flags`. It's also recommended to set `PF_PUI_DISABLED` to avoid conflicts. Use bitwise OR to set flags (`paramsDef[paramIndex].ui_flags |= PF_PUI_INVISIBLE`) and bitwise AND with negation to unset them (`paramsDef[paramIndex].ui_flags &= ~PF_PUI_INVISIBLE`). Call `PF_UpdateParamUI` with the modified parameter definition. Ensure that no error code is set before calling this function, as it will only execute if `err` is 0.

```cpp
if (!err && in_data->appl_id == kAppID_Premiere)
{
    paramsDef[paramIndex].ui_flags |= PF_PUI_INVISIBLE;
    paramsDef[paramIndex].ui_flags |= PF_PUI_DISABLED;
    ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref,
        paramIndex,
        &paramsDef[paramIndex]));
}
```

*Tags: `params`, `premiere`, `ui`*

---

## Is there an open-source After Effects plugin project that demonstrates parameter manipulation?

ISF4AE is an open-source project on GitHub that demonstrates various parameter handling techniques for After Effects plugins, including dynamic parameter visibility and manipulation. It can be a useful reference for plugin developers working with parameters.

*Tags: `open-source`, `reference`, `params`*

---

## How can you efficiently cache data when scraping parameters from dummy effects on multiple layers?

When implementing a control scheme using layer parameters on the main effect with dummy effects on other layers, call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects. This ensures After Effects caches everything properly and maintains good performance even when reading all layers one by one.

*Tags: `aegp`, `params`, `caching`, `memory`, `layer-checkout`*

---

## What is a good approach for implementing multi-layer effect controls in After Effects plugins?

A viable user-friendly approach is to use layer parameters on the main effect combined with dummy effects on other layers, rather than requiring users to switch to artisan rendering mode. This integrates well with other effects and provides better usability while maintaining acceptable performance. Testing at quarter resolution can help evaluate performance characteristics.

*Tags: `aegp`, `params`, `ui`, `layer-checkout`, `render-loop`*

---

## How can you keep effect parameters synced between multiple locations when users apply keyframes and expressions?

This is identified as a challenging problem in plugin development. When trying to mirror parameters across multiple UI locations (such as keeping layer effect controls synced with a master adjustment layer control), synchronization becomes difficult if the user applies keyframes or expressions to either location. The conversation suggests using AEGP (After Effects General Plugin) calls or userchangedparam script callbacks as potential approaches, but the questioner notes that constantly mirroring these complex parameter states is not straightforward.

*Tags: `aegp`, `params`, `ui`, `scripting`*

---

## What is an effective architecture for creating interactive material systems with child plugins on layers controlled by a master adjustment layer?

One approach is to use an adjustment layer with a master plugin that loops over child layers and inspects their effects for specifically-named control instances. Rather than creating custom effects, you can use built-in Slider Control and Checkbox Control effects renamed in the UI, with the master plugin scanning for these named controls. The master effect reads their values to drive the material interactions. This keeps material-specific controls directly on the affected layers rather than centralizing all parameters in the master effect, improving UX. A more polished approach would involve creating dummy effects that don't render but provide a cleaner interface for users to specify layer properties without manually adding and renaming slider controls.

*Tags: `params`, `ui`, `aegp`, `mfr`*

---

## What alignment guarantees does PF_LayerDef::rowbytes provide in After Effects?

PF_LayerDef::rowbytes is guaranteed to always be a multiple of sizeof(PF_Pixel8), which is 4 bytes. 16-byte alignment is not always guaranteed. The rowbytes value is arbitrary and your code should be prepared to deal with any stride. Additionally, a buffer might be a sub-reference to another buffer, so you should not write into the bytes outside your image reference frame into the rowbytes gutter.

*Tags: `memory`, `params`, `aegp`, `output-rect`*

---

## How can you trigger parameter updates in Premiere Pro from a CEP script when PF_userChangedParam doesn't work?

In After Effects, using a hidden parameter updated by a script calling PF_userChangedParam works well for triggering updates. However, Premiere Pro does not consider script interactions as user actions, so this approach is not reactive. Alternative solutions include: monitoring external file dependencies for changes (though this is less reactive), or setting up a button to force manual refresh of the plugin parameters.

*Tags: `premiere`, `scripting`, `params`, `cep`*

---

## How should popup menu items be formatted in After Effects plugin parameter definitions?

When using PF_ADD_POPUPX or add_popup to define popup menu parameters, the separator (pipe character |) must be placed at the end of each line for all choices except the last one. For example: "choice1|" "choice2|" "Choice3". Ensure the parameter definition is properly structured with AEFX_CLR_STRUCT before the popup definition and verify there are no missing parameter definitions between the popup and any subsequent function calls.

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_POPUPX("Color", 3, 2,
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## Why were dropdown menus in a transition plugin malfunctioning during debugging?

The issue occurred when opening a Premiere project that already had the transition plugin applied. Opening a fresh instance of the plugin and applying it newly resolved the dropdown malfunction. The problem was related to the state of the plugin when loaded from an existing project versus a fresh application.

*Tags: `debugging`, `premiere`, `ui`, `params`*

---

## How should ARGB_8u be defined in globalSetup for proper colorspace handling?

When using ARGB_8u colorspace, it needs to be properly defined in the globalSetup function. The user inquired about the correct definition method and suggested checking colorspace settings and adding null checks for inoutworld (if inoutworld is null return err) to see if Premiere will redo the render.

*Tags: `premiere`, `params`, `debugging`, `render-loop`*

---

## What is the issue with the no_params_vary flag and how does it differ between After Effects and Premiere?

The no_params_vary flag was identified as the origin of a problem in plugin behavior. In After Effects, this flag's behavior can be replaced using MIX_GUI during the pre_render thread. However, in Premiere, the flag does nothing, indicating different parameter variation handling between the two applications.

*Tags: `premiere`, `params`, `mfr`, `render-loop`*

---

## How can I avoid noise artifacts when After Effects automatically converts 16/32-bit project input to 8-bit for my plugin?

Rather than relying on After Effects' automatic conversion, you should modify your plugin to accept 16 and 32 bits per channel (bpc) inputs and perform the conversion to 8-bit yourself. This approach provides several benefits: users will benefit from having the output remain in 16/32 bpc, you will avoid the warning sign next to your plugin in the UI, and you'll have full control over the conversion process to avoid dithering artifacts introduced by AE's automatic conversion.

*Tags: `params`, `output-rect`, `aegp`, `debugging`*

---

## How can I make a plugin support 8-bit, 16-bit, and 32-bit projects while sharing the same internal codebase without duplicating logic?

Use C++ templates to create multiple instances of your algorithm with different data types and range constants. This allows you to maintain a single shared codebase while handling different bit depths. Instead of letting After Effects automatically convert bit depths (which introduces color noise), you can manually handle the conversion within your plugin by instantiating template versions for 8-bit, 16-bit, and 32-bit processing.

*Tags: `mfr`, `params`, `build`, `reference`*

---

## How can I continuously update the ECW (Enhanced Custom UI) drawing while dragging a 2D point parameter?

ECW custom UIs receive idle calls on regular intervals only when the cursor is within their perimeter. To force continuous updates during parameter interactions, call PF_RefreshAllWindows during the interaction. This forces a redraw of the ECW, though it is resource-intensive. If the point parameter is supervised, this approach will keep the ARB viewer updated in real-time as the user drags the point.

*Tags: `ui`, `ecw`, `params`, `render-loop`*

---

## How can I hide an effect parameter from both the effect window and timeline window?

Set the AEGP_DynStreamFlag_HIDDEN flag during the PF_Cmd_UPDATE_PARAMS_UI command. This command is called when a project loads and the effect is shown for the first time, ensuring hidden parameters remain hidden across project loads and new effect applications. Note: There is a known bug in After Effects where owner-drawn portions of hidden parameters may still display space in certain circumstances (e.g., when applying a new effect instance), though the stream itself will be correctly hidden.

*Tags: `params`, `ui`, `aegp`, `debugging`*

---

## What is the correct command to call when hiding deprecated effect parameters?

Use PF_Cmd_UPDATE_PARAMS_UI to set the AEGP_DynStreamFlag_HIDDEN flag on parameters you want to hide. This ensures parameters are hidden when projects load and when effects are first displayed, making it the recommended approach over setting PF_PUI_NO_ECW_UI which only removes parameters from the effect window but not the timeline window.

*Tags: `params`, `ui`, `aegp`*

---

## How can you retrieve the string value of an arbitrary parameter in an After Effects effect using ExtendScript?

One approach is to use a hidden checkbox parameter that is supervised. From the JavaScript side, leave a message in global scope that the C side can read using AEGP_ExecuteScript(). Toggle the checkbox to trigger a USER_CHANGED_PARAM call. From the C side, use AEGP_ExecuteScript() to check for the message, read the string value from the arb data, and pass it back to JavaScript via AEGP_ExecuteScript(). The execution then returns to JavaScript where the string value is available in global scope. An alternative simpler approach is to use hidden UI elements and send the string to their controller name, then retrieve the property name and combine them in the JSX side. Note that there may be a limit on parameter name length, so avoid exceeding it to prevent crashes.

*Tags: `scripting`, `arb-data`, `aegp`, `params`*

---

## How do you handle Unicode characters in After Effects plugin parameter names and categories on macOS?

The name field is defined as 'A_char[32]', so you need to convert your string to UTF-8 and feed it to the param macro. Make sure the length of the resulting UTF-8 string is no more than 31 characters long (including multibyte characters), as the last character must be reserved for null termination. One successful approach mentioned was using ConvertUTF8ToGBK() in a UTF-8 encoded .cpp file when working with XCode.

*Tags: `macos`, `params`, `unicode`, `build`, `xcode`*

---

## How should you detect parameter changes that occur from keyframe timeline scrubbing rather than direct user interaction?

Parameter synchronization across user interaction, timeline scrubbing, and rendering presents challenges. For detecting changes during timeline scrubbing, you can use PF_Cmd_UPDATE_PARAMS_UI. However, a more robust approach is to use expressions instead of keyframes on dependent parameters. You can set expressions programmatically during UPDATE_PARAMS_UI using AEGP_SetExpression, which handles all three scenarios (user interaction, timeline scrubbing, and rendering) consistently.

*Tags: `params`, `aegp`, `render-loop`*

---

## How can you retrieve the original parameter value with full precision before downsampling is applied by in_data->downsample_x and downsample_y?

Use AEGP_GetNewStreamValue instead of the plain checkout method to retrieve parameter values. This approach preserves the original precision of the parameter value before any downsampling by in_data->downsample_x and in_data->downsample_y is applied, which is critical for maintaining double precision in 3D point parameters.

*Tags: `params`, `aegp`, `memory`*

---

## How can you make a parameter read-only and prevent manual editing while still allowing programmatic keyframe changes?

Experiment with PF_PUI_DISABLED flag, which should prevent users from manually changing the parameter but may still allow expression connections. Alternatively, use PF_PUI_STD_CONTROL_ONLY to make the parameter non-keyframable, though this may affect expression connectivity. The expression-based approach mentioned above (using AEGP_SetExpression during UPDATE_PARAMS_UI) provides a workaround for creating dependent, programmatically-controlled parameters.

*Tags: `params`, `ui`*

---

## What is the correct first argument to pass to transform_world and other suite callbacks in After Effects plugins?

According to community expert Shachar Carmi, the Adobe documentation is incorrect about the first argument. Instead of passing in_data as documented, you should pass NULL or in_data->effect_ref. Passing NULL works because many suite callbacks accept null instead of effect_ref, and the PF_INTERRUPT macro internally uses effect_ref to make interrupt checks. The incorrect documentation may be legacy from before CC2015 when a separate rendering thread was introduced.

*Tags: `sdk`, `transform_world`, `params`, `debugging`, `reference`*

---

## Why does transform_world give better results when entering parameter values numerically versus dragging sliders?

When working with transform_world, numeric input (typing a value and pressing enter) produces better results than slider dragging. This is theorized to be related to how After Effects halts the transformation process during interactive slider adjustments to provide a smoother user experience, though this behavior may be legacy code from before CC2015's separate rendering thread was introduced.

*Tags: `transform_world`, `params`, `ui`, `render-loop`*

---

## How can two different plugin types share arbitrary data through their arb parameters?

AE blocks direct expression connections between arb parameters of different plugin types, as it assumes different effects cannot be guaranteed to have the same structure. However, you can read arb values from other effects using AEGP_GetNewStreamValue on the C/C++ side. To reference another plugin instance, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH and effect index. For consistent identification across sessions, store an ID in sequence data and access it via AEGP_EffectCallGeneric, though this has limitations with duplicated effects. Alternatively, rename an invisible parameter, though this won't survive project reloads.

*Tags: `arb-data`, `aegp`, `params`, `cross-plugin`*

---

## What is the minimal code structure needed to implement smart render in an After Effects plugin?

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

*Tags: `smart-render`, `aegp`, `reference`, `params`*

---

## Why doesn't AEGP_EffectCallGeneric() trigger PF_Cmd_COMPLETELY_GENERAL on Mac when trying to identify a specific effect instance?

According to community expert shachar carmi, AEGP_EffectCallGeneric() has limitations and cannot reliably call from one effect instance to another instance of the same type. The expert suggests that if you need inter-instance communication, you should create a separate AEGP plugin to handle these calls during idle processing, rather than calling directly between effect instances of the same type. As an alternative for identification that doesn't need to survive save/load, you can change a hidden parameter's name to be the identifier and read it via the stream suite.

```cpp
// Instead of calling between same-type effects, use a separate AEGP
// Or use hidden parameter name as identifier via stream suite
AEGP_SetStreamName()
```

*Tags: `aegp`, `params`, `debugging`, `cross-platform`, `macos`, `windows`*

---

## How can an effect instance identify itself given an AEGP_EffectRefH handle?

Only an effect instance has access to its own sequence data handle when AE passes it during calls, and AE may move that memory between calls. For identifying specific instances, experts recommend using a mechanism where each effect instance stores a unique identifier (like a 64-bit ID generated at instantiation). Since direct inter-instance communication via AEGP_EffectCallGeneric() is problematic for same-type effects, alternative approaches include: (1) using a separate AEGP plugin to mediate calls, or (2) for non-persistent identification, changing a hidden parameter's name to encode the identifier and reading it via the stream suite.

*Tags: `aegp`, `sequence-data`, `params`, `arb-data`*

---

## How do you apply transformation matrices with a custom pivot point instead of the upper left corner?

To change the pivot/origin point from the upper left corner to a custom location (like center), you need to use matrix multiplication of three separate matrices: first create a matrix for the anchor point (using negative values of the offset), multiply it by the rotation matrix, then multiply the result by the position matrix. You cannot place all transformations in a single matrix placement.

```cpp
matrix.mat[0][0] = cosf(angle) * scale;
matrix.mat[0][1] = sinf(angle) * scale;
matrix.mat[1][0] = -sinf(angle) * scale;
matrix.mat[1][1] = cosf(angle) * scale;
matrix.mat[2][0] = position_x;
matrix.mat[2][1] = position_y;
matrix.mat[2][2] = 1;
```

*Tags: `params`, `matrix-math`, `transformation`*

---

## How can I set a parameter value during the paramSetup function?

ParamSetup is called once per session per effect type, not once per instance, so all instances would get the same value. Instead, use sequence data setup (PF_Cmd_SEQUENCE_RESETUP), which is triggered once per instance when created. However, you cannot directly set parameters during sequence setup because the call doesn't have a specific instance associated with it and the params array contains junk data. The correct approach is to set a default value during the PF_ADD_FLOAT_SLIDER call in paramSetup, or set a flag in sequence data indicating a new instance, then check that flag during idle_hook or UPDATE_PARAMS_UI to make the change there.

*Tags: `params`, `sequence-data`, `aegp`*

---

## Can I modify parameters during PF_Cmd_SEQUENCE_RESETUP?

No, you cannot modify parameters during sequence setup calls. Sequence setup typically occurs without a specific instance context, meaning the passed params array contains invalid data and attempting to acquire an AEGP_EffectRef will crash. The workaround is to set a flag in the new sequence data marking it as a brand new instance, then check this flag during idle_hook or UPDATE_PARAMS_UI to apply parameter changes there, remembering to clear the flag afterward.

*Tags: `sequence-data`, `params`, `aegp`*

---

## How do you properly apply CANNOT_TIME_VARY and COLLAPSE_TWIRLY flags to arbitrary parameters in After Effects plugins?

These flags should be placed in the param flags argument (one position before the UI flags argument), not in the PF_PUI_CONTROL flags. The param flags and param UI flags are separate arguments in PF_ADD_ARBITRARY2, and CANNOT_TIME_VARY and COLLAPSE_TWIRLY belong in the param flags position, not the UI flags position where PF_PUI_CONTROL resides.

```cpp
PF_ADD_ARBITRARY2("Custom Parameter",
  UI_BOX_WIDTH,
  UI_BOX_HEIGHT,
  PF_ParamFlag_CANNOT_TIME_VARY | PF_ParamFlag_COLLAPSE_TWIRLY,
  PF_PUI_CONTROL | PF_PUI_DONT_ERASE_CONTROL,
  def.u.arb_d.dephault,
  PARAM_DISK_ID,
  ARB_REFCON);
```

*Tags: `params`, `arb-data`, `ui`, `sdk`*

---

## Is it possible to hot reload After Effects plugins without restarting the application?

No, hot reloading is not practical for native C++ After Effects plugins. Each time you build C++ code, you must restart After Effects because the application calls parameter setup only once per session when the plugin is first loaded, so hot reloading cannot trigger re-initialization of parameters. However, ExtendScript and CEP plugins can be reloaded without restarting the entire AE application. Debugging through an IDE like Microsoft Visual Studio or Apple Xcode can make the restart cycle faster than manually launching AE.

*Tags: `params`, `build`, `debugging`, `sdk`, `scripting`*

---

## What needs to be unique when creating multiple arbitrary parameters in an After Effects plugin?

When creating multiple arbitrary parameters, each parameter needs its own unique parameter ID (like CUSTOMUI_DISK_ID). The ARB_REFCON can be anything you want—it doesn't need to be unique; After Effects doesn't care about its content. What matters is how you use it: you can point it to a function handler or class instance specific to that parameter. Each arbitrary parameter must have its own allocated handle for the default value, even if the default values are the same. The PF_CustomUIInfo registration only needs to be done once per PARAM_SETUP call, not per parameter.

```cpp
switch(param_index) {
  case ARB_1_INDEX:
    ArbHandler1();
    break;
  case ARB_2_INDEX:
    ArbHandler2();
    break;
}
```

*Tags: `arb-data`, `params`, `ui`, `sdk`*

---

## How should you handle multiple arbitrary parameters in the HandleArbitrary function?

Rather than using a switch statement on param_index for each arbitrary parameter, a more elegant C++ approach is to use ARB_REFCON as a pointer to the appropriate handler (either a function pointer or class instance). Each arbitrary parameter gets its own ARB_REFCON pointing to its specific handler. This eliminates the need for large switch statements and allows you to use a class-based architecture where a generic arb base class handles common operations and specific subclasses override methods as needed. This approach scales much better for plugins with many arbitrary parameters.

*Tags: `arb-data`, `params`, `sdk`*

---

## How can I access post-processed shape path vertices (with effects like offset, zig zag, or wiggle applied) using the After Effects SDK?

Unfortunately, After Effects does not provide direct access to post-processed paths through the SDK. You can only read the parameter streams directly and then manually emulate the processed path behavior by implementing the effects yourself.

*Tags: `aegp`, `sdk`, `shape-layers`, `params`*

---

## How should I store time-dependent curve data for a custom parameter and ensure the UI updates as the user draws?

Use an arbitrary parameter (PF_ADD_ARBITRARY2) to store your data. The workflow is: user interacts with UI → data is stored in a LOCAL array → local array is serialized and stored in the arb parameter → AE automatically invalidates cached frames and re-renders. Before displaying the UI, read the value from the arb param, deserialize it into your local array, and draw accordingly. This also ensures undo/redo operations reflect correctly in the UI. You don't need to convert data to a string; serialize it into one continuous block of memory using whatever method works for your data type.

*Tags: `params`, `ui`, `arb-data`, `caching`, `render-loop`*

---

## Should I use global variables to store custom parameter data that needs to be accessed from multiple functions?

No. Instead of using global variables, store user input in a LOCAL array during UI interactions (like DrawEvent), then serialize that local array into an arbitrary parameter. This allows the data to persist, be invalidated properly by AE, and trigger re-renders automatically without relying on global state.

*Tags: `params`, `arb-data`, `ui`, `memory`*

---

## What is the best practice for spawning a window with text input UI from effect parameters?

There are several approaches: (1) Use AEGP_ExecuteScript to launch a JavaScript prompt for easy cross-OS compatibility—it takes text input from the user and you can check the returned value to see if the user hit 'ok' or 'cancel'. (2) Use OS-level windows for more advanced UI, which works on both macOS and Windows but requires more documentation review. (3) Store the text data using either sequence data or an arbitrary (arb) parameter—the arb route is preferred as it allows undo/redo. You can add a custom UI to the arb parameter that both displays the current string value and is clickable, or use a simple button parameter to trigger the window while keeping the arb parameter hidden.

*Tags: `ui`, `params`, `arb-data`, `cross-platform`, `aegp`, `scripting`*

---

## How should text input be stored with a layer when using arbitrary parameters?

Arbitrary parameters can store the text string and are preferred over sequence data because they support undo/redo operations. The arb parameter itself can be hidden from the user if desired. You can implement a custom UI on the arb parameter that displays the current text value and is clickable, or alternatively use a separate button parameter to trigger the input dialog while the arb parameter stores the data invisibly.

*Tags: `params`, `arb-data`, `ui`, `layer-checkout`*

---

## How do you properly update UI parameter values using PF_Cmd_UPDATE_PARAMS_UI in After Effects plugins?

During UPDATE_PARAMS_UI, you should only change parameter appearance (hidden, disabled, etc.) and not values. To change values, set proper defaults during PARAM_SETUP instead. When updating parameters, do not modify values and flags directly on the original params array. Instead, make a copy of the param struct, modify the copy, and pass it back to PF_UpdateParamUI. Also ensure you set out_data->out_flags |= PF_OutFlag_REFRESH_UI before returning. Refer to the Supervisor SDK sample project for correct implementation details.

```cpp
static PF_Err UpdateUI(PF_InData* in_data, PF_OutData* out_data, PF_ParamDef* params[]) {
  PF_Err err = PF_Err_NONE;
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  // Make a copy, modify the copy, then use PF_UpdateParamUI
  ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref, SKELETON_COLOR, params[SKELETON_COLOR]));
  out_data->out_flags |= PF_OutFlag_REFRESH_UI;
  return err;
}
```

*Tags: `params`, `ui`, `pipl`, `reference`*

---

## What is the Supervisor SDK sample project and what does it demonstrate?

The Supervisor SDK sample project is an official Adobe After Effects SDK example that demonstrates proper parameter supervision and UI updating. While somewhat convoluted, the first few lines of its UpdateParameterUI() function show the correct implementation for using PF_UpdateParamUI with parameter copies and proper flag handling.

*Tags: `sdk`, `params`, `ui`, `reference`, `open-source`*

---

## How can you enable an alpha channel in a COLOR parameter for user editing?

The native COLOR parameter in After Effects does not support alpha channel editing in the color picker. According to community experts, you cannot enable this directly. However, you have two workarounds: (1) Use PF_AppColorPickerDialog to access the system color dialog, though it typically won't have an alpha component either, or (2) Implement your own custom color picking dialog and launch it via param supervision or custom UI. Alternatively, consider adding a separate opacity parameter instead of trying to include alpha in the color picker.

*Tags: `params`, `ui`, `sdk`, `color-picker`*

---

## How are colors represented in After Effects expressions?

In After Effects expressions, colors use a four-number array format: [red, green, blue, alpha]. All four values must be provided in expressions, or you can use color space conversions to generate the values. While the color data type supports four channels internally, not all color parameters expose the alpha channel for user editing.

```cpp
[red, green, blue, alpha]
```

*Tags: `params`, `expressions`, `scripting`*

---

## How do you set keyframes on effect parameters during plugin initialization?

Setting keyframes during GlobalSetup won't work because the effect_ref is garbage at that point since there's no effect instance yet. Instead, set keyframes on the first call to UPDATE_PARAMS_UI. Alternatively, if you're adding the effect via script, set the keyframes through the script instead of the plugin code. The correct approach involves using the AEGP KeyframeSuite functions (AEGP_StartAddKeyframes, AEGP_AddKeyframes, AEGP_SetAddKeyframe, AEGP_EndAddKeyframes) with a valid effect reference obtained after the effect instance is created.

```cpp
ERR(suites.UtilitySuite5()->AEGP_RegisterWithAEGP(NULL, pluginName, &pluginID));
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(pluginID, in_data->effect_ref, &effectPH));
ERR(suites.StreamSuite5()->AEGP_GetNewEffectStreamByIndex(pluginID, effectPH, 25, &streamH));
ERR(suites.StreamSuite5()->AEGP_GetNewStreamValue(pluginID, streamH, AEGP_LTimeMode_CompTime, 0, TRUE, &valueP));
valueP.val.one_d = 20;
ERR(suites.KeyframeSuite4()->AEGP_StartAddKeyframes(streamH, &akH));
ERR(suites.KeyframeSuite4()->AEGP_AddKeyframes(akH, AEGP_LTimeMode_CompTime, 0, &indexPL));
ERR(suites.KeyframeSuite4()->AEGP_SetAddKeyframe(akH, indexPL, &valueP));
ERR(suites.KeyframeSuite4()->AEGP_EndAddKeyframes(true, akH));
```

*Tags: `params`, `aegp`, `scripting`, `sdk`*

---

## How do I properly flatten sequence data so it persists when reopening a project?

To flatten sequence data, set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag during global setup. This will cause your plugin to receive PF_Cmd_SEQUENCE_FLATTEN calls where you can perform the flattening. Do not attempt to flatten data in SequenceSetdown, as that callback is only for destructing and freeing data, not for saving it.

*Tags: `sequence-data`, `params`, `sdk`, `aegp`*

---

## How can I prevent users from adding multiple instances of my effect to the same layer?

You can identify your effect uniquely by storing its AEGP_InstalledEffectKey during global setup using AEGP_GetNumInstalledEffects, AEGP_GetNextInstalledEffect, and AEGP_GetEffectMatchName. Then during UPDATE_PARAMS_UI, scan the effect's layer and check each effect's install key using AEGP_GetInstalledKeyFromLayerEffect to detect multiple instances. Once detected, you can alert the user, delete the redundant instance, disable it, or skip rendering for redundant copies by passing input to output. Note that AEGP_DisableCommand cannot remove effects from the menu, and users may still duplicate or copy/paste the effect.

*Tags: `aegp`, `params`, `ui`, `plugin-development`*

---

## What is the issue with storing effect identification data in sequence data across render calls?

Sequence data stored during effect initialization may not persist consistently across different command calls like PF_Cmd_Render. Using AEGP_InstalledEffectKey instead provides a more reliable way to uniquely identify effect instances across the plugin lifecycle without relying on sequence data storage.

*Tags: `sequence-data`, `params`, `aegp`*

---

## Can you hide the second dropdown in PF_ADD_LAYER layer selector parameters?

No, the second dropdown (which allows selection between "Source", "Masks", or "Effects & Masks") cannot be hidden. That is the standard appearance of the layer selector in current versions of the After Effects SDK.

*Tags: `params`, `ui`, `sdk`, `mfr`*

---

## Is it possible to add or remove effect parameters after the PF_Cmd_PARAM_SETUP event?

No, you cannot add or remove parameters at any time other than the initial setup. The number and count of parameters are fixed. However, you can hide and show parameters dynamically based on user input. An alternative workaround used by plugins like Plexus is to add additional effects to the layer that contain sets of controls, and have the main effect read their parameters during rendering.

*Tags: `params`, `pipl`, `effect_api`*

---

## Can you change the name of an effect parameter after initial setup?

Yes, you can change a parameter's name by getting its streamRef and changing the stream's name. However, this change will not persist between sessions—when you load a project, the default names will be used again. You can work around this by renaming parameters again during the sequence_resetup event. Note that this approach has been reported to cause issues in Premiere Pro.

*Tags: `params`, `pipl`, `premiere`*

---

## How do you convert an A_Time value to a floating-point seconds value in the After Effects SDK?

A_Time is a rational type with two components: value and scale. To convert to seconds as an A_FpLong, divide the value by the scale: `A_FpLong currentTime = currT.value / currT.scale;`. Note that A_Time values are rationals and may not map exactly to floating point, potentially causing off-by-one frame issues, so for precision it's better to work directly with rational time operations.

```cpp
ERR(suites.ItemSuite6()->AEGP_GetItemCurrentTime(itemH, &currT));
A_FpLong currentTime = currT.value / currT.scale;
```

*Tags: `aegp`, `params`, `reference`*

---

## What is example code for converting seconds to an A_Time value in Objective-C?

Here is sample Objective-C code that converts from seconds to A_Time by calculating a timescale based on the composition's framerate: Create a timescale by multiplying framerate by 100, then construct the A_Time with `A_Time time = {seconds * timescale, timescale};`. You can retrieve the framerate using `AEGP_GetCompFramerate()` from the CompSuite.

```cpp
+ (A_Time) timeFromSeconds:(CFTimeInterval)seconds {
  AEGP_CompH comp = [self activeComposition];
  A_FpLong framerate = [self framerateFromComp:comp];
  A_FpLong timescale = framerate * 100;
  A_Time time = {seconds * timescale, timescale};
  return time;
}
```

*Tags: `aegp`, `params`, `macos`*

---

## How can a C++ effect plugin modify transform parameters of other layers in a project?

Use the StreamSuite to affect any parameter (which are internally streams) in the project. The 'project dumper' sample project demonstrates how to access streams. However, since CC2015, you cannot change the project during a render call. To modify project parameters, use UI events (anything other than render and pre-render calls) when you can modify the project as needed.

*Tags: `aegp`, `params`, `render-loop`, `streaming`, `sdk`*

---

## How can you create independent keyframe streams for individual parameters in an arbitrary data effect instead of concatenating all parameters into one keyframe stream?

Adobe deliberately designs some effects with concatenated keyframes (all parameters share one keyframe stream) and others with individual controls (each parameter has independent keyframes). The Levels effect has two versions shipped with After Effects: the standard Levels (concatenated) and Levels (Individual Controls) (independent parameters). This is a design choice rather than a technical constraint. If you need independent animations and keyframing, you can either use the individual controls variant pattern or calculate effects on-the-fly during the Render event without storing interpolation data in ArbData, similar to how Levels works without relying on ArbData for histogram display.

*Tags: `arb-data`, `params`, `ui`, `animation`, `plugin-design`*

---

## Why does the Hue/Saturation effect concatenate all parameters into a single keyframe stream instead of allowing independent parameter animation like Levels (Individual Controls)?

The Hue/Saturation effect uses USER_CHANGED_PARAM to create keys on arbitrary parameters, which results in all slider changes being keyed together rather than independently. This design choice means you cannot animate individual sliders (like Lightness) separately if another slider (like Saturation) is already animated, as all changes are stored in a single ArbData keyframe stream. The Levels (Individual Controls) variant demonstrates the alternative approach where parameters are independent, suggesting that Hue/Saturation's design prioritizes a different workflow, though the constraint could be overcome by redesigning parameter architecture to use separate animation streams.

*Tags: `arb-data`, `params`, `animation`, `ui`, `plugin-design`*

---

## How can I prevent the timeline properties from opening when setting AEGP_DynStreamFlag_HIDDEN?

According to shachar carmi, there is no way to expose a parameter in the ECW (Effect Control Window) without exposing it in the timeline. As a workaround, try using PF_ParamFlag_SKIP_REVEAL_WHEN_UNHIDDEN flag, which may prevent the parameter from being exposed in both places. Alternatively, PF_UpdateParamUI differs from AEGP_SetDynamicStreamFlag in its behaviors and may produce more suitable results. The 'Supervisor' SDK sample project demonstrates how PF_UpdateParamUI is used.

*Tags: `aegp`, `params`, `ui`, `sdk`*

---

## What is a good SDK sample project to learn how to manage dynamic parameter visibility?

The 'Supervisor' SDK sample project is recommended as a reference for understanding how to use PF_UpdateParamUI for managing parameter visibility and behavior in After Effects plugins. It demonstrates techniques for controlling whether parameters appear in the Effect Control Window and timeline.

*Tags: `sdk`, `reference`, `params`, `ui`*

---

## How can you rename an effect instance programmatically using AEGP?

To rename an effect instance, get the effect's param 1 reference stream, then retrieve its parent stream using AEGP_GetNewParentStreamRef. This parent stream is the effect's stream, which you can then rename using AEGP_SetStreamName.

*Tags: `aegp`, `params`, `scripting`*

---

## How can I hide a custom arbitrary parameter from the timeline while keeping it visible in the Effect Controls Window?

If the custom arbitrary parameter is purely for UI purposes and doesn't need to vary over time, use PF_ADD_NULL instead of creating an arbitrary parameter. This approach avoids the <error> display in the timeline while still allowing UI controls in the ECW. The HistoGrid example demonstrates this pattern effectively.

```cpp
PF_ADD_NULL("parameter_name");
```

*Tags: `ui`, `params`, `arb-data`, `reference`*

---

## What causes a custom arbitrary parameter to display as <error> in the timeline?

The <error> message occurs when an arbitrary parameter doesn't have a proper name assigned to it. Ensure that the custom arb param is given a descriptive name string in the parameter definition.

```cpp
PF_ADD_ARBITRARY2("Banner Name",
  A_long(PF_FpLong(BANNER_WIDTH) * 0.5),
  A_long(PF_FpLong(BANNER_HEIGHT) * 1.0),
  PF_ParamFlag_CANNOT_TIME_VARY,
  PF_PUI_CONTROL,
  def.u.arb_d.dephault,
  BANNER_ID,
  0);
```

*Tags: `params`, `arb-data`, `ui`, `debugging`*

---

## Can invisible effect parameters be accessed from expressions in After Effects?

Invisible parameters marked with PF_PUI_INVISIBLE cannot be accessed by name or match name in expressions, but they can be accessed by their index number. This was confirmed through experimentation where accessing by index worked successfully, while name and match name references returned property missing errors.

*Tags: `expressions`, `params`, `scripting`, `aegp`*

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

## How do you get the composition width and height in the PF_Cmd_USER_CHANGED_PARAM function?

The in_data->width and height fields denote your effect's layer source item full resolution size (unmasked, unchanged by effects), not the composition size. If you need the actual composition dimensions, you need to get the effect's layer source item and retrieve the project item's dimensions from it. The output pointer will be NULL in PF_Cmd_USER_CHANGED_PARAM, unlike in SmartRender where you can use checkout_output.

*Tags: `params`, `aegp`, `reference`*

---

## Why isn't sequence_data updated when modified in UpdateParameterUI and passed to SequenceResetup?

The issue is that UpdateParameterUI is not the correct place to modify sequence_data with FORCE_RERENDER. According to Adobe documentation, FORCE_RERENDER works during PF_Cmd_USER_CHANGED_PARAM and also in CLICK and DRAG events (if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented). Modifications to sequence_data should be made in PF_Cmd_USER_CHANGED_PARAM instead of UpdateParameterUI to ensure the data is properly synchronized between UI and render threads.

*Tags: `sequence-data`, `aegp`, `threading`, `params`, `ui`*

---

## What should the refconPV value be when creating arbitrary data parameters in After Effects plugins?

The refconPV is a pointer to whatever you'd like, and that pointer is passed back to the plugin during arbitrary data calls for that parameter. If you have different handler classes for each arbitrary parameter, you can pass a pointer to an instance of that class. If unused, you can pass null. The value 0xDEADBEEFDEADBEEF seen in examples is just programmer's leetspeak and not significant. This allows you to maintain separate state or handlers for multiple arbitrary data parameters in the same plugin.

```cpp
#define ARB_REFCON (void*)0xDEADBEEFDEADBEEF
PF_ADD_ARBITRARY2( "Color Grid",
UI_GRID_WIDTH,
UI_GRID_HEIGHT,
PF_PUI_CONTROL | PF_PUI_DONT_ERASE_CONTROL,
def.u.arb_d.dephault,
COLOR_GRID_UI,
ARB_REFCON);
```

*Tags: `arb-data`, `params`, `ui`, `reference`*

---

## Should I use PF_Param_FLOAT_SLIDER instead of the deprecated PF_Param_SLIDER for slider parameters in After Effects plugins?

According to the Adobe AE SDK documentation, PF_Param_SLIDER and PF_Param_FIX_SLIDER are deprecated in favor of PF_Param_FLOAT_SLIDER. However, PF_Param_SLIDER still works and hasn't been completely removed from the API. If you need to force integer-only sliders, PF_Param_SLIDER remains a viable option. For PF_Param_FLOAT_SLIDER, you can set precision to 0 and round values in param supervision, though this won't prevent keyframe interpolation from creating fractional values. Community experts suggest that PF_Param_SLIDER is still safe to use if integer sliders are a requirement.

*Tags: `params`, `ui`, `reference`*

---

## How can I create integer-only sliders in After Effects plugins when using PF_Param_FLOAT_SLIDER?

There is currently no built-in way to force PF_Param_FLOAT_SLIDER to accept only integer values. The recommended workaround is to implement param supervision that rounds the value to an integer when the parameter changes. However, this approach won't prevent keyframe interpolation from creating fractional values between keyframes. Alternatively, use PF_Param_SLIDER if you specifically need integer-only behavior, as it inherently enforces integer values.

*Tags: `params`, `ui`*

---

## How can I localize parameter labels in After Effects plugins to display in different languages like Chinese?

Adobe provides official documentation on localization for After Effects plugins. The localization guide is available in the official After Effects SDK documentation at https://ae-plugins.docsforadobe.dev/intro/localization.html, which covers how to implement parameter label localization for different languages.

*Tags: `params`, `ui`, `localization`, `reference`, `sdk`*

---

## How do you trigger PF_Cmd_UPDATE_PARAMS_UI to dynamically show/hide parameters when user input changes?

To trigger PF_Cmd_UPDATE_PARAMS_UI, you need to set two flags: (1) PF_OutFlag_SEND_UPDATE_PARAMS_UI in global setup, and (2) PF_ParamFlag_SUPERVISE on the parameter you wish to supervise. Without PF_OutFlag_SEND_UPDATE_PARAMS_UI set during global setup, the UPDATE_PARAMS_UI command will not be triggered even if sequence_data changes. Additionally, use PF_UpdateParamUI and the Stream/DynamicStream suites to modify parameter visibility at runtime, and set PF_OutFlag_REFRESH_UI in out_data->out_flags to refresh the UI.

```cpp
// In global setup:
out_data->out_flags = out_data->out_flags | PF_OutFlag_SEND_UPDATE_PARAMS_UI;

// In param setup:
def.flags = PF_ParamFlag_SUPERVISE;

// In PF_Cmd_USER_CHANGED_PARAM:
out_data->out_flags = out_data->out_flags | PF_OutFlag_FORCE_RERENDER | PF_OutFlag_REFRESH_UI;
```

*Tags: `params`, `ui`, `aegp`, `sequence-data`*

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

## Is it necessary to handle PF_Cmd_UPDATE_PARAMS_UI if PF_Cmd_USER_CHANGED_PARAM is already implemented?

PF_Cmd_USER_CHANGED_PARAM is called only on user interactions, while PF_Cmd_UPDATE_PARAMS_UI is called at other times such as when a new instance of an effect is created or when an instance is loaded with a project. If the invocation times of PF_Cmd_UPDATE_PARAMS_UI are not relevant to your plugin's functionality, then there is no need to implement it.

*Tags: `params`, `ui`, `aegp`*

---

## How can a C++ plugin read the time slider position when its own UI window is displayed and plugin parameters haven't changed?

Use the AEGP suite to query the current time directly instead of relying on in_data->current_time, which only updates when EffectMain commands are triggered. Call AEGP_GetActiveItem() to get the active item, then use AEGP_GetItemCurrentTime() to retrieve the current time value. Alternatively, use an idle hook on the plugin side combined with a timer on the window side to synchronize playback between the AE timeline and the custom win32 window.

```cpp
A_long time_ae(PF_InData * in_data)
{
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  A_Time time;
  AEGP_ItemH myitem;
  suites.ItemSuite8()->AEGP_GetActiveItem(&myitem);
  suites.ItemSuite8()->AEGP_GetItemCurrentTime(myitem, &time);
  return time.value;
}
```

*Tags: `aegp`, `ui`, `params`, `windows`, `debugging`*

---

## How can you trigger cache purging from within a C++ plugin when a user changes a parameter?

You can use JavaScript execution via the C API to trigger the cache purge command. Execute `app.executeCommand(2370)` through the ExecuteScript function. However, this approach is strongly discouraged because purging RAM forces users to re-cache all other timelines or portions of the current timeline. Instead, ensure that parameter changes properly invalidate cached frames automatically, or use an invisible parameter change to force invalidation if needed.

```cpp
app.executeCommand(2370);
```

*Tags: `params`, `scripting`, `caching`, `memory`*

---

## How can I update plugin parameters when the effect is first applied to a layer?

Parameter value changes made directly during UPDATE_PARAMS_UI are ignored by After Effects. Instead, use SetStreamValue to modify parameter values during initialization, which will persist the changes. While not considered best practice, this is the reliable method to initialize parameters on first application of the effect.

```cpp
params[EM_TOPLEFT]->u.td.x_value = FLOAT2FIX(globP->x0);
params[EM_TOPLEFT]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `sdk`, `update`*

---

## Will changing non-PUI_ONLY parameters trigger an automatic re-render in After Effects?

Yes, modifying any parameter that is not marked as PUI_ONLY should automatically trigger a re-render. If a re-render is not occurring after parameter changes via SetStreamValue, verify that the modified parameters are not PUI_ONLY flags.

*Tags: `params`, `render-loop`, `smart-render`*

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

## How do you properly handle undo/redo with sequence data and arbitrary data in After Effects plugins?

Sequence data does not participate in undo/redo, which makes it problematic as a base for calculations that depend on parameter changes. The recommended approach is to store a state identifier in both arbitrary data (arb data) and sequence data. When using sequence data, compare the state identifier in the arb to the one in the sequence data—if they mismatch, regenerate the sequence data and store the corresponding state identifier. Sequence data is best used only when other storage methods don't meet needs, such as for data that should survive a reset button, data that shouldn't trigger undo entries, or temporary data passed between modules.

*Tags: `sequence-data`, `arb-data`, `params`, `debugging`*

---

## What are the appropriate use cases for sequence data in After Effects plugin development?

Sequence data has limited but specific use cases: (1) flagging new instances during unique sequence setup calls, (2) storing data that survives the reset button (unlike arb data), (3) storing data that should not undo/redo or trigger re-renders, and (4) maintaining session data without bloating the project file. It should not be used as a base for parameter-dependent calculations or processed image data, as there is no reliable way to tell when param values have changed and different frames may be processed simultaneously.

*Tags: `sequence-data`, `arb-data`, `params`*

---

## How should arbitrary data reallocation be handled when it depends on non-animatable parameters like 'rows' and 'cols' in a C++ effect plugin?

Arbitrary parameter handlers in After Effects are stateless functions that operate without access to a specific effect instance, effect reference, or sequence data. Instead of trying to fetch parameter values during HandleArbitrary() calls (where effect_ref is usually null and checkout/checkin rarely work), you should reallocate and store allocation-related metadata like 'rows' and 'cols' in your arbitrary data structure itself during UserChangedParam() or the render function. This way, HandleArbitrary() can process the data independently without querying system or instance state. The render function should fetch raw arbitrary data and other relevant pieces available during render time, then process them there rather than during arbitrary parameter handling calls.

*Tags: `arb-data`, `params`, `skeleton`, `memory`*

---

## How can I detect when my effect plugin is first applied to a layer in After Effects?

Detect a new effect application by setting a flag in sequence data during the SEQUENCE_SETUP call, which is the only call unique to a new application. At that time, you cannot get a layer handle because AE hasn't associated the instance yet. Wait for the UPDATE_PARAMS_UI call (after the flag is set) to get the layer handle. Then iterate through effects on the layer and use AEGP_GetInstalledKeyFromLayerEffect() to compare InstallKeys with your effect's InstallKey to identify duplicates.

*Tags: `aegp`, `sequence-data`, `params`, `plugin`*

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

## How do I draw custom UI in the title area of a parameter to position it as far left as possible?

When defining the parameter, set PF_PUI_TOPIC instead of PF_PUI_CONTROL. You'll need to respond to PF_EA_PARAM_TITLE instead of PF_EA_CONTROL during the event call. If you need UI in both the title area and control area, you can define both for the same parameter, but note that the two areas won't join into one continuous segment.

*Tags: `ui`, `pipl`, `params`*

---

## How can I create a plugin that adds image layers and regenerates them when properties change?

This is possible by combining an AEGP panel plugin with an effect plugin. The AEGP panel (implemented similar to the SDK's 'Panelator' sample) creates and manages layers with an inspector window for user controls. The effect plugin handles rendering and stores configuration data using either hidden parameters (which can be read/written by the AEGP window) or sequence data (a memory chunk that stores arbitrary data without UI representation, undo stack entries, or reset behavior). The effect plugin can regenerate images whenever properties change, with the decisions made in the panel interface driving the effect's rendering behavior.

*Tags: `aegp`, `params`, `sequence-data`, `ui`, `reference`*

---

## How can an effect plugin determine what type of layer it is attached to?

Use AEGP_GetLayerObjectType() to determine the layer type. This function returns one of several values: AEGP_ObjectType_NONE (-1), AEGP_ObjectType_AV (includes all pre-AE 5.0 layer types including audio, video sources, and adjustment layers), AEGP_ObjectType_LIGHT, AEGP_ObjectType_CAMERA, AEGP_ObjectType_TEXT, or AEGP_ObjectType_VECTOR. To access the layer handle from within an effect, use AEGP_GetEffectLayer().

```cpp
AEGP_ObjectType_NONE = -1,
AEGP_ObjectType_AV, /* Includes all pre-AE 5.0 layer types (audio or video source, including adjustment layers) */
AEGP_ObjectType_LIGHT,
AEGP_ObjectType_CAMERA,
AEGP_ObjectType_TEXT,
AEGP_ObjectType_VECTOR,
```

*Tags: `aegp`, `params`, `reference`*

---

## How does the PF_FloatMatrix work and how do you use it with transform_world() to transform a PF_EffectWorld?

PF_FloatMatrix is a standard 3x3 transformation matrix used in graphics programming. It can perform translation, rotation, scaling, and skewing transformations all in one concatenated matrix. The matrix indices control different transformation properties: the first row typically contains x-translation and scaling/rotation components, while the third row contains y-translation and scaling/rotation components. When applying rotations using a matrix, you place sin and cos (and negative values) in specific positions following standard 2D/3D graphics conventions. Important note: After Effects uses row-based matrices, which is the opposite of the OpenGL standard, so you need to swap the axis accordingly. The transform_world() function applies the supplied matrix to transform the input image, and the result is transformed along with the layer. There is no "current transformation" state—the matrix is applied independently.

*Tags: `params`, `transformation`, `mfr`, `reference`*

---

## Is there a built-in After Effects flag to prevent an effect from rendering?

The PF_OutFlag_AUDIO_EFFECT_ONLY flag can be used to avoid rendering on non-audio layers, but it does not work for audio layers themselves. For a universal solution that works on all layer types, manually copying the input to the output during the render pass is the recommended approach rather than relying on flags or output rectangle tricks.

*Tags: `params`, `aegp`, `reference`*

---

## How should parameter IDs be numbered to avoid issues when reordering parameters between plugin versions?

Parameter IDs in the non-_ID enum should start from 1, not 0. Value 0 is reserved for the default invisible layer parameter that serves as the effect's input. If you start numbering IDs from 0 (NULL), After Effects may assume you haven't bothered to ID params and will auto-assign IDs, which causes problems when reordering parameters in new versions. Always explicitly set parameter IDs starting from 1 to maintain backwards compatibility.

```cpp
enum
{
  MOVEMENT_DIRECTION_ID = 1,
  AMOUNT_SLIDER_ID = 2,
  NEW_FIELD_SLIDER_ID = 3
}
```

*Tags: `params`, `sdk`, `versioning`, `backwards-compatibility`*

---

## What is the recommended approach for changing parameter order in newer versions of an After Effects plugin while preserving saved project data?

Adobe's AE Plugin SDK Guide provides detailed documentation on changing parameter orders: https://ae-plugin-sdk-guide.readthedocs.io/effect-details/changing-parameter-orders.html. The guide explains how to properly handle parameter ID assignment and reordering to maintain backwards compatibility with projects saved using older plugin versions.

*Tags: `params`, `versioning`, `reference`, `sdk`*

---

## Where should AEGP_ExecuteScript be called in a C++ plugin to display a dialog each time the effect is applied?

AEGP_ExecuteScript should not be placed in GlobalSetup or ParamsSetup, as both occur only once per session. Instead, use sequence_setup, which is called once when a new instance is applied to a layer (not when duplicated or copy/pasted), or update_params_ui, which requires tracking to avoid launching the dialog multiple times for the same instance.

*Tags: `aegp`, `scripting`, `ui`, `params`*

---

## How can I read layer transform data like position, anchor, and scale in an After Effects plugin?

To get a layer's numeric transform values (position, anchor, scale), use AEGP_GetNewLayerStream to get the parameter's stream and AEGP_GetNewStreamValue to retrieve its numeric value. However, if you need all of the layer's transformations in the composition including parenting or camera movement effects that affect the layer's position on screen, use AEGP_GetLayerToWorldXform mixed with the camera matrix. Note that you'll need to get AEGP_PluginID during GlobalSetup(), which is required for many AEGP methods.

*Tags: `aegp`, `params`, `layer-checkout`, `transform`*

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

## Is it safe to start a background process from the UserChangedParam callback?

Yes, there are no issues starting background processes from UserChangedParam. In fact, it's the preferred location for such operations because it's user-triggered (rare), and it's called on the UI thread rather than the render thread, making it safer for UI-related work.

*Tags: `params`, `threading`, `ui`*

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

## What API function should be used to add a custom options button to an After Effects plugin?

The PF_SetOptionsButtonName() function should be used to implement an options button in After Effects plugins. This button will be displayed next to the 'Reset' button and provides a visible alternative to the hidden 'About' button in After Effects 2020 and later.

```cpp
PF_SetOptionsButtonName()
```

*Tags: `ui`, `pipl`, `params`*

---

## Why doesn't an After Effects plugin receive PF_Event_KEYDOWN when selected in the Effects Controls window?

The plugin needs to have at least one parameter with an ECW (Effects Controls Window) custom UI exposed and untwirled (not collapsed) to receive keyboard events. As a workaround, adding an invisible ARB parameter with custom UI can also fix the issue.

*Tags: `ui`, `debugging`, `params`, `aegp`*

---

## How can you get the index of an effect on a layer in After Effects?

To find the effect index on a layer, get the stream reference of the first effect parameter using EffectSuite4, then use AEGP_GetNewParentStreamRef to get the effect's stream representation, and finally use AEGP_GetStreamIndexInParent to retrieve the effect index on the layer. This is the reverse operation of AEGP_GetLayerEffectByIndex.

*Tags: `aegp`, `params`, `reference`*

---

## What is the best way to persist custom objects from the UI to the rendering thread in After Effects plugins?

The recommended approach is to use invisible arb parameters, which create undo entries and provide immediate invalidation of cached frames on changes. Alternatively, if arb parameters don't fit your design, you can devise a message-passing mechanism by placing instructions in a global data structure common to both threads, with the render thread checking for messages. Direct use of sequence data flattening outside of USER_CHANGED_PARAM or UI click/drag events is not reliable and should be avoided, as it can lead to sync issues with SmartRendering and PreRendering.

*Tags: `arb-data`, `threading`, `sequence-data`, `smart-render`, `params`*

---

## Why does changing a button parameter name cause a heap corruption crash when the project closes?

The issue occurs because modifying button parameter names using strcpy on param_union.button_d.u.namesptr outside of PF_Cmd_UPDATE_PARAMS_UI causes heap corruption that manifests during sequence data setdown. The solution is to only modify button text within the PF_Cmd_UPDATE_PARAMS_UI command selector. If modification is needed elsewhere, use strncpy with the string length (without the null terminator +1) to avoid corrupting the buffer, though this may cause garbled text display. Alternatively, consider using AEGP_ExecuteScript() with JavaScript to change button text, which may avoid the issue entirely.

```cpp
// CORRECT METHOD - only call within PF_Cmd_UPDATE_PARAMS_UI
PF_ParamDef param = *params[ID];
strcpy((char*)param.u.button_d.u.namesptr, newLabel);
AEGP_SuiteHandler handler(in_data->pica_basicP);
ERR(handler.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref, ID, &param));

// WORKAROUND - outside UPDATE_PARAMS_UI, use strncpy without null terminator
strncpy((char*)param_union.button_d.u.namesptr, label.c_str(), label.length());
```

*Tags: `params`, `ui`, `memory`, `debugging`, `sequence-data`, `aegp`*

---

## How can you refresh the UI to show parameter changes made in PF_Cmd_USER_CHANGED_PARAM?

The PF_OutFlag_REFRESH_UI flag can only be set from specific command selectors (PF_Cmd_EVENT, PF_Cmd_RENDER, PF_Cmd_DO_DIALOG), not from PF_Cmd_USER_CHANGED_PARAM where parameter changes typically need to be reflected. One workaround is to use AEGP_ExecuteScript() to execute JavaScript that triggers a UI update, or restructure the plugin logic to defer UI updates until one of the supported command selectors is called.

*Tags: `params`, `ui`, `render-loop`, `aegp`, `scripting`*

---

## How do I get the layer handle from a selected layer in a layer parameter?

The layer parameter's u.ld field contains a layer ID. Use the AEGP_GetLayerFromLayerID function to convert that layer ID into a layer handle, which you can then use to access the layer's transforms and other properties.

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How should PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS be used correctly when adding new parameters to legacy plugin versions?

The flag PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS is meant to provide a default value for old projects that don't have the new parameter. However, the PF_ADD_CHECKBOX macro copies the passed default over the passed value, which can cause unexpected behavior. The solution is to either copy the macro code into your own files and fix it there, or manually define the parameter without using the macro, defining it the old-fashioned way. Avoid modifying the AE headers directly as SDK updates would undo the changes.

```cpp
def.u.bd.value = FALSE;        // value for legacy projects which did not have this param
PF_ADD_CHECKBOX(
    STR(StrID_Checkbox_Param_Name),
    STR(StrID_Checkbox_Description),
    TRUE, // value for new applications, and when reset
    PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS,
    DOWNSAMPLE_DISK_ID);
```

*Tags: `params`, `pipl`, `sdk`, `reference`*

---

## How do I access input from different layers and at different times in an After Effects plugin?

Inputs from different layers and at different times can be accessed using layer parameters. Every effect has a default layer param (index 0) used to checkout the input layer. To get images from other times, look up 'WIDE_TIME' in the After Effects SDK documentation. Both the effect's own layer and other layers are accessed via layer parameters.

*Tags: `sdk`, `params`, `layer-checkout`*

---

## How can you collapse a parameter in the After Effects timeline using the SDK?

An alternative way to collapse a parameter is to use AEGP_SetStreamFlags() to hide and then unhide the parameter, which may collapse the parameter in the timeline as well. This approach can be used in addition to PF_ParamFlag_START_COLLAPSED and PF_ParamFlag_COLLAPSE_TWIRLY, which work for collapsing twirlies in the Effect Control Window (ECW).

*Tags: `sdk`, `params`, `ui`, `aegp`*

---

## When should I set expressions on parameters in a plugin to ensure proper undo behavior?

While UpdateParameterUI is a common location, consider using USER_CHANGED_PARAM instead if UpdateParameterUI causes undo stack issues. USER_CHANGED_PARAM may provide better control over when expressions are applied and how they integrate with After Effects' undo system.

```cpp
ERR(suites.StreamSuite4()->AEGP_SetExpression(globP->my_id, aegp_get_position_streamH, ("transform.position")));
```

*Tags: `aegp`, `stream`, `expression`, `params`*

---

## Why are undo groups not working when called from PF_Cmd_USER_CHANGED_PARAM?

There are two common issues: (1) When creating the action string, ensure it is properly null-terminated by using string literal syntax like `const A_char action[] = "Text";` instead of character array syntax. (2) AEGP_StartUndoGroup() cannot accept NULL as a parameter in newer After Effects versions—you must pass an empty string (two consecutive quotes "") instead. Also note that undo groups may only work properly with AEGP-style callbacks, and you should verify that parameter value changes are working in your context.

```cpp
const A_char action[] = "Text";
suites.UtilitySuite5()->AEGP_StartUndoGroup(action);
params[TEXTBOX_GRAD_OFFSETSTARTX]->u.fs_d.value = 50;
params[TEXTBOX_GRAD_OFFSETSTARTX]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
suites.UtilitySuite5()->AEGP_EndUndoGroup();
```

*Tags: `params`, `undo`, `aegp`, `sdk`*

---

## How do you force re-rendering when masks are deleted or added in an AEGP plugin?

Set the PF_OutFlag2_DEPENDS_ON_UNREFERENCED_MASKS flag in your plugin's global setup. This tells After Effects that your effect depends on mask information even when those masks aren't directly referenced as parameters, ensuring the render refreshes when masks change.

*Tags: `aegp`, `mfr`, `params`, `render-loop`*

---

## What other techniques can help trigger re-renders when external layer properties change in AEGP plugins?

Use GuidMixInPtr to force in external factors, set 'non param vary' in global setup, or consider implementing SmartFX for more sophisticated dependency tracking. Note that GuidMixInPtr may require SmartFX implementation and additional tweaking.

*Tags: `aegp`, `smartfx`, `mfr`, `params`*

---

## How can I make PF_Topic rename changes non-undoable without cluttering the undo history?

Start and end an undo group with an empty string as the name (two quotes, not NULL). Wrap your AEGP_SetStreamName() call within this undo group. The operation won't appear in the undo queue, but it remains technically undoable—just not listed in the user-visible undo history.

```cpp
// Start undo group with empty name
A_Err err = AEGP_StartUndoGroup("");
// Perform your name change
AEGP_SetStreamName(cur_streamH, new_name);
// End undo group
AEGP_EndUndoGroup();
```

*Tags: `aegp`, `params`, `undo`, `plugin`, `c++`*

---

## Is there a way to access app.settings.saveSetting from C++ plugin code instead of ExtendScript?

The conversation does not provide a direct C++ API equivalent for app.settings.saveSetting. However, the suggested workaround is to defer script execution to a different command handler like PF_Cmd_UPDATE_PARAMS_UI where no modal dialogs are blocking execution, allowing AEGP_ExecuteScript to run successfully.

*Tags: `aegp`, `scripting`, `params`*

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

## Are effect_ref and AEGP_EffectRefH the same thing and interchangeable?

No, effect_ref and AEGP_EffectRefH are not interchangeable. While they may have similar values because both denote RAM addresses, they hold different kinds of data. The effect_ref is used for very few things like GetNewEffectForEffect(), while AEGP_EffectRefH is used for a wide range of AEGP callbacks. Additionally, you cannot rely on the validity of either beyond the scope of a single AE call—after a call ends, AE may change its dataset and invalidate the references. To safely use AEGP functions from an idle hook, store your effect's index on the layer, layer ID, and comp item ID instead, then look up a fresh AEGP_EffectRefH during each idle hook call.

*Tags: `aegp`, `threading`, `params`, `memory`*

---

## How can I safely modify effect parameters from a custom thread in After Effects?

You cannot safely call most AEGP functions from custom threads—doing so will cause catastrophic crashes. The only exception is AEGP_CauseIdleRoutinesToBeCalled(), which is safe to call from any thread. To modify effect parameters, use an idle_hook instead. Store your effect's index on the layer, layer ID, and comp item ID (not AEGP_EffectRefH), then during the idle hook call, look up the comp, layer, and effect to obtain a fresh AEGP_EffectRefH for use in that single call. This ensures references remain valid and all operations occur on AE's main thread.

*Tags: `aegp`, `threading`, `idle-hook`, `params`*

---

## Why doesn't PF_Cmd_RENDER execute when an audio plugin uses PF_OutFlag_I_USE_AUDIO on a precomposed audio-only layer?

When a plugin sets PF_OutFlag_I_USE_AUDIO and the source layer is precomposed with no visual layers, After Effects may not trigger PF_Cmd_RENDER. The solution is to add PF_OutFlag_NON_PARAM_VARY to the global output flags in the plugin's global setup. This flag tells AE that the plugin's output varies based on non-parameter data (in this case, audio data), ensuring render commands are properly triggered even when the precomp contains only audio. Note: This flag is not required when the source layer is not precomposed, as AE automatically handles render triggering in that case.

*Tags: `audio`, `params`, `aegp`, `render-loop`, `layer-checkout`*

---

## Why does an angle parameter show weird fixed-point values instead of the expected float value?

Angle parameters in After Effects use fixed-point representation internally. To convert a fixed-point angle value to float, use the FIX_2_FLOAT(X) macro. Alternatively, use the PF_AngleParamSuite1::PF_GetFloatingPointValueFromAngleDef() function from the SDK, which handles the conversion automatically.

```cpp
FIX_2_FLOAT(paramdef.u.ad)
```

*Tags: `params`, `sdk`, `aegp`, `reference`*

---

## How do you capture when a popup parameter selection changes in an After Effects plugin?

To capture popup menu selection changes, the parameter must be created with the PF_ParamFlag_SUPERVISE flag. When such a parameter is changed, the plugin receives a USER_CHANGED_PARAM call. All plugin calls go through the main entry point function (EffectMain), which should implement the selector to handle this notification.

*Tags: `params`, `ui`, `aegp`, `plugin-development`*

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

## Does arbitrary data support undo/redo operations in After Effects plugins?

Yes, arbitrary data supports undo/redo operations, unlike sequence data which does not. This makes arbitrary data suitable for storing user-modified parameter state that needs to be reversible.

*Tags: `arb-data`, `params`*

---

## Can AEGP_RegisterIdleHook be used in an effect plugin to sync sequence data and UI?

AEGP_RegisterIdleHook is a general call that is not instance-specific, even when placed in an effect plugin. It executes once globally rather than once per effect instance. To access sequence data from individual effect instances, you must use EffectCallGeneric() to find and call each instance separately, allowing each instance to perform its own sequence data comparison. This is the only way to access sequence data from an idle hook in an effect plugin.

*Tags: `aegp`, `sequence-data`, `params`, `effect-plugin`*

---

## How do you add static text labels to an After Effects effect UI?

You can use PF_ADD_TOPIC and PF_END_TOPIC macros without putting any params in between to create a text section. Alternatively, you can use PF_ADD_NULL, declare a minimal custom UI size, and not implement it.

*Tags: `ui`, `pipl`, `params`, `reference`*

---

## What does the dimensionL parameter represent in AEGP_GetKeyframeTemporalEase, and how should it be used?

The dimensionL parameter refers to the separate X, Y, and Z axes for properties where each axis can be controlled individually with different ease settings. For example, position and scale properties can have different ease settings per axis. The dimensionL ranges from 0 to (temporal_dimensionality - 1), where 0 typically represents a single dimension for properties like sliders, colors, or points that animate as one uniform property.

*Tags: `aegp`, `params`, `keyframe`*

---

## What is an alternative approach to copying and pasting keyframes between parameters in After Effects plugins?

Instead of manually copying keyframe properties through AEGP_GetKeyframeTemporalEase, you can emulate copy and paste functionality using a collection to set selected keyframes and then execute the native After Effects copy/paste commands via AEGP_DoCommand. Alternatively, you can implement the entire process in JavaScript and invoke it from the C API using AEGP_ExecuteScript().

*Tags: `aegp`, `scripting`, `params`*

---

## What is the correct way to change a parameter value during PF_Event_IDLE?

According to Adobe SDK documentation, parameter values can only be modified during PF_Cmd_USER_CHANGED_PARAM and specific PF_Cmd_EVENT types (PF_Event_DO_CLICK, PF_Event_DRAG, PF_Event_KEYDOWN). To change parameters outside these events, use AEGP_SetStreamValue() from the AEGP suite. If the parameter has keyframes, use the keyframe suite instead, though AEGP_SetStreamValue() will still work despite what the documentation states. AEGP_SetStreamValue() is available in all After Effects versions since 2000.

```cpp
params[someindex]->u.fs_d.value = somevalue;
params[someindex]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `sdk`, `idle-event`*

---

## How can I access data from another plugin in After Effects?

Getting parameter data is relatively straightforward using both the C and JavaScript APIs, and the 'project dumper' sample project demonstrates this. However, there are two significant limitations: (1) accessing arbitrary parameter data (custom type params) is difficult—while you can retrieve the data chunk, deciphering its structure is challenging without knowing the format; (2) accessing sequence data (data stored internally by the plugin rather than in params) is only possible if the plugin provides specific custom infrastructure to expose it, otherwise only the plugin itself can access it.

*Tags: `arb-data`, `params`, `sequence-data`, `scripting`, `reference`*

---

## Is there a sample project that demonstrates how to extract parameter data from After Effects projects?

Yes, Adobe provides the 'project dumper' sample project as a reference implementation for getting parameter data from plugins using both the C and JavaScript APIs.

*Tags: `reference`, `params`, `scripting`, `open-source`*

---

## Can I implement a mirror effect using the transform_world function?

Yes, transform_world handles all 2D transformations including rotating, scaling, and moving. If 2D transformations are sufficient for your mirror effect, there is no reason not to use transform_world rather than manually calculating transformations.

*Tags: `mfr`, `params`, `reference`*

---

## How can you adjust the level of motion blur when using transform_world in a plugin?

According to shachar carmi, transform_world spreads 16 instances across provided matrices to create motion blur. The level of motion blur can be adjusted in three ways: (1) by controlling the length of the blur (shutter angle), (2) by changing the number of samples in the blur, or (3) by adjusting the mix between the motion-blurred result and non-motion-blurred render. Providing more matrices gives a more refined look with better rotations and curved paths, though it doesn't increase the total number of samples. With 2 matrices you get 16 linear positions between them; with 3 matrices you get 8 samples from mat1 to mat2 and another 8 from mat2 to mat3.

*Tags: `transform_world`, `motion-blur`, `smartfx`, `premiere`, `params`*

---

## How can I make PF_Param_POINT and PF_Param_POINT_3D parameters maintain constant values independent of layer resizing?

PF_Param_POINT and PF_Param_POINT_3D parameters by default scale proportionally with layer dimensions. To maintain absolute coordinates invariant under layer size changes, shachar carmi recommends three workarounds: (1) Create a custom parameter with custom composition UI to have full control over behavior; (2) Add an expression to your point parameter programmatically that divides the current position by layer dimensions and multiplies by a scaling factor to maintain the same absolute coordinates when layer dimensions change; (3) Try the PF_ValueDisplayFlag_PIXEL flag, though its applicability to point parameters is undocumented and may have unexpected behavior.

*Tags: `params`, `ui`, `sdk`*

---

## How can I detect when the user performs undo or redo actions in my After Effects effect plugin so I can update my custom UI?

One effective approach is to save a random number or state identifier with your data, then poll the effect data (stored in a data parameter) once or twice per second to check if the state index differs from what your UI window expects. If there's a mismatch, the data has changed via undo/redo or project load. To optimize performance, you can save the random number to a separate parameter like a slider. When saving data, use StartUndoGroup() and EndUndoGroup() to bundle operations into a single undo entry. This technique works well for effects with custom external UIs that manage internal parameters in a structure, which is then serialized to an AEGP data parameter using AEGP_SetStreamValue().

*Tags: `undo`, `redo`, `ui`, `aegp`, `params`, `debugging`*

---

## How can I group parameters into folders in an After Effects plugin?

Parameters can be grouped into folders (commonly referred to as "groups") by defining them between PF_ADD_TOPIC and PF_END_TOPIC macros. Topics can also be nested to create hierarchical parameter organization.

*Tags: `params`, `ui`, `pipl`, `sdk`*

---

## How can I display a custom image or logo in an effect parameter?

Every parameter can have a custom UI in which you can draw an image. Look at the "Color Grid" sample to see how a custom UI is implemented in the ECW (Effect Custom UI). You can use any parameter type with a custom UI—in the case of Keylight, it appears to be an arbitrary parameter with a custom UI. You may need to simplify the sample if you only want to draw the image without handling clicks and drags.

*Tags: `params`, `ui`, `custom_ecw_ui`, `effect_api`*

---

## How do I embed an image resource in a Windows .aex plugin file?

Add your image file as an image resource or generic binary resource from the Visual Studio File menu. You will get a custom identifier for this resource. Then use standard Windows resource functions to load it. Example code: HRSRC myResource = ::FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA); HGLOBAL myResourceData = ::LoadResource(NULL, myResource); void* pMyBinaryData = ::LockResource(myResourceData);

```cpp
HRSRC myResource = ::FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA);
HGLOBAL myResourceData = ::LoadResource(NULL, myResource);
void* pMyBinaryData = ::LockResource(myResourceData);
```

*Tags: `windows`, `build`, `params`, `ui`, `deployment`*

---

## How can you track which version of a plugin was used to save an After Effects project?

There is no dedicated API in the After Effects SDK to retrieve the plugin version that saved a project. The recommended approach is to store the version number in the sequence data of your effect. This allows you to detect version changes when reopening projects and handle updates accordingly.

*Tags: `sequence-data`, `params`, `aegp`, `versioning`*

---

## How do you prevent sequence data from being lost during the flatten call when saving projects?

In After Effects versions before CC2017, you need to purge unwanted sequence data during the flatten call and reconstruct it during resetup. Starting with CC2017 and later, a new API call allows you to keep the original sequence data while delivering a flattened copy, preventing data loss. However, this new call is currently only implemented for UI event calls and not for copy, duplicate, or save operations. Eventually it will replace the older flatten call entirely.

*Tags: `sequence-data`, `aegp`, `params`, `macos`, `windows`*

---

## Why is sequence_data different between PF_Cmd_RENDER and PF_Cmd_USER_CHANGED_PARAM in After Effects CC 2015 and later?

In After Effects CC 2015 and later, the render thread and UI thread have separate sequence_data handles that only synchronize at specific occasions. This is a design change from earlier versions like CS5.5. To force synchronization, set PF_OutFlag_FORCE_RERENDER during UserChangedParam, which forces a flatten call on the UI thread and an unflatten on the render thread, syncing the render thread's sequence data with the UI data.

*Tags: `sequence-data`, `threading`, `params`, `render-loop`, `ui`*

---

## How should arbitrary data parameters be saved and loaded in After Effects plugins?

When dealing with an ARB (arbitrary) parameter, you should honor the PF_Arbitrary_UNFLATTEN_FUNC and PF_Arbitrary_FLATTEN_FUNC callbacks. The sequence data commands (PF_Cmd_SEQUENCE_FLATTEN and PF_Cmd_SEQUENCE_RESETUP) have nothing to do with ARB parameter persistence. ARB data may get flattened/unflattened during various events such as copy/paste, not only during save/load. If data doesn't reconstruct correctly after honoring these callbacks, the issue is likely with how the flatten/unflatten operations are implemented.

*Tags: `arb-data`, `params`, `aegp`, `memory`*

---

## Is there a sample project demonstrating how to implement arbitrary data parameters in After Effects?

The 'ColorGrid' sample project is a good reference for understanding how to properly handle arbitrary parameters. It demonstrates the correct implementation of ARB param flatten and unflatten operations.

*Tags: `arb-data`, `params`, `reference`, `open-source`*

---

## How can I access mask position and tangent information at a particular parameter for a stroke effect plugin?

Use the PF_PathEvalSegLengthDeriv1 function from the After Effects SDK. This function allows you to evaluate path segments and obtain derivative information, which provides the position and tangent data needed for stamping patterns along mask paths without having to write your own Bezier curve rasterizer.

*Tags: `aegp`, `mask`, `params`, `reference`*

---

## How do you set AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 values in the PiPL.r file?

To set these flags correctly, place a breakpoint in the plugin's global setup function and observe the numerical values being applied to the outflags and outflags2 variables. Copy these exact numerical values to the corresponding fields in your PiPL.r file. The values must match what your plugin sets during initialization, or a mismatch error will occur at plugin launch. After making changes, perform a clean rebuild to ensure the pipl_tool regenerates the .rc file from your updated .r file.

*Tags: `pipl`, `params`, `build`, `debugging`*

---

## How can you exchange data between an effect plugin and an IdleHook callback function?

While an IdleHook can reside in an effect plugin, the callback doesn't have direct access to the effect's in_data structure or parameters. The recommended approach is to allocate and lock a shared piece of RAM during global setup, store it in the global data structure, and pass it as the IdleRefCon to AEGP_RegisterIdleHook. AE will then pass this shared memory to the hook handling function on each idle call. Since this RAM is shared between all instances and threads, use a mutex to prevent race conditions. During idle calls, you can read/write parameter values using AEGP_GetNewStreamValue(), passing the instance reference data through the shared RAM.

*Tags: `aegp`, `memory`, `threading`, `params`*

---

## Why doesn't PF_Event_IDLE run after restoring a project until a control is touched?

PF_Event_IDLE is only called when an effect instance is the sole selected item. When you apply an effect from the menu, it automatically becomes selected alone, so idle events trigger. However, when loading a project, effects are not automatically selected, so idle events don't fire until a control is manually touched. This is expected behavior and cannot be overridden.

*Tags: `aegp`, `ui`, `debugging`, `params`*

---

## How can I handle UI synchronization tasks that need to run even when an effect isn't selected after project restore?

Use AEGP_RegisterIdleHook() to register a general-purpose idle hook that receives regular calls not tied to a specific effect instance. During these calls, you can scan the current comp or entire project for your effect instances using AEGP_GetActiveItem(), AEGP_GetFirstProjItem(), AEGP_GetNextProjItem(), AEGP_GetCompFromItem(), AEGP_GetCompLayerByIndex(), AEGP_GetLayerEffectByIndex(), and AEGP_GetInstalledKeyFromLayerEffect() to identify your effects. You can then programmatically select your effect using AEGP_NewCollection(), AEGP_CollectionPushBack(), and AEGP_SetSelection() so your regular effect process picks it up.

*Tags: `aegp`, `ui`, `params`, `debugging`*

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

## How can I add markers to a layer from an effect panel in After Effects?

The user reported finding a solution to add markers to layers from an effect panel, though the specific implementation details were not fully shared in this conversation. The discussion suggests using the After Effects SDK to manipulate layer markers programmatically from custom UI panels.

*Tags: `sdk`, `ui`, `params`, `aegp`*

---

## How do modular plugins like Plexus work, where multiple plugins are stacked to produce a final result?

Modular plugins typically don't render anything themselves. Instead, the individual 'module' effects allow users to input parameters, while a main/master effect reads the parameter values from all the module effects and performs the actual rendering based on those combined parameter values. This is simpler and more elegant than trying to maintain off-screen buffers or create custom channels.

*Tags: `params`, `architecture`, `plugin-design`, `reference`*

---

## What is the difference between PF_SUBPIXEL_SAMPLE and subpixel_sample in After Effects plugins?

PF_SUBPIXEL_SAMPLE is a macro that wraps the in_data->utils->subpixel_sample() function. Both provide the same sampling functionality. The macro approach can be used even though some documentation may mark it as unsupported. The key performance difference is not in which function you use, but in how you manage suite acquisition—acquire the sampling suite once before your loop rather than repeatedly inside it.

*Tags: `mfr`, `aegp`, `sampling`, `params`*

---

## Should arbitrary parameter data be stored in a single struct for all arb params or separate structs per parameter?

You can use either approach, but a single generalized arb handler that is reusable across multiple arb params is more efficient. Store the data in AE rather than in the handler class itself. During global setup, create and store the handlers in the global data structure, then pass pointers to them during param setup. Keep handlers locked throughout the session using placed new allocation. The key is that except for the 'create new' function, most handler functions remain the same across different arb types.

```cpp
// C++ style approach with reusable handler
// Store handler pointer in arb definition
// Handler class with virtual methods for each arb type
class ArbHandler {
public:
  virtual PF_Err CreateDefault(PF_Handle *arbH) = 0;
  virtual PF_Err Handle(PF_Cmd cmd, PF_Handle arbH) = 0;
};

// During global setup:
ArbHandler *handler = new ArbHandler();
ArbHandler **handlerH = (ArbHandler**)AE_Allocate(sizeof(ArbHandler*));
*handlerH = handler;
// Lock handle and store in global data
```

*Tags: `arb-data`, `params`, `plugin-architecture`, `c++`, `aegp`*

---

## How can you determine which arbitrary parameter is being referenced in callback handlers that are shared across multiple arb params?

Use the C style approach with a switch statement on the disk_id that AE transfers with each PF_Cmd_ARBITRARY_CALLBACK call, or use the C++ style approach where you place a pointer to an arb handling class in the arb definition parameter. This pointer is sent back with every callback, allowing you to use virtual methods to handle different arb types. The C++ approach with virtual methods is cleaner and easier to maintain than switch statements.

```cpp
// C style with disk_id switch
case PF_Cmd_ARBITRARY_CALLBACK:
  switch(params->arb_d.disk_id) {
    case ARB_PARAM_1:
      return Handle_Arb1(arbH);
    case ARB_PARAM_2:
      return Handle_Arb2(arbH);
  }

// C++ style - pointer passed back in callback
// Arb definition includes: (void*)arbHandlerPtr
// In callback, cast back and call virtual methods
```

*Tags: `arb-data`, `params`, `aegp`, `callback`*

---

## How can you dynamically change effect parameter popup entries in After Effects plugins?

You cannot change the number of popup entries dynamically, but you can change the names of entries. The recommended approach is to create multiple hidden popups with different numbers of options (1, 2, 3, 4, 5, etc.) up to the maximum number of options needed, then show/hide the appropriate popup and populate its strings during ParamChange() or UI() callbacks. This allows the effect to appear to have dynamic popup content while working within SDK constraints.

*Tags: `params`, `ui`, `aegp`, `reference`*

---

## What is a good reference for dynamically modifying popup parameter strings in After Effects plugins?

Adobe's forum thread at https://forums.adobe.com/message/2446142#2446142 contains detailed explanation on how to modify popup entry names and manage hidden popups for dynamic parameter changes. Additionally, the thread 'Re: Questions before writing my plugin' discusses similar techniques for modifying parameters during ParamChange() and UI() calls.

*Tags: `params`, `ui`, `reference`, `debugging`*

---

## How can an effect plugin listen to changes in another layer and react to them?

An effect plugin can monitor changes in another layer by creating an invisible parameter with an expression that links to a source parameter on the target layer. When the source parameter changes (e.g., text content), the invisible parameter will update and trigger a re-render. To ensure the expression evaluates correctly, use PF_PUI_INVISIBLE to mark the parameter as invisible rather than AEGP_GetDynamicStreamFlags.

*Tags: `params`, `ui`, `render-loop`, `aegp`*

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

## How can you get the zoom factor in a UI doClick handler in After Effects?

Use event_extraP->cbs.frame_to_source() to convert frame coordinates to source coordinates. Check the coordinates of (100, 0) and divide the resulting X value by 100 to get the scale factor.

```cpp
// Get zoom factor using frame_to_source
A_Ratio x, y;
event_extraP->cbs.frame_to_source(100, 0, &x, &y);
A_Ratio scale_factor = x / 100.0;
```

*Tags: `ui`, `params`, `aegp`*

---

## How can I access output width and height values in a UI event handler?

The out_data.width and out_data.height values are only available during rendering and will be 0 in UI event handlers. To access these values in a DragHandle or DoClick event, you need to cache the desired values in sequence_data during the render call, then retrieve them during the event call. Note that in After Effects 13.5+, sequence_data cannot be changed from the render thread—only changes from the UI thread persist to the project and render thread.

*Tags: `ui`, `render-loop`, `sequence-data`, `threading`, `params`*

---

## How can I get composition dimensions from within an After Effects plugin?

To get composition dimensions, use the following AEGP API sequence: (1) AEGP_GetEffectLayer() to get the effect layer, (2) AEGP_GetLayerParentComp() to get the parent composition, (3) AEGP_GetItemFromComp() to get the item from the composition, (4) AEGP_GetItemDimensions() to retrieve the dimensions. Note that this returns the comp size at full resolution, so you may need to account for downsample factors depending on your use case.

*Tags: `aegp`, `params`, `output-rect`*

---

## Why do project files crash after adding new parameters to an effect plugin?

Project files crash when parameters are added because After Effects uses DISK_ID values to associate saved parameter data with plugin parameters. If you change the order of parameter definitions in your enum or reassign DISK_IDs, After Effects cannot correctly map the old saved data to the new parameters. For example, if a slider parameter with DISK_ID=3 is replaced by a checkbox parameter in the new version, the saved slider data will be incorrectly applied to the checkbox, causing crashes. The solution is to ensure that existing parameters retain their exact same DISK_ID values from the old version, and only assign new DISK_IDs to newly added parameters. The DISK_ID order has nothing to do with the visual parameter order in the UI.

*Tags: `params`, `arb-data`, `deployment`, `aegp`*

---

## What is the relationship between parameter definition order and DISK_IDs in After Effects plugins?

There are two separate enums in After Effects plugin parameter setup: one that correlates with the order of definitions in param_setup (used to access params by index), and another for DISK_IDs (used as a unique identifier for each parameter). These two do not need to match. A parameter can be in index #5 in the UI but still have DISK_ID=3 if it had DISK_ID=3 in the previous version. The DISK_ID is what matters for backward compatibility and project file loading, not the parameter order or index.

*Tags: `params`, `arb-data`, `aegp`*

---

## How should you handle sequence data structure changes when adding new parameters?

When modifying sequence data structure, if you extend or modify the structure, old project files loaded with the new plugin will attempt to match the old sequence data to the new structure, potentially causing conflicts. Best practice is to only add new parameters with new IDs at the end of the current parameter set without changing the order of existing parameters. Do not replace existing parameter definitions, only append new ones. This ensures backward compatibility with projects saved using older plugin versions.

*Tags: `params`, `arb-data`, `sequence-data`, `deployment`*

---

## How does the A_Time structure work and how do you calculate the value and scale components?

A_Time is represented as a ratio where value/scale equals time in seconds. You can use any whole numbers that express the desired time as a numerator/denominator ratio. For example, {1500, 500} equals 1.5 seconds, and {94208, 25600} equals approximately 3.68 seconds. To set A_Time from a double value like 1.875 seconds, you need to find appropriate whole numbers whose ratio equals that decimal value.

*Tags: `aegp`, `params`, `time`, `keyframe`*

---

## How do you insert keyframes at specific times using AEGP_InsertKeyframe with the correct A_Time values?

Use the AEGP_InsertKeyframe function from the KeyframeSuite3, passing a properly formatted A_Time structure where the ratio of value to scale equals your desired time in seconds. For example, to insert a keyframe at 1.2 seconds, you could use A_Time {12, 10} or {60, 50}. The function signature is: AEGP_InsertKeyframe(position_streamH, AEGP_LTimeMode_LayerTime, &timePT, &new_indexL) where timePT is your A_Time structure.

```cpp
ERR(suites.KeyframeSuite3()->AEGP_InsertKeyframe(position_streamH,
  AEGP_LTimeMode_LayerTime,
  &timePT,
  &new_indexL));
```

*Tags: `aegp`, `keyframe`, `params`, `time`*

---

## How can I retrieve keyframe values and interpolation data from After Effects parameters in a plugin?

Use the AEGP_GetNewStreamValue() function to get parameter values at any composition time. This function allows you to retrieve interpolated values based on keyframes and their temporal interpolation settings (such as Bezier). It is available in several SDK sample projects and will return the calculated value at any given frame, taking into account all keyframe interpolation between start and end frames.

*Tags: `aegp`, `params`, `reference`, `sdk`*

---

## How can an After Effects plugin programmatically create keyframes on its own parameters when a button is pressed?

During the USER_CHANGED_PARAM event for the button parameter, use AEGP_GetNewEffectForEffect to get the effect reference, then use AEGP_GetNewEffectStreamByIndex with the target parameter index to get the parameter stream. Once you have the stream reference, you can use keyframe manipulation functions to set keyframes as needed.

*Tags: `params`, `aegp`, `keyframes`, `sdk`*

---

## How do you get frame data like PF_world from sequence_data in After Effects plugins?

sequence_data is a PF_Handle that you should cast to whatever structure you're using. Since you know the structure you're working with, you can cast the handle directly to access the corresponding memory region. The key distinction is between smartFX and non-smartFX effects: non-smartFX effects get input images through params[0]->u.ld, while both types can use PF_CHECKOUT_PARAM() to specify source times. smartFX effects use checkout_layer_pixels() instead. For any source time checkout, you must set the PF_OutFlag_WIDE_TIME_INPUT flag. For accessing pixels outside layer params, use AEGP_GetReceiptWorld().

*Tags: `sequence-data`, `smartfx`, `layer-checkout`, `params`*

---

## Why doesn't changing a layer parameter's path trigger a re-render in After Effects?

Layer parameters track changes in the layer's source pixels only, not transforms, masks, or effects. This is intended behavior in After Effects. A layer selector provides the selected layer's source pixels, and only changes to those pixels trigger re-renders. To work around this, you can use an expression that checks the path value and drives a slider parameter to force updates, or alternatively create a light parented to the tracked layer with the I_USE_3D_LIGHTS flag set (though this approach has its own complications).

*Tags: `params`, `layer-checkout`, `render-loop`, `aegp`*

---

## How do you hide a layer parameter in After Effects without breaking parameter evaluation?

To hide a layer parameter while maintaining evaluation, you must set the ui_flags during parameter setup using PF_PUI_NO_ECW_UI. If you hide the parameter later using AEGP_SetStreamFlag(), the parameter will stop evaluating. The correct approach is to hide it at initialization time rather than at runtime.

```cpp
def.ui_flags = PF_PUI_NO_ECW_UI;
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## How can I set time-variant stream values for every frame in an AEGP plugin without creating keyframes at each frame?

According to shachar carmi, expressions are the only way to modify parameter values between keyframes without creating keyframes at each frame. Using AEGP_SetLayerStreamValue() or the Keyframe Suite will automatically create a keyframe at the current time if the stream is already keyframed. On AE 13.5+, only the UI thread can safely change parameter values. The SetStreamValue() function sets the value at the current item time, but if keyframes already exist on that stream, it will create or modify a keyframe at that time.

*Tags: `aegp`, `params`, `keyframe`, `threading`*

---

## What is the recommended approach for animating parameters in AEGP plugins?

Use the Keyframe Suite to animate parameters. This is the proper API for managing keyframe-based animation. Direct SetStreamValue() calls will create keyframes on already-keyframed streams, so the Keyframe Suite should be used for intentional parameter animation.

*Tags: `aegp`, `params`, `keyframe`*

---

## How do you query a color picker parameter and get its floating-point values instead of 16-bit 0-255 range?

Use the PF_GetFloatingPointColorFromColorDef function from the ColorParamSuite1. Create a PF_PixelFloat variable and call suites.ColorParamSuite1()->PF_GetFloatingPointColorFromColorDef(in_data->effect_ref, params[COLOR_PICKER_INDEX], &fp_colorP) to convert the color parameter to floating-point representation. This allows you to work with color values in full floating-point precision rather than the default 16-bit packed color format.

```cpp
PF_PixelFloat fp_colorP;
suites.ColorParamSuite1()->PF_GetFloatingPointColorFromColorDef(
    in_data->effect_ref,
    params[MATTE_PICKER],
    &fp_colorP);
```

*Tags: `params`, `ui`, `sdk`, `reference`*

---

## What is the best way to store all pixels from a source layer into an array for processing?

Use the MemorySuite to allocate an int array of any size, then lock the memory handle onto an int pointer which will behave as an array. You can pass a pointer to your custom data as the refcon parameter to the Iterate8Suite, cast it in your iteration function, and fetch your int data according to the x,y coordinates provided by the suite. Alternatively, manually iterate through pixels using getXY() helper function to access PF_Pixel data by calculating the memory offset using rowbytes.

```cpp
static PF_Pixel *getXY(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

// In Render function:
for(int i = 0; i < tInfo->width; i++){
  for(int j = 0; j < tInfo->height; j++){
    PF_Pixel currentPixel = *getXY(*tInfo->input, i, j);
    int alpha = currentPixel.alpha;
    int red = currentPixel.red;
    int green = currentPixel.green;
    int blue = currentPixel.blue;
  }
}
```

*Tags: `memory`, `aegp`, `params`, `reference`*

---

## How can I allow users to abort a long-running keyframe baking process triggered by PF_Cmd_USER_CHANGED_PARAM?

Instead of blocking the entire baking process in a single PF_Cmd_USER_CHANGED_PARAM call, implement incremental processing: get a clock reading at the beginning and process keyframe changes in ~1 second chunks. Let the AEGP handle continuation during idle time. This allows users to interrupt for interaction, and processing resumes immediately after. This approach enables user interaction while the baking process continues in segments rather than blocking on a single long operation.

*Tags: `params`, `aegp`, `ui`, `threading`*

---

## How do I use PF_SPRINTF to format messages in an After Effects plugin?

PF_SPRINTF is a macro that expects the same arguments as the standard C sprintf() function. It expands to (*in_data->utils->ansi.sprintf)(args...). The function takes a buffer (like out_data->return_msg) as the first argument, followed by a format string and any values to format. Do not pass in_data or other non-format arguments where sprintf() expects a format string. The macro must be used in a context where the in_data variable is available.

```cpp
PF_SPRINTF(out_data->return_msg, "test hello");
PF_SPRINTF(out_data->return_msg, "value: %d", some_int);
```

*Tags: `ui`, `params`, `debugging`, `reference`*

---

## How can I detect if an effect instance is already applied to a layer to prevent duplicate applications?

To detect if another instance of your effect is present on a layer, you need to scan the layer's effects using AEGP_GetLayerNumEffects and AEGP_GetLayerEffectByIndex, then check each effect's installed key using AEGP_GetInstalledKeyFromLayerEffect and match it to your own. If an effect is already present in a saved effect, you'll receive a SequenceResetup() or Unflatten() call instead of SequenceSetup(). Note that an effect cannot directly remove another effect from the same layer; you can either use a separate AEGP with an idle hook for deletion, or set a flag on the second instance to disable its processing.

*Tags: `aegp`, `params`, `sdk`*

---

## How can I toggle a stopwatch or create keyframes for slider parameters programmatically in an Effects plugin?

You can create keyframes for any parameter using AEGP_KEYFRAMESUITE3. Most AEGP suites work for both AEGP plugins and Effects plugins, including the keyframe suite. The "CheesyCheese" sample project demonstrates how to implement this functionality.

*Tags: `aegp`, `params`, `keyframe`, `effects`*

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

## How can I set default values for PF_Param_POINT parameters based on sequence resolution in After Effects plugins?

Point parameters in After Effects use percentage-based coordinates for their defaults, so a default value of {50, 50} will always represent the center of the composition or sequence, regardless of resolution. This means you don't need to dynamically set defaults based on sequence information during PF_Cmd_USER_CHANGED_PARAM or other callbacks—the percentage-based system handles this automatically.

*Tags: `params`, `ae transition extensions`, `premiere`, `pf_paramdef`*

---

## How do you force After Effects to re-render when custom UI parameters change but the effect's cached result is stale?

Use PF_GetCurrentState and PF_HasParamChanged during a draw event to detect if the plug-in's parameter state has changed since the last draw call. If parameters have changed, set the evt_out_flags to include PF_OutFlag_FORCE_RERENDER to force a fresh render. Simply setting change_flags or PF_EO_HANDLED_EVENT alone is not sufficient to overcome AE's caching behavior.

*Tags: `caching`, `ui`, `smart-render`, `params`*

---

## How should you safely read camera data from an effect plugin to avoid crashes during parameter dragging?

Use PF_ABORT before reading camera data to check if an abort signal has been triggered. If you get an abort signal, skip reading the data. This helps prevent crashes that can occur during parameter dragging operations, particularly when dealing with composition cameras and keyframing operations.

*Tags: `aegp`, `params`, `debugging`, `render-loop`*

---

## How can you save and restore arbitrary data from a 3rd party plugin's property without using presets?

You can use the AEGP (After Effects General Plugin) approach: have an effect plugin communicate with an AEGP via a special suite (see "checkout" and "sweetie" samples). When the effect needs to apply changes, it sends data to the AEGP and stores it there without executing changes immediately. Let the effect finish execution and return. Then, during the AEGP's idle_hook call, check for messages from the effect and execute the changes after the effect is no longer in mid-call. This avoids crashes from modifying the scene while an effect is operating on it. Alternatively, you can bundle a preset file within your plugin and apply it via applyPreset() with a file path, avoiding the need to keep presets in Adobe's preset directory.

```cpp
var thePreset = File("C:/Program Files/Adobe/Adobe After Effects CS5.5/Support Files/Presets/Behaviors/wigglerama.ffx");
theLayer.applyPreset(thePreset);
```

*Tags: `aegp`, `arb-data`, `params`, `plugin-communication`*

---

## How can I efficiently update many parameters without causing UI slowness from repeated PF_ChangeFlag_CHANGED_VALUE flags?

Instead of setting PF_ChangeFlag_CHANGED_VALUE for each of many parameters, use AEGP_SetStreamValue() from the AEGP Suite to update parameter values more efficiently. Alternatively, if the parameters don't need to be directly exposed as individual sliders in the UI, store them in sequence data or arbitrary data, which are automatically saved with the project without requiring change flags for each value.

```cpp
params[paramId]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `ui`, `performance`*

---

## What is the recommended way to store large numbers of parameters that need to be saved in a project file?

For storing many parameters (e.g., 1000+ values) that need to be persisted in the project file, use sequence data or arbitrary data rather than individual built-in parameter types. These are automatically saved and recalled with the project. Ensure you correctly handle copying and flattening/deflattening of values in the appropriate functions, especially if using pointers. The SDK examples demonstrate the correct implementation.

*Tags: `arb-data`, `sequence-data`, `params`, `deployment`*

---

## How do you set a parameter group to default to open (expanded) instead of collapsed?

Use the PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG flag. To selectively control which groups are open or closed, set the PF_ParamFlag_COLLAPSE_TWIRLY flag in the def.flags structure when calling PF_ADD_TOPIC(). For example: AEFX_CLR_STRUCT(def); def.flags = PF_ParamFlag_COLLAPSE_TWIRLY; PF_ADD_TOPIC("name", GROUP_DISK_ID);

```cpp
AEFX_CLR_STRUCT(def);
def.flags = PF_ParamFlag_COLLAPSE_TWIRLY;
PF_ADD_TOPIC("name", GROUP_DISK_ID);
```

*Tags: `params`, `ui`, `sdk`*

---

## How do I define a mask parameter in an effect plugin?

Use the PathMaster sample project as a reference, which demonstrates how to add a path parameter to your effect. Note that mask selector parameters are limited to masks on the same layer as the effect only. There is no built-in API for a mask selector on a different layer. To access masks on other layers, use the AEGP suites, but be aware that the API function to render a mask will only work on the local effect layer with mask selector parameters. For cross-layer mask selection, you must either implement your own path rasterizing function or use workarounds such as invisible mask selectors pointing to temporary masks.

*Tags: `mask`, `params`, `aegp`, `ui`, `reference`*

---

## How can I retrieve text properties like font, color, and size from a text layer in After Effects?

You can use the JavaScript API through AEGP_ExecuteScript() to access text layer properties. To fetch the font name from a text layer (CS6 and up), use: var textProp = myTextLayer.property("Source Text"); var textDocument = textProp.value; textDocument.font; This approach works because text layers have limited SDK support, so scripting is the recommended method. The script is passed as a char pointer and doesn't require an external script file. For more details on launching AEGP_ExecuteScript() and retrieving results back to the C side, see: http://forums.adobe.com/message/3625857#3625857

```cpp
var textProp = myTextLayer.property("Source Text");
var textDocument = textProp.value;
textDocument.font;
```

*Tags: `aegp`, `scripting`, `params`, `reference`*

---

## How do you hide and show parameter groups dynamically in After Effects plugins when a checkbox is toggled?

Use AEGP_GetNewEffectStreamByIndex to get stream references for parameters in each group, then call AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to toggle visibility. Handle this in response to PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_UPDATE_PARAMS_UI commands. Note that you must hide/unhide every individual parameter within a group, not just the group topic itself, otherwise bugs can occur such as parameters appearing in the timeline when keyframed.

```cpp
AEGP_StreamRefH wfm_streamH = NULL;
A_Boolean hide_wfmB = !params[QPGA_SCOPE_OPTION_WFM]->u.bd.value;
AEGP_EffectRefH meH = NULL;
AEGP_SuiteHandler suites(in_data->pica_basicP);
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(NULL, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(NULL, meH, QPGA_WFM_TOPIC_START, &wfm_streamH));
ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(wfm_streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, hide_wfmB));
if (meH){
  ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
}
if (wfm_streamH){
  ERR2(suites.StreamSuite2()->AEGP_DisposeStream(wfm_streamH));
}
```

*Tags: `params`, `ui`, `aegp`, `debugging`*

---

## Why do parameter groups not stay hidden when applying an effect preset in After Effects?

When applying an effect preset, parameter visibility flags are not preserved if only the group topic is hidden. You must explicitly hide every individual parameter within each group using AEGP_SetDynamicStreamFlag on each stream reference. Hiding only the topic without hiding each parameter will cause them to appear in the UI without their group containers. Sequence data does not affect this behavior.

*Tags: `params`, `ui`, `preset`, `aegp`*

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

## What is a good reference sample for implementing dynamic parameter visibility in After Effects plugins?

The 'Supervisor' sample in the After Effects SDK demonstrates how to dynamically hide and unhide parameters at runtime. It shows the proper use of the UPDATE_PARAMS_UI command and how to leverage AEGP API calls to inspect layer properties and adjust the UI accordingly.

*Tags: `reference`, `ui`, `params`, `aegp`, `open-source`*

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

## How can you create dependent parameters that work correctly with keyframing and expressions in After Effects plugins?

When a parameter is keyframed or uses expressions, the PF_Cmd_USER_CHANGED_PARAM callback is not called during timeline scrubbing, making it difficult to synchronize dependent parameters. The PF_ParamFlag_SUPERVISE flag works for user-initiated changes but not for time-varying data. If the dependent parameter doesn't affect rendering (PUI_ONLY flag), you can use it. Otherwise, one practical solution is to use Motion Script expressions to express dependencies between sliders and attach these expressions during sequence setup, allowing the effect to use the computed parameters during render.

```cpp
params[SLIDER2]->u.fs_d.value = params[SLIDER1]->u.fs_d.value;
params[SLIDER2]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `keyframing`, `expressions`, `scripting`, `aegp`*

---

## Why is a newly added layer not visible in the composition, showing the source layer instead on the current frame?

Adding a layer during a PF_Cmd_RENDER call causes rendering issues because the layers involved in the render are listed and checked prior to the render call. When you add a new layer during rendering, you mess with pre-made lists of render items. The solution is to add the new layer to the composition during other calls, preferably during PF_Cmd_UPDATE_PARAMS_UI or during idle process, not during render.

```cpp
ERR(suites.LayerSuite7()->AEGP_AddLayer(res_item, cResCompH, &res_layer));
```

*Tags: `aegp`, `layer-checkout`, `render-loop`, `debugging`, `params`*

---

## How can you detect when a user sets parenting on a layer in After Effects?

There is no direct event to detect parenting changes. However, you can use several workarounds: (1) Keep a hidden slider that tracks the parent layer's index and detect changes via re-render calls when the index changes, (2) Register a command hook with the 'ALL COMMANDS' flag to monitor all commands sent by AE and look for parenting-related commands, or (3) Use expressions to recursively detect parenting changes through the layer chain. A slider with expressions is a practical approach for tracking parent changes.

*Tags: `aegp_api`, `params`, `ui`, `debugging`*

---

## What is an alternative approach to directly detecting parenting changes in After Effects effects?

Instead of trying to detect parenting events directly, you can use expressions to keyframe parenting behavior. One effective method is to use layer markers' comments as parent layer names within position, rotation, and scale expressions. This allows you to simulate dynamic parenting through expressions without needing to monitor parenting change events.

*Tags: `params`, `scripting`, `ui`*

---

## How can I set a slider parameter to match the height or width of the layer the plugin is applied to?

PF_Cmd_PARAMS_SETUP occurs before the plugin is applied to the layer, so it cannot read layer parameters or even identify which layer it's on. Instead, use the PF_Cmd_UPDATE_PARAMS_UI callback that occurs afterwards to set parameter values. While the documentation recommends only cosmetic changes during this phase, you can use the Stream Suite to update parameter values programmatically rather than direct parameter array access.

*Tags: `params`, `aegp`, `effect_api`, `sdk`*

---

## What causes 'parameter count mismatch in plug-in effect' error in After Effects plugins?

This error occurs when out_data->num_params does not match the actual number of parameters defined in ParamsSetup. The num_params value must equal the last entry in your parameter enumeration (e.g., SHIFT_NUM_PARAMS), which counts all parameters including the base layer input at index 0. Additionally, each parameter must have a unique stream ID assigned for proper data persistence across plugin versions.

```cpp
enum {
    BASE = 0,           // layer input always at 0
    MY_SLIDER_1,
    MY_SLIDER_2,
    MY_POINT,
    NUM_PARAMS          // use this for out_data->num_params
};

out_data->num_params = NUM_PARAMS;
```

*Tags: `params`, `aegp`, `debugging`*

---

## How should parameter stream IDs be managed when modifying plugin parameter order across versions?

Maintain separate enumerations: one for UI order (parameter definition order) and one for disk IDs (stream identifiers). When adding or reordering parameters in a new plugin version, keep the disk ID enumeration unchanged from previous versions. This allows After Effects to correctly map saved project data to parameters even if their UI positions change. For example, if a slider had disk_id=7 in v1, keep it at 7 in v2 even if it moves to a different position in the UI enumeration.

```cpp
// UI enumeration (definition order)
enum {
    BASE = 0,
    SLIDER_1,
    SLIDER_2,
    POINT,
    NUM_PARAMS
};

// Disk ID enumeration (never change these values)
enum {
    SLIDER_1_ID = 1,
    SLIDER_2_ID = 2,
    POINT_ID = 3
};
```

*Tags: `params`, `aegp`, `deployment`*

---

## Why don't animations on a nested composition appear when querying the source composition in an AEGP plugin?

When you have a composition (compA) with animated text layers that is then nested as a layer in another composition (compB) with animation applied to the nested layer itself, you will only see the text layer animations if you query compA directly. The animation applied to the nested composition layer in compB is separate and stored with that layer in compB, not in compA. To see both animations combined, you need to query the layer in compB that contains the nested composition and retrieve its position stream using AEGP_GetNewLayerStream() with AEGP_LayerStream_POSITION.

*Tags: `aegp`, `nested_comp`, `layer-checkout`, `params`*

---

## How do you disable, enable, and hide parameter controls based on a popup selection in an After Effects plugin?

To disable a parameter, use PF_ParamFlag_SUPERVISE on the popup and handle PF_Cmd_USER_CHANGED_PARAM. In the UserChangedParam callback, disable controls by setting ui_flags |= PF_PUI_DISABLED and enable by clearing the flag with &= ~PF_PUI_DISABLED. To hide parameters, use the DynamicStreamSuite2 with AEGP_SetDynamicStreamFlag and AEGP_DynStreamFlag_HIDDEN. First get the effect handle with AEGP_GetNewEffectForEffect, then get the stream with AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag. When hiding topics, hide all parameters within the topic individually; the topic end param does not need to be hidden. Update the UI with PF_UpdateParamUI after making changes.

```cpp
case PF_Cmd_USER_CHANGED_PARAM:
  ERR(UserChangedParam(in_data, out_data, params, output, reinterpret_cast<PF_UserChangedParamExtra *>(extra)));
  break;

// In UserChangedParam:
if (extra->param_index == POPUP_MODE_ID) {
  if (params[POPUP_MODE_ID]->u.cd.value == 2) {
    params[CONTROL_ID]->ui_flags |= PF_PUI_DISABLED;
  } else {
    params[CONTROL_ID]->ui_flags &= ~PF_PUI_DISABLED;
  }
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(
    in_data->effect_ref,
    CONTROL_ID,
    params[CONTROL_ID]
  ));
}

// For hiding with DynamicStreamSuite:
AEGP_SuiteHandler suites(in_data->pica_basicP);
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, SUPER_FLAVOR, &flavor_streamH));
ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(flavor_streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, hide_themB));
```

*Tags: `params`, `ui`, `aegp`, `reference`*

---

## Is there a sample plugin that demonstrates conditional parameter visibility and disabling in After Effects?

The 'Supervisor' sample project included in the After Effects SDK demonstrates how to hide, show, and disable parameters based on user interaction. It shows the proper use of AEGP_GetNewEffectForEffect, AEGP_GetNewEffectStreamByIndex, and AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to control parameter visibility.

*Tags: `ui`, `params`, `reference`, `open-source`, `aegp`*

---

## Why can't I access parameter input layer pixel data in the DoClick function?

The incoming image data is only available during the render call, not in DoClick. However, there are several workarounds: (1) Store input/output pixels in sequence data during render for later access, though cached frames won't update; (2) Use AEGP_GetReceiptWorld() to fetch pixels of any video item at any time, though rendering composites can be slow; (3) In CS4 and earlier, use the UI drawing mechanism during the draw event; (4) Best practice: store click location and flags in sequence data during DoClick, then trigger a re-render to process pixels during the render call.

*Tags: `doclick`, `pixel`, `params`, `sequence-data`, `render-loop`, `aegp`*

---

## How should I store fixed pixel coordinates in an After Effects plugin preset?

You can store x,y coordinates using several methods: (1) Point params - simple approach, can be hidden from UI; (2) Arbitrary type params - allows custom data structures, supports undo, triggers re-render; (3) Sequence data - single instance per plugin, handles data changes manually, doesn't trigger re-render. Data is stored with the project file. Refer to the ColorGrid sample for arbitrary params and the Supervisor sample for sequence_data.

*Tags: `params`, `arb-data`, `sequence-data`, `ui`*

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

## How can I determine the bit depth and pixel format of input/output images in an After Effects plugin?

The easiest way to get the bit depth is using `extra->input->bitdepth` in the smart_render call, which returns 8, 16, or 32. Alternatively, use the PF_WorldSuite2 to call `PF_GetPixelFormat()` on the world. After Effects always uses ARGB pixel format. The pixel format enum returns values like PF_PixelFormat_ARGB128 or PF_PixelFormat_ARGB.

```cpp
PF_WorldSuite2 *wsP = NULL;
ERR(suites.Pica()->AcquireSuite(kPFWorldSuite, kPFWorldSuiteVersion2, (const void**)&wsP));
ERR(wsP->PF_GetPixelFormat(inputP, pixelFormat));
ERR(suites.Pica()->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2));
```

*Tags: `params`, `mfr`, `smart-render`, `sdk`, `reference`*

---

## How can I make my After Effects plugin support multiple bit depths instead of just 8-bit?

You cannot explicitly restrict AE to only process at specific bit depths through plugin settings. Instead, check the bitdepth at runtime using `extra->input->bitdepth`, and if the bitdepth is not supported, copy the input to output without processing and inform the user via a message overlay or alert that the effect requires a specific bit depth.

*Tags: `params`, `smart-render`, `ui`, `sdk`*

---

## How can I detect when a composition's color depth or pixel format is changed by the user?

There is no direct event for color depth changes, but the frame is re-rendered when the comp depth is changed. You can check the current pixel format using PF_GetPixelFormat() during the render call. To update parameter UI in response, use PF_UpdateParamUI during the render call, and you can hide/unhide parameters at any time using AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN. Parameter property changes can be made when receiving events or PARAMS_UI_CHANGED.

*Tags: `params`, `ui`, `aegp`, `pixel_format`, `bit_depth`*

---

## How can I get a Pixel Bender kernel's match name to use in an After Effects native plugin?

Create a new After Effects project with a composition containing a single solid layer. Apply your Pixel Bender plugin to the layer. Then run this ExtendScript to retrieve the match name: alert(app.project.item(1).layer(1).effect(1).matchName); This will return the match name (typically based on the id data in the Pixel Bender script) that you can use in your native plugin to make After Effects treat both as the same plugin for backward compatibility.

```cpp
alert(app.project.item(1).layer(1).effect(1).matchName);
```

*Tags: `scripting`, `params`, `reference`, `pipl`*

---

## What parameter type should be used to replace Pixel Bender percent sliders when porting to the After Effects SDK?

Use PF_ADD_FLOAT_SLIDERX instead of PF_ADD_PERCENT, since Pixel Bender implements percent sliders as float sliders. You can add the PF_ValueDisplayFlag_PERCENT flag to the PF_ADD_FLOAT_SLIDER call to display the float slider as a percentage in the UI without changing the underlying implementation.

```cpp
PF_ADD_FLOAT_SLIDERX("param_name", 0, 100, 0, 100, 1, PF_ValueDisplayFlag_PERCENT);
```

*Tags: `params`, `ui`, `pipl`*

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

## Is there an API to draw standard After Effects controls like buttons and checkboxes outside the Effect Controls Window?

No, there is no API available to draw standard AE controls outside the ECW (Effect Controls Window). Effect controls can only be created during param_setup in effects and will display only in the ECW and timeline. Custom controls would need to be reimplemented with custom drawing code.

*Tags: `ui`, `params`, `aegp`, `reference`*

---

## How can I prevent PF_Param_POINT from intercepting mouse click and drag events near custom UI controls?

The issue is that After Effects' interface takes higher priority over plug-in interface and intercepts drag events. The solution is to disable the point parameter UI when needed by setting the PF_PUI_DISABLED flag using PF_UpdateParamUI. This makes the point parameter unselectable and prevents it from intercepting clicks and drags. You can set this flag selectively when the cursor is close to your custom control or inside the comp window.

```cpp
def.ui_flags = PF_PUI_DISABLED;
suites.ParamUtilsSuite1()->PF_UpdateParamUI(in_data->effect_ref, paramIndex, &def);
```

*Tags: `ui`, `params`, `debugging`*

---

## Why does calling PF_ForceRerender() on a text layer result in 'layer does not have a source' error?

Text layers and shape layers do not have a source, unlike solids. Any source-related operation performed on a text layer will generate this error. As a workaround, instead of using PF_ForceRerender(), you can add a dummy parameter to the effect and call AEGP_SetStreamValue() to trigger rendering of all layers with the effect.

*Tags: `aegp`, `text-layer`, `params`, `debugging`*

---

## How do you create parameter groups in After Effects plugins to organize the UI?

Use the PF_ADD_TOPIC and PF_END_TOPIC macros to create parameter groups. First, define enum IDs for your topic and end topic in your header file. Then in your parameter setup, use PF_ADD_TOPIC with a group name and ID, add your parameters (checkboxes, points, etc.) between them, and close with PF_END_TOPIC using the corresponding end ID. This organizes related parameters into collapsible groups in the Effect Controls panel.

```cpp
// In .h file
enum {
  TOPIC_FEEDBACK_ID,
  SHOW_HISTOGRAM_DISK_ID,
  HIST_CORNER_ID,
  HIST_SIZE_ID,
  END_TOPIC_FEEDBACK_ID
};

// In .cpp file
PF_ADD_TOPIC("Histogram Overlay", TOPIC_FEEDBACK_ID);
AEFX_CLR_STRUCT(def);
def.flags = PF_ParamFlag_CANNOT_TIME_VARY;
PF_ADD_CHECKBOX(STR(StrID_ShowHist_Name), STR(StrID_ShowHist_Description), FALSE, 0, SHOW_HISTOGRAM_DISK_ID);
AEFX_CLR_STRUCT(def);
PF_ADD_POINT("Histogram location", HIST_LOCATION_X, HIST_LOCATION_Y, RESTRICT_BOUNDS, HIST_CORNER_ID);
AEFX_CLR_STRUCT(def);
PF_ADD_PERCENT("Histogram width", HIST_PCT_DFLT, HIST_SIZE_ID);
AEFX_CLR_STRUCT(def);
PF_END_TOPIC(END_TOPIC_FEEDBACK_ID);
```

*Tags: `params`, `ui`, `pipl`*

---

## How can I access the composited color of layers beneath the current layer in After Effects?

There is no direct API to access intermediate composition buffers during normal effect rendering, as After Effects does not render bottom-to-top and optimizes by skipping rendering of fully opaque regions. However, several workarounds exist: (1) Write an AEGP artisan-type plugin (see the 'arti' sample) which handles composition rendering and has access to intermediate buffers, though this is very difficult; (2) Create a duplicate composition with only needed layers and render it using AEGP_GetReceiptWorld(); (3) Apply your effect on an adjustment layer, which receives the composited buffer of underlying layers, then use checked-out layer params to access original sources; (4) Use sampleImage() expressions on hidden parameters to sample pixel data, though this is slow and limited; (5) Ask the user to provide the background as a layer parameter, a common practice in effects that need background information.

*Tags: `aegp`, `render-loop`, `layer-checkout`, `sdk`, `params`*

---

## How do you retrieve shape layer content parameters (like Polystar, ZigZag, Repeater) in AEGP?

Shape layer contents are not accessed the same way as effects. For path data, use PF_PathDataSuite1 to retrieve path vertices. For rendered pixels, you cannot directly read the source since shape layers don't have one. As an alternative, you can get the shape and render it yourself, or duplicate the composition, delete everything except the shape layer, and use AEGP_GetReceiptWorld() to render that new composition. The specific method was discussed in detail in an Adobe forum thread.

*Tags: `aegp`, `shape-layer`, `params`, `reference`*

---

## What is the correct AEGP API method to access shape layer contents versus effects?

To get layer effects, use AEGP_GetLayerEffectByIndex. However, shape layer contents (like Polystar, ZigZag, Repeater) are not accessed through the stream suite or effect methods. Instead, use PF_PathDataSuite1 for path geometry, or employ workarounds such as rendering the shape yourself or using AEGP_GetReceiptWorld() on a duplicate composition containing only the shape layer.

*Tags: `aegp`, `shape-layer`, `params`, `debugging`*

---

## How can you create dynamic stroke parameters similar to the Paint effect in After Effects plugins?

Strokes are handled by the dynamicStreamSuite, which allows you to add, remove, and alter strokes and text layer parameters. However, custom parameters that behave like strokes cannot be created through the API. Two workarounds are: (1) Create your effect with hidden parameter groups (e.g., 50 groups, each with width, length, start, and end sliders) and unhide them as needed—this is simpler but limited by the number of compiled groups. (2) Create a dummy effect with the desired sliders and add new instances of it to the layer whenever more strokes are needed—this is more flexible and allows unlimited strokes but is significantly more complex to implement.

*Tags: `params`, `ui`, `sdk`, `aegp`, `pipl`*

---

## Can you create custom stream group types in After Effects plugins beyond existing types?

No, you can only add stream groups and atoms of existing types through the API. You cannot create custom types of your own. Any custom parameter grouping behavior must be implemented using workarounds such as hidden parameter groups or multiple plugin instances.

*Tags: `params`, `sdk`, `aegp`*

---

## How do you use PF_AppGetColor() to retrieve After Effects application colors in a custom effect?

Use the AppSuite3() to call PF_AppGetColor() with the desired color type and a pointer to store the result. The syntax is: suites.AppSuite3()->PF_AppGetColor(PF_App_Color_HOT_TEXT, &appColor1); where PF_App_Color_HOT_TEXT is the color type you want to retrieve and appColor1 is the variable that will store the retrieved color.

```cpp
suites.AppSuite3()->PF_AppGetColor(PF_App_Color_HOT_TEXT, &appColor1);
```

*Tags: `ui`, `aegp`, `params`, `reference`*

---

## How can I force a render on a custom effect from AEGP without adding the action to the undo/redo stack?

Create an undo group with a NULL instead of a name. Anything that happens in that group will NOT be added to the undo stack. Alternatively, you can use AEGP_SetStreamValue() to change an invisible parameter's value, which will trigger a render without affecting the result, but this approach will add the action to the undo/redo stack unless wrapped in a NULL undo group.

*Tags: `aegp`, `undo`, `render-loop`, `params`*

---

## What is the correct way to trigger a force render on a custom effect from AEGP?

You can use AEGP_SetStreamValue() to modify an invisible parameter value, which will trigger a render without affecting the effect output. Note that setting PF_OutFlag_Force_Rerender in PF_CMD_COMPLETELY_GENERAL does not work for triggering renders from AEGP.

*Tags: `aegp`, `params`, `render-loop`*

---

## When duplicating a layer with custom effects, which effect receives the PF_Cmd_SEQUENCE_FLATTEN command—the old or new effect?

The FLATTEN command is sent only to the old effect. It tells the effect to collect and prepare all its data for storing. Both the old and new effects receive the UNFLATTEN command afterward. To distinguish between old and new effects during UNFLATTEN, store identification data during FLATTEN (such as effect index, layer ID, and composition item ID), then check these identifiers during UNFLATTEN or the first updateParamsUI call to determine which effect is which.

*Tags: `sequence-data`, `params`, `aegp`*

---

## How do I distinguish between the original and duplicated effect when both have identical sequence data after duplication?

Store identifying information about the effect during the FLATTEN command, such as its effect index, layer ID, and composition item ID. During the UNFLATTEN command (or preferably at the first updateParamsUI call), compare the current effect's properties against the stored identification data. If the properties match, it is the old effect; if any differ, it is the new duplicated effect.

*Tags: `sequence-data`, `params`, `aegp`*

---

## How can I set an effect parameter value on a layer in an After Effects plugin?

Effect parameters in After Effects are referred to as streams. Use the AEGP_SetStreamValue() function to set parameter values. To determine which stream index corresponds to which parameter, you need to experiment with different effects and learn their stream numbers. If you're applying effects you've written yourself, you should know the parameter index numbers directly.

*Tags: `aegp`, `params`, `sdk`*

---

## How can I prevent a slider parameter from resetting to its default value when the Effect Reset button is clicked?

You cannot tell a parameter to ignore the reset operation directly. However, you can store the parameter value in sequence data and restore it when the SEQUENCE_RESETUP callback is triggered. Alternatively, if the parameter is used as a unique identifier, you can skip storing it as a parameter altogether and maintain it only in sequence data, retrieving it via AEGP_EffectCallGeneric() when needed. You could also try setting a different default value during the param_setup call, though param_setup is typically called only once when the effect is first applied.

*Tags: `params`, `sequence-data`, `aegp`, `sdk`*

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

## Can I draw custom graphics and keyframe indicators on the After Effects TimeGraph?

No, the After Effects SDK does not provide buffers or APIs for drawing on the TimeGraph or timeline. You can only draw into buffers that AE provides for the composition window, layer window, and effects controls window. Custom parameter UIs created in the Effects Controls Window will not appear in the timeline. As an alternative, you can create a custom UI using an arbitrary data type parameter that displays a synchronized alternative timeline view centered on the current time cursor, allowing users to edit your special keyframes with custom interpolation behavior.

*Tags: `ui`, `params`, `arb-data`, `aegp`, `sdk`*

---

## How do you properly use the transform_world() function to scale and rotate a PF_LayerDef?

To use transform_world() effectively: (1) Pass NULL for mask_world0 to use the whole frame. (2) You can pass one or two matrices—two matrices enable motion blur between transformations. Use syntax (&matrix1, &matrix2) for two matrices or &matrix1 for one, and specify the matrix count. (3) Note that AE's 3x3 matrices have swapped x and y compared to standard math notation—what should be matrix[1][2] goes in matrix[2][1]. See the CCU sample for identity and transformation matrix functions. (4) Always pass a full-frame rectangle initially; transform_world() does not accept NULL for the destination rectangle. The offset should be embedded in the matrix itself, not assumed from the rectangle's top-left corner.

*Tags: `mfr`, `params`, `matrix`, `transform`, `reference`*

---
