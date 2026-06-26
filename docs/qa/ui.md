# Q&A: ui

**265 entries** tagged with `ui`.

---

## Can you disable the crosshair/visual representation of point parameters in the AE comp window?

No, the crosshairs are the visual representation of point parameters and cannot be disabled. As a workaround, you could use a different parameter type like an int or float param pair for x and y values instead of a point parameter.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2025-12-29 · Tags: `point-parameters`, `crosshair`, `comp-window`, `ui`, `workaround`*

---

## How can a C++ plugin display toast notifications next to the mouse cursor in After Effects?

This can be achieved with a C++ plugin working alongside CEP extensions. Scott Black's Control Groups extension uses this technique to display toast-like notifications that appear alongside the mouse cursor. This is a more heavy-handed solution if your only goal is toast notifications, but useful if you already have a C++ plugin component.

*Contributors: [**Scott Black**](../contributors/scott-black/) · Source: aescripts discord · 2025-12-05 · Tags: `toast-notification`, `ui`, `cep`, `cpp-plugin`, `mouse-cursor`*

---

## How do you hide a banner in the timeline while showing it in the Effect Control Window?

The banner needs to be associated with a keyframeable parameter to show in both places like the Color Grid plugin does. For a static banner without keyframing, you need to configure it to only display in the ECW and not in the timeline.

*Tags: `ui`, `params`, `aegp`*

---

## What causes an infinite progress bar at the end of a caching effect?

An infinite progress bar look can be triggered by an incoherent PF_Progress value. Give it impossible values like 200/100 and it should display this infinite progress behavior.

*Tags: `ui`, `debugging`, `caching`*

---

## Is it possible to set or change the value of a LAYER_PARAM in the plugin UI?

Yes, you can set the layer ID value in a streamVal2 using SetStreamValue.

*Tags: `params`, `ui`, `aegp`*

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

## Can PF_ADD_ARBITRARY2 be used with PF_PUI_INVISIBLE flag?

PF_ADD_ARBITRARY2 can be used with PF_PUI_INVISIBLE, but there are specific requirements for the ARB param setup. The ARB needs special functions implemented correctly, similar to the ColorGrid example in the SDK.

*Tags: `arb-data`, `params`, `ui`*

---

## Can a CEP panel change parameter values of a C++ plugin?

CEP cannot directly change plugin parameter values. Parameter changes must be made through the plugin's own UI or other indirect mechanisms.

*Tags: `cep`, `params`, `ui`*

---

## Can a C++ plugin detect if a CEP panel is open or bring it into focus?

A C++ plugin cannot directly check if a CEP panel is open or bring it into focus through standard plugin APIs.

*Tags: `cep`, `ui`, `plugin-communication`*

---

## How can a C++ plugin open a CEP panel?

A C++ plugin can open a CEP panel by using ExecuteScript to find the Command ID of the extension by name, then executing that command through the execute command suite.

*Tags: `cep`, `scripting`, `ui`*

---

## How can a slider value be made to slide slower or with more precision without holding cmd/ctrl?

You can click on the slider and enter the value directly. If you control the code, you can use a UI flag such as PF_Precision_HUNDREDTHS or similar precision flags to control slider precision.

*Tags: `ui`, `params`*

---

## How can I make a slider value change more slowly or with finer precision without requiring modifier keys?

You can hold cmd/ctrl while dragging to slide slower, but there may be other approaches depending on your specific UI implementation. Consider implementing custom slider behavior with adjustable sensitivity or step values in your plugin's parameter interface.

*Tags: `ui`, `params`*

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

## How can you enable or toggle a slider parameter to be collapsed in After Effects?

Slider parameters cannot be collapsed individually. The PF_ParamFlag_COLLAPSE_TWIRLY flag only works with parameter groups (topics), not individual slider parameters. To collapse parameters, you need to use PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG on a group/topic, not on individual sliders.

```cpp
def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;
```

*Tags: `params`, `ui`, `pipl`*

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

## How do you disable the annoying popup that appeared in After Effects 2024?

Use CMD+F12 to open the console. Click the hamburger menu next to the console and change to Debug Database. Search for 'ShowPreviousCrashWarning' and turn it off. Alternatively, the setting can be modified in the prefs.txt file.

*Tags: `debugging`, `macos`, `ui`*

---

## How should a plugin handle GPU out-of-memory errors and warn users appropriately?

When a plugin runs out of GPU memory, it should return PF_Err_OUT_OF_MEMORY. During MFR (Multi-Frame Rendering) queue renders, After Effects may retry with lower thread counts. However, for interactive scrubbing in the composition, After Effects may not warn users and they see black frames instead. The challenge is that there's no standard way to distinguish between queue renders and interactive scrubbing to conditionally show warning messages in out_data without failing the render. A workaround would be to return the error code and accept that After Effects' handling may be limited, or document the resolution and bitdepth requirements for users to manually adjust.

*Tags: `gpu`, `memory`, `mfr`, `debugging`, `ui`*

---

## What is a pseudoeffect and how is it used?

A pseudoeffect is a nicer alternative to using standard expression control effects like Slider Control or Point Control. It is written in XML and creates a regular-looking effect parameter UI but doesn't actually do anything—it just acts as controls that can be read from expressions, making the interface look more professional than having multiple Slider Controls.

*Tags: `params`, `ui`, `scripting`*

---

## When is PF_Cmd_GLOBAL_SETUP called in After Effects and Premiere?

In After Effects, GLOBAL_SETUP is called the first time the user applies the effect plugin to a layer. In Premiere, it is called when the application is loading.

*Tags: `aegp`, `pipl`, `premiere`, `ui`*

---

## Is it possible to show a dialog during Global Setup?

It should be possible to show any dialog during Global Setup, though there is some uncertainty about whether dialogs can be displayed during the initial AE loading screen when plugins are being scanned.

*Tags: `ui`, `aegp`, `pipl`*

---

## Is it possible to make dropdown options bold in an After Effects effect?

No, standard text formatting syntax like **bold**, *bold*, and <bold>bold</bold> do not work for dropdown options in After Effects effects.

*Tags: `ui`, `params`*

---

## What causes slowness when displaying a timeline with many layers in After Effects?

The slowness is primarily in the UI rendering rather than the plugin rendering itself. The bottleneck is likely in After Effects internal calls such as 'get layer info' and 'get layer sprite'. Even with many layers hidden by default, performance issues persist. After Effects also has poor memory management, making it inefficient with large numbers of layers (10k+).

*Tags: `ui`, `memory`, `performance`, `aegp`*

---

## What is the purpose of the changedB parameter in PathOutline?

The changedB parameter appears to be a flag that indicates whether path data has been modified, though the exact public API for modifying path data is not exposed in the SDK. Setting it to TRUE may signal to After Effects that path data needs to be updated in the UI, but the internal APIs used by After Effects to modify path data are not publicly available.

*Tags: `params`, `ui`, `aegp`*

---

## What is causing custom UI arb parameters to glitch in After Effects 2025?

The issue is not directly related to arb parameters, but rather to drawing image buffers using the DrawBot suite. The problem occurs when the UI composites multiple images per plugin context. The culprit is likely NewImageFromBuffer, which on every call probably sets the pixels of all retained DRAWBOT_ImageRef objects within the same context, causing flashes or scrambled/duplicated UI. The issue can be avoided if the UI either doesn't composite an image at all or only composites a single image per plugin context.

*Tags: `ui`, `arb-data`, `debugging`*

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

## Is there a method for forcing masks on the layer to be applied after the plugin processes?

There is no built-in method to force masks to be applied after plugin processing. As a workaround, you can have the user select the mask type to None and compute the alpha yourself within your plugin, though this requires additional implementation effort.

*Tags: `params`, `ui`, `render-loop`*

---

## How do you create a render-only version of a plugin that doesn't require additional licenses for render farms?

One approach is to make all parameters invisible while still allowing the plugin to read them, creating a read-only version of the plugin that can process renders without displaying the GUI interface.

*Tags: `deployment`, `ui`, `render-loop`, `pipl`*

---

## How can internally computed position values from a plugin render be exposed to users for use in expressions or as parameters?

The recommended approach is to output the internally computed values through a point control parameter. This allows users to access the generated positions either in expressions or by exporting them as nulls. The answer suggests this is a viable solution, with a reference to similar functionality being planned for the Vision plugin on aescripts.com.

*Tags: `params`, `ui`, `scripting`, `arb-data`*

---

## How can a C++ plugin update a text layer every frame after rendering completes?

This is not cleanly possible due to AE's project locking during render operations. Possible workarounds include: (1) writing data to a compute cache or sequence data and reading it from the UI thread via the aegp_idle hook, (2) using AEGP_SetText from the UI thread instead of the render thread, (3) rendering text yourself in the render thread using an external library, or (4) saving to disk and having a ScriptUI panel with a timer read the data. The aegp_idle hook approach may not work without a GUI or in MediaEncoder. Setting text from the render thread causes a locked project error.

*Tags: `aegp`, `render-loop`, `threading`, `ui`, `sequence-data`, `compute-cache`*

---

## Would the aegp_idle hook approach work when rendering through MediaEncoder or without a GUI?

No, the aegp_idle hook approach would not work in MediaEncoder or without a GUI, since the idle hook requires an active UI thread to execute.

*Tags: `aegp`, `threading`, `ui`, `deployment`*

---

## What is a workaround for the broken Kerning API in ExtendScript for text layers?

The Kerning API works well in expression script but is broken in ExtendScript. A workaround is to write an expression on a text layer that writes a value to a separate text layer, then read the value from that second layer back in ExtendScript.

*Tags: `scripting`, `ui`*

---

## What is a method to set source text dynamically in After Effects using rendered data?

Encode and render dead pixels in your render output, then use a sampleImage expression to retrieve those pixels which can then set the source text. This method mostly works with a few caveats and also works with Null objects.

*Tags: `scripting`, `ui`, `render-loop`*

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

## Is it possible to achieve progressive rendering in an After Effects plugin that continuously shows results to the user as more samples are computed?

The conversation acknowledges the desire for progressive rendering where samples accumulate and display over time rather than waiting for all samples to complete before showing results. While the technical feasibility is not explicitly detailed in the response, the user notes they have this working outside of After Effects and finds it more interactive. The challenge with current AE plugin behavior is that it waits for complete render before display, making it feel slow compared to progressive sampling approaches.

*Tags: `smart-render`, `render-loop`, `ui`, `preview`*

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

## How do you disable the crash warning dialog in After Effects 2026 on macOS?

Use cmd + F12 to switch to database view, then search for "crash window" or similar to locate and disable the crash warning.

*Tags: `macos`, `debugging`, `ui`*

---

## How do you properly initialize ImGui with OpenGL on macOS?

When initializing ImGui with OpenGL on macOS, window flags must be set before window creation, not after. This differs from Windows behavior and is a common mistake. The OpenGL loader initialization fails if flags are set in the wrong order during the setup process.

*Tags: `macos`, `opengl`, `ui`, `debugging`*

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

## What is the best documentation to get started with UXP for Premiere Pro?

According to the discussion, there is a comprehensive article that contains all the links for UXP documentation, forums, and other resources for Premiere Pro development. UXP in Premiere Pro is currently limited but will be expanded soon.

*Tags: `premiere`, `uxp`, `ui`, `reference`*

---

## How do you hide an arbitrary parameter banner in the timeline while keeping it visible in the Effect Control Window?

When working with arbitrary parameters in After Effects, the visibility in the timeline versus the Effect Control Window (ECW) is tied to whether the parameter is keyframeable. Static banners that don't need keyframing can be hidden from the timeline while remaining visible in the ECW by ensuring they are not set up as keyframeable streams. The Color Grid effect demonstrates having parameters visible in both locations because they are keyframeable, but for static-only parameters, the visibility behavior differs.

*Tags: `arb-data`, `ui`, `params`, `debugging`*

---

## What causes an infinite progress bar at the end of an effect render?

An infinite progress bar can be triggered by an incoherent PF_Progress value. Setting impossible values like 200/100 (where the current progress exceeds the total) will display this infinite progress bar effect.

*Tags: `ui`, `debugging`, `aegp`*

---

## Is it possible to set or change the value of a LAYER_PARAM in a plugin UI?

Yes, you can set/change the value of a LAYER_PARAM in your plugin UI by setting the layer ID value in a streamVal2 using SetStreamValue.

*Tags: `params`, `ui`, `aegp`*

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

## How do you reproduce warp stabilizer banner messages in the viewport—using the Canvas suite or drawing directly on the rendered frame?

The user is asking about the proper approach to display banner messages similar to the warp stabilizer effect. This involves choosing between using the Canvas suite for viewport drawing versus drawing directly on the rendered frame output.

*Tags: `ui`, `render-loop`, `aegp`, `output-rect`*

---

## What drawing libraries or tools are recommended for After Effects plugin development?

Cairo and DrawBot are recommended options for drawing in After Effects plugins. The canvas suite should not be confused with a drawing toolkit—it actually refers to the rendering pipeline rather than a suite for drawing on the viewport.

*Tags: `ui`, `tool`, `reference`*

---

## How can you obtain a drawing reference in After Effects plugin render loops without requiring user interaction?

The standard approach using EffectCustomUISuite1()->PF_GetDrawingReference() only provides a drawing context during PF_Cmd_EVENT callbacks triggered by user interaction. To draw during render without user interaction (similar to VFX Stabilizer behavior), you need an alternative method to access the drawing reference that doesn't depend on the extraPV parameter passed through PF_EventExtra. This requires investigating lower-level drawing APIs or background rendering approaches that don't rely on event-driven context delivery.

*Tags: `ui`, `render-loop`, `aegp`, `debugging`*

---

## How can you draw to the composition window during rendering without relying on user interaction events?

The challenge is that PF_GetDrawingReference() is only available through PF_EventExtra during PF_Cmd_EVENT, which requires user interaction with the effect parameters or composition preview. To draw during a render loop without user interaction (similar to VFX Stabilizer behavior), you need an alternative approach to obtain the drawing reference that isn't tied to user interaction events. This is a difficult problem with limited documented solutions; consulting experienced plugin developers like Shachar Gran is recommended.

*Tags: `ui`, `render-loop`, `aegp`, `debugging`*

---

## Why is the paramUI suite slower when changing a parameter name in After Effects 2023?

The paramUI suite has been observed to be slower when changing a parameter name in After Effects 2023 (version 23.2.1). An alternative approach using the streamSuite has been found to work well as a workaround for this performance issue.

*Tags: `params`, `ui`, `performance`, `aegp`*

---

## Where can I find discussion and solutions about PF_PROGRESS bar not showing in After Effects plugins?

Adobe Community has a discussion thread about PF_PROGRESS bar issues: https://community.adobe.com/t5/after-effects-discussions/progress-bar-won-t-show-using-pf-progress/m-p/10474224. The thread discusses limitations of progress reporting in the render function, particularly when the main thread is blocked during computation.

*Tags: `ui`, `reference`, `debugging`*

---

## Can arbitrary parameters with PF_PUI_INVISIBLE flag be used to store custom data in After Effects plugins?

Yes, arbitrary (Arb) parameters can be used to store custom data invisibly in plugin parameters. However, there are important constraints: the data structure must be serializable and cannot contain C++ std::string types. Instead, use fixed-size char arrays (char[size]) for string data, as std::string cannot be flattened or saved to disk. Refer to the ColorGrid example in the SDK for proper Arb parameter setup including special functions for initialization and updates.

*Tags: `arb-data`, `params`, `ui`, `reference`*

---

## Is there a way to communicate between a CEP panel and a C++ plugin in After Effects?

Direct communication between CEP and C++ plugins is not possible. However, indirect communication can be achieved: a C++ plugin can open a CEP panel by using the ExecuteScript to find the Command ID (menu) by the extension name, then executing that command through the command suite. CEP cannot directly change plugin parameter values, and the plugin cannot directly check if a CEP panel is open or bring it into focus.

*Tags: `cep`, `c++`, `scripting`, `ui`, `aegp`*

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

## How should DrawImage be called in drawbotSuites to ensure correct opacity and alpha rendering?

When using drawbotSuites.surface_suiteP->DrawImage, you must use an opacity of 1.0. If you don't use 1.0 opacity, the drawing will appear darker than intended and the alpha will still show as 1.0 since After Effects' UI will be behind it with solid alpha. This is important for correct compositing behavior.

```cpp
drawbotSuites.surface_suiteP->DrawImage(/* ... */, 1.0 /* opacity */, /* ... */)
```

*Tags: `ui`, `drawing`, `aegp`, `debugging`*

---

## Why does text appear faint when using FillPath and StrokePath in After Effects plugin UI?

When writing on UI using FillPath and StrokePath with opacity set to 1.0, faint text may result from After Effects UI brightness preferences or project color space compensation. The After Effects suite may automatically compensate for these settings, affecting how drawing operations are rendered on the UI.

*Tags: `ui`, `debugging`, `aegp`*

---

## How can you implement a custom text editor in the Effect Control Window that captures key events like Tab?

When implementing a custom text editor in the ECW using Custom UI events, PF_Event_KEYDOWN events are available but don't include an index for the particular param. Additionally, some keys like Tab don't reach the KEYDOWN handler. To solve this, you need to claim key-focus similar to how built-in slider numeric value editing works, which allows the plugin to intercept all keyboard input including Tab and other special keys that would normally be consumed by the host.

*Tags: `ui`, `custom-ui`, `aegp`, `debugging`*

---

## How can you implement a button in an After Effects plugin that triggers a script and passes data to it?

Use an arbitrary data parameter to store text/script content, then trigger AEGP_ExecuteScript() via a button event to execute the script string. You can manipulate the script string (e.g., via find-and-replace) before execution to pass data into it, and the script can return data back to After Effects.

*Tags: `aegp`, `arb-data`, `ui`, `scripting`*

---

## How can you collapse a slider parameter in After Effects plugins?

To collapse a slider parameter, you need to set the global flag PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG and then set the param flag with 'def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;'. However, this approach may not work for individual sliders in the same way it works for topic groups.

```cpp
def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## What is the best approach to restrict two float sliders so that one cannot be equal to or greater than the other?

Handle parameter changes in the UserChangedParams callback. When one slider changes, check the value of both sliders and apply corrective values to maintain the constraint. However, avoid updating parameter min/max values dynamically while sliding, as this can cause issues with keyframing—keyframes with previously acceptable values can become unmutable if they later violate the constraint. A better approach is to treat both parameters as interchangeable "endpoints" and use min() and max() in your code to determine which is Start and which is End, rather than enforcing strict ordering on the parameters themselves.

*Tags: `params`, `ui`, `scripting`*

---

## How do you disable the crash warning popup in After Effects 2024?

Press CMD+F12 to open the Debug Database console. Click the hamburger menu next to the console and change to Debug Database. Search for 'ShowPreviousCrashWarning' and turn it off. This setting can also be configured in the prefs.txt file and can be scripted for automation.

*Tags: `debugging`, `macos`, `scripting`, `ui`*

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

## How can you create a custom banner in the After Effects effects menu like FxFactory plugins do?

This appears to be a FxFactory-specific feature. The discussion suggests several investigation approaches: comparing AE installation folders before and after FxFactory installation to identify changes, examining the .aex plugin file and its PiPL resources for unknown SDK functions, or contacting FxFactory developers directly. It's unclear whether the banner is drawn through standard AEGP suite functions or through custom modifications to the plugin manager.

*Tags: `pipl`, `ui`, `reverse-engineering`, `aegp`, `debugging`*

---

## What is a pseudoeffect in After Effects plugin development?

A pseudoeffect is a nicer alternative to using standard expression control effects like Slider Control or Point Control. Pseudoeffects are written in XML and create a regular-looking effect parameter UI without actually performing any rendering—they function purely as control elements that can be read from expressions. They provide a more professional appearance than exposing multiple Slider Controls directly to users.

*Tags: `ui`, `params`, `scripting`*

---

## Can you display dialogs during the Global Setup phase of a plugin?

According to discussions in the community, it should be possible to show dialogs during Global Setup. However, there is some uncertainty about whether Global Setup is called during the AE loading screen when scanning plugins, or only when the effect is first applied. The capability exists, but the exact execution context may affect dialog behavior.

*Tags: `ui`, `pipl`, `aegp`, `debugging`*

---

## How can you implement live drawing and UI interactions like rotoscope in After Effects plugins?

Live drawing and UI interactions such as rotoscope functionality in After Effects plugins are implemented through the PF_CMD_EVENT command, which allows plugins to handle real-time user input and interactive drawing on the canvas.

*Tags: `ui`, `aegp`, `interactive`, `drawing`, `events`*

---

## What are the common performance bottlenecks when working with large numbers of layers in After Effects plugins?

Performance issues with large layer counts (800+ layers) typically stem from UI rendering rather than the render engine itself. The bottleneck often comes from repeated After Effects API calls like 'get layer info' and 'get layer sprite', not from Artisan optimization. Additionally, After Effects has poor memory management with large layer counts, making it inefficient to handle thousands of layers.

*Tags: `aegp`, `ui`, `memory`, `performance`, `debugging`*

---

## Can hiding layers by default improve performance when dealing with many layers in After Effects?

Hiding layers by default can be an effective optimization strategy to reduce the performance load on the UI and API calls, though the specific performance gains depend on the implementation and number of layers being processed.

*Tags: `ui`, `performance`, `optimization`, `layer-checkout`*

---

## How can you reliably assign a persistent ID to an effect instance that survives effect reordering?

One approach is to hash the layer ID, composition ID, and index of the effect together. However, this has the limitation that if the effect's index changes (e.g., when effects are reordered), the ID must be recalculated. An alternative suggestion from Adobe forums is to store a copy of the ID in a hidden arbitrary parameter and use the UI/ARB thread to update the value from sequence data, though this approach has not been thoroughly tested and may not be intuitive to implement.

*Tags: `arb-data`, `params`, `ui`, `aegp`*

---

## Does the 'Blend colors using 1.0 gamma' checkbox affect how a composition composites onto its background color?

No. The 'Blend colors using 1.0 gamma' checkbox only affects blending modes between layers. It does not affect the final composite of a composition onto the composition's background color, because that compositing step does not respect layer blending modes. For example, if a layer's blend mode is set to 'Overlay', that blending mode is only visible when compositing against another layer below it—it does not apply when compositing the composition onto its background color.

*Tags: `ui`, `blending`, `compositing`, `color-space`*

---

## What causes custom UI artifacts and glitching in After Effects 2025 when using DrawBot with arbitrary data parameters?

The issue is not directly related to arbitrary data (arb) parameters, but rather to how DrawBot's Image buffer compositing works. The problem occurs when the UI composites multiple images per plugin context. The culprit appears to be NewImageFromBuffer, which on every call sets the pixels of all retained DRAWBOT_ImageRef objects within the same context, causing visual artifacts like flashes, scrambled, or duplicated UI elements. The issue can be avoided by either not compositing any image in the UI, or by only compositing a single image per plugin context.

*Tags: `ui`, `arb-data`, `drawbot`, `ae-2025`, `debugging`*

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

## What causes the 'Cannot run a script while a modal dialog is waiting for response' error when calling scripting from a plugin?

This error occurs when attempting to run scripting from a plugin while a modal dialog is active and waiting for user response. It is a limitation of using scripting from plugins - the scripting engine cannot execute while After Effects has a modal dialog blocking the main thread. This can happen even after checking if scripting is available first, and may occur during parameter setup.

*Tags: `scripting`, `aegp`, `debugging`, `ui`*

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

## How can you find and use the 'Reveal in Composition' command ID in After Effects scripting?

The 'Reveal in Composition' command ID can be found using app.findMenuCommandId('Reveal in Composition'), which returns 2775. However, this command may not work in all contexts and might return null if the menu is just a category menu rather than an executable command. An alternative approach is to mimic the behavior manually by opening the desired composition and selecting the desired layer, which often produces better results than relying on the command ID.

```cpp
app.findMenuCommandId('Reveal in Composition'); /// 2775
```

*Tags: `scripting`, `ui`, `aegp`, `reference`*

---

## What is the command ID for 'Reveal in Timeline' in After Effects?

The 'Reveal in Timeline' command ID is 2536, though it may require specific context to work properly or may not function as expected in all scenarios. Users attempting to use this command should be aware that it may need particular conditions to be met or may behave differently than anticipated.

*Tags: `scripting`, `ui`, `aegp`, `reference`*

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

## How do you create collapsible parameter groups in After Effects plugins?

To create collapsible parameter groups, set the global PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG flag. Then for each individual parameter, set PF_ParamFlag_START_COLLAPSED if that parameter should be collapsed by default, or omit the flag if it should not be collapsed.

*Tags: `params`, `ui`, `pipl`*

---

## How can you create a render-only version of an After Effects plugin that doesn't require a GUI license on render farms?

One approach is to make all parameters invisible while still allowing the plugin to read them, effectively creating a read-only version of the plugin. This way, the plugin can process parameters set during the authoring phase without requiring interactive UI on render farm machines.

*Tags: `params`, `ui`, `deployment`, `render-loop`*

---

## What UI approach do some AE plugins use to improve interactivity with parameters?

Some plugins like Gifgun and Datamosh use an external application window (often built with C++ and ImGui) that allows users to adjust parameters interactively in a fullscreen UI, then pre-render and send results back to After Effects. This avoids the need for re-rendering on every parameter change within the comp, which would be required if using a custom UI directly in After Effects like Optical Flares does.

*Tags: `ui`, `workflow`, `sequence-data`, `open-source`*

---

## How can internally computed positions from a plugin be exposed to users in After Effects?

Internally computed positions can be exposed to users by outputting them through a point control parameter. This allows users to index the values in expressions or access them programmatically. One practical example is exporting bounding boxes as nulls, which is a technique used in projects like Vision (https://aescripts.com/vision).

*Tags: `params`, `ui`, `scripting`, `expressions`, `arb-data`*

---

## How can internally computed positions generated during plugin rendering be exposed to users for use in expressions or parameters?

When a plugin generates its own set of positions as part of the render process (rather than reading from an existing AE path), those values can potentially be exposed to users in two ways: (1) by indexing them in expressions, allowing users to reference computed values dynamically, or (2) by outputting them through a point control parameter, which would make the values available as a controllable/readable parameter in the After Effects UI.

*Tags: `params`, `ui`, `scripting`, `aegp`*

---

## How can a plugin render content that extends beyond the adjustment layer bounds?

When rendering content that exceeds the layer boundaries (such as text on an adjustment layer smaller than the composition), you need to use the output_rect parameter to specify the actual rendering region. By setting output_rect to encompass the full area where your content will be drawn, rather than constraining it to the layer bounds, the plugin can render beyond the adjustment layer's dimensions and display the full content within the composition.

*Tags: `output-rect`, `render-loop`, `aegp`, `ui`*

---

## How can a C++ plugin update a text layer every frame during rendering without locking the project?

This is a fundamental limitation in After Effects: calling AEGP_SetText from the render thread causes the project to lock. Several workarounds exist: (1) Write data to sequence data or compute cache from the render thread, then read and apply it from the UI thread using the aegp_idle hook, which is called multiple times per second; (2) Use your own text rendering engine in the render thread with an external library; (3) Write to disk and have a ScriptUI panel with a timer read it back (though this is unreliable). However, none of these work reliably in MediaEncoder or headless rendering without a GUI. The fundamental issue is that modifying a text layer from render could create infinite loops, which is why AE locks the project.

*Tags: `aegp`, `render-loop`, `threading`, `ui`, `sequence-data`, `compute-cache`*

---

## What are alternative approaches to accessing text layer properties when the standard AEGP text API is broken?

Alex Bizeau shared that the Kerning API in ExtendScript is broken, but works well in Expression Script. A workaround is to use ExtendScript to write an expression to a text layer that writes values to a separate text layer, then read that second layer's value back in ExtendScript. This bypasses the broken AEGP text kerning API. While hacky, this demonstrates using expressions as an intermediary to work around AE API limitations.

*Tags: `aegp`, `scripting`, `ui`, `reference`*

---

## Can you pass text output from an effect to a text layer instead of rendering it yourself?

Yes, you can encode text data into pixels and pass it to a text layer. One approach is to use dead pixels (out-of-frame pixels or pixels with alpha of 0) to encode the string data in RGB color values. This allows an effect that produces both pixels and text to delegate text rendering to a native text layer rather than rendering it directly.

*Tags: `ui`, `params`, `aegp`, `output-rect`*

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

## Is there an open-source example of hide/show functionality for After Effects plugins?

Alex Bizeau from Maxon mentioned the 36pix wrapper they created years ago as a working reference implementation for hide/show functionality in AE plugins. They also noted there is a special trick required for Premiere Pro but that hide/show is feasible to implement. The codebase is accessible to authorized developers.

*Tags: `ui`, `open-source`, `reference`, `premiere`*

---

## Is it possible to achieve progressive rendering in an After Effects plugin where samples render incrementally and display results to the user in real-time?

gabgren mentioned porting functionality from outside AE where progressive sampling over time with continuous result display is possible, making the interaction feel more responsive. They noted that the current AE plugin workflow requires waiting for all samples to render before seeing results on screen, which feels slow. They're exploring whether AE plugins can support this type of progressive rendering approach.

*Tags: `smart-render`, `render-loop`, `ui`, `mfr`, `performance`*

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

## What are examples of After Effects plugins that were superseded by native Adobe features?

Several plugins have been made redundant by native Adobe implementations. James Whiffin's Thicc Stroke plugin was superseded when Adobe released native tapered strokes. Tobias Fleischer's VariFont plugin, which provided variable font support in After Effects, was eventually replaced when Adobe finally implemented native variable font support after years of the plugin being the only solution available.

*Tags: `ui`, `reference`, `open-source`*

---

## Why does ImGui fail to initialize OpenGL loader on macOS?

The issue was related to window flag initialization order. On macOS, window flags need to be set before window creation, not after. This differs from Windows behavior where setting flags after creation may work. Ensure all window configuration flags are applied before creating the window.

*Tags: `macos`, `opengl`, `ui`, `debugging`*

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

## What is the current status of UXP support for Premiere Pro?

UXP for Premiere Pro is now available in public beta. Developers with existing CEP panels are encouraged to migrate to UXP and provide feedback to the Premiere Pro team about their migration needs. More information is available at: https://forums.creativeclouddeveloper.com/t/uxp-now-available-in-premiere-pro-beta/8795

*Tags: `premiere`, `uxp`, `ui`, `deployment`, `resource`*

---

## What is the best documentation to start learning UXP for Premiere Pro?

UXP in Premiere Pro is currently very limited but will change soon. There is a comprehensive article that contains all the relevant links to documentation, forums, and other resources to get started with UXP development for Premiere Pro.

*Tags: `uxp`, `premiere`, `ui`, `reference`, `documentation`*

---

## Can you integrate a C++ backend with UXP for Premiere Pro, or do you need to use a separate service with IPC for computational tasks?

There is a new hybrid UXP approach that combines HTML and C++ functionality, though details on how it works are still emerging. This hybrid approach is currently in beta and promises to allow C++ backend integration rather than requiring a separate IPC service for computational tasks.

*Tags: `uxp`, `premiere`, `ui`, `c++`*

---

## What is Bolt UXP?

Bolt UXP is a framework or resource for UXP development in Premiere Pro. It was shared as a relevant tool for developers working with UXP plugins.

*Tags: `uxp`, `premiere`, `ui`, `tool`, `reference`*

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

## How can I make an After Effects plugin save a new file when a UI button is clicked?

The After Effects SDK only handles importing files into projects or rendering to custom file types. To save arbitrary data to a file from a button click in a plugin, use standard C file I/O functions directly: fopen(), fwrite(), and fclose(). This is similar to how you would use .saveDlg(), .open(), .write(), and .close() in ExtendScript/JavaScript.

```cpp
fopen/fwrite/fclose
```

*Tags: `ui`, `aegp`, `sdk`, `reference`*

---

## How can you make a parameter read-only and prevent manual editing while still allowing programmatic keyframe changes?

Experiment with PF_PUI_DISABLED flag, which should prevent users from manually changing the parameter but may still allow expression connections. Alternatively, use PF_PUI_STD_CONTROL_ONLY to make the parameter non-keyframable, though this may affect expression connectivity. The expression-based approach mentioned above (using AEGP_SetExpression during UPDATE_PARAMS_UI) provides a workaround for creating dependent, programmatically-controlled parameters.

*Tags: `params`, `ui`*

---

## Why does transform_world give better results when entering parameter values numerically versus dragging sliders?

When working with transform_world, numeric input (typing a value and pressing enter) produces better results than slider dragging. This is theorized to be related to how After Effects halts the transformation process during interactive slider adjustments to provide a smoother user experience, though this behavior may be legacy code from before CC2015's separate rendering thread was introduced.

*Tags: `transform_world`, `params`, `ui`, `render-loop`*

---

## How does the AddArc function work when drawing multiple circles in After Effects UI?

The AddArc function treats all drawing operations as a continuous path. When drawing multiple circles, you must call close() after each AddArc to properly close that shape before starting a new one. Without closing, the path continues from the end point of one circle back to the beginning of the next, creating unexpected intersections. Alternatively, you can draw each shape in separate paths with separate drawing calls. The center argument for AddArc should use absolute position coordinates.

```cpp
DRAWBOT_PointF32 center1 = { drawRectF.left + point.x , drawRectF.top + drawRectF.height - point.y };
suites.PathSuiteCurrent()->AddArc(plotPath, &center1, 4.0f, 0.0f, 360.0f);
suites.PathSuiteCurrent()->Close(plotPath);
DRAWBOT_PointF32 center2 = { drawRectF.left + point.x + 100 , drawRectF.top + drawRectF.height - point.y };
suites.PathSuiteCurrent()->AddArc(plotPath, &center2, 4.0f, 0.0f, 360.0f);
suites.PathSuiteCurrent()->Close(plotPath);
```

*Tags: `ui`, `drawbot`, `path`, `debugging`*

---

## How do you remove a proxy from a composition or footage in After Effects?

There are several ways to remove a proxy in After Effects. You can click the small red square icon next to your composition or footage in the Project window to toggle the proxy on and off. Alternatively, you can remove the proxy altogether from the Interpret Footage window. Another method is to select the Proxy object in the Project Panel, then go to File > Set Proxy... and choose None.

*Tags: `ui`, `reference`, `aegp`*

---

## How can I ensure an ECW element receives draw calls and redraws consistently?

ECW UI elements receive idle calls only when the cursor is within their borders. If that's sufficient, you can use the native ECW idle calls. For redrawing regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows(). For softer redraw methods when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `ui`, `ecw`, `idle-calls`, `redraw`*

---

## How can I get the mouse position relative to the composition when the user clicks in the comp preview window?

Mouse coordinates are passed to the plugin through UI event callbacks. You can get the mouse click location in composition window coordinates and convert those into layer source coordinates. The CCU SDK sample project demonstrates how to implement this functionality with working code examples.

*Tags: `ui`, `reference`, `sdk`, `mouse-input`, `coords`*

---

## What is a good SDK sample project that demonstrates handling mouse clicks in the composition window?

The CCU SDK sample project is an official Adobe After Effects SDK example that shows how to get mouse click locations in comp window coordinates and convert them into layer source coordinates. It provides a complete working implementation for custom UI event handling.

*Tags: `reference`, `sdk`, `open-source`, `ui`, `sample-project`*

---

## Is there an API to set custom layer icons for AEGP plugins?

No official API exists for customizing layer icons in AEGP. While there are no standard SDK methods to change layer icons (like the star icon for shape layers or camera icon for camera layers), some developers have explored workarounds by directly manipulating the After Effects interface as an OS window, though this approach is not recommended or officially supported.

*Tags: `aegp`, `ui`, `sdk`*

---

## How can I make an AEGP plugin invisible and prevent it from appearing in the After Effects Window toolbar?

To make an AEGP plugin invisible and remove it from the Window toolbar, do not use AEGP_InsertMenuCommand or AEGP_RegisterCommandHook. These functions are only needed to create menu entries. By omitting these calls, the plugin will work in the background without appearing in the UI.

*Tags: `aegp`, `ui`, `deployment`*

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

## How can I prevent users from adding multiple instances of my effect to the same layer?

You can identify your effect uniquely by storing its AEGP_InstalledEffectKey during global setup using AEGP_GetNumInstalledEffects, AEGP_GetNextInstalledEffect, and AEGP_GetEffectMatchName. Then during UPDATE_PARAMS_UI, scan the effect's layer and check each effect's install key using AEGP_GetInstalledKeyFromLayerEffect to detect multiple instances. Once detected, you can alert the user, delete the redundant instance, disable it, or skip rendering for redundant copies by passing input to output. Note that AEGP_DisableCommand cannot remove effects from the menu, and users may still duplicate or copy/paste the effect.

*Tags: `aegp`, `params`, `ui`, `plugin-development`*

---

## Can you hide the second dropdown in PF_ADD_LAYER layer selector parameters?

No, the second dropdown (which allows selection between "Source", "Masks", or "Effects & Masks") cannot be hidden. That is the standard appearance of the layer selector in current versions of the After Effects SDK.

*Tags: `params`, `ui`, `sdk`, `mfr`*

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

## Is there a way to embed a CEP panel directly inside a plugin rather than as a separate extension?

No, After Effects requires CEP panels to be placed in designated CEP folders. However, two partial solutions exist: (1) Place a shortcut in the CEP folder pointing to your effect's folder to organize code elsewhere, or (2) Create an empty CEP that calls a function like 'FetchPanelCode()' defined in After Effects, which returns an HTML string that the panel can then evaluate dynamically.

*Tags: `cep`, `ui`, `plugin-architecture`*

---

## How can I display text in an After Effects plugin render buffer?

After Effects' API doesn't offer built-in text generating functions for the render buffer (unlike the UI buffer which has the drawbot suite). However, you can fill the render buffer using OS-native text tools: use GDI+ on Windows or Quartz on macOS to generate text in a native OS buffer, then copy that content back to After Effects' render buffer.

*Tags: `ui`, `windows`, `macos`, `render-loop`, `output-rect`*

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

## Why isn't sequence_data updated when modified in UpdateParameterUI and passed to SequenceResetup?

The issue is that UpdateParameterUI is not the correct place to modify sequence_data with FORCE_RERENDER. According to Adobe documentation, FORCE_RERENDER works during PF_Cmd_USER_CHANGED_PARAM and also in CLICK and DRAG events (if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented). Modifications to sequence_data should be made in PF_Cmd_USER_CHANGED_PARAM instead of UpdateParameterUI to ensure the data is properly synchronized between UI and render threads.

*Tags: `sequence-data`, `aegp`, `threading`, `params`, `ui`*

---

## How can you dynamically find After Effects menu command IDs instead of hardcoding them?

Use app.findMenuCommandId() to dynamically look up command IDs by name. This is the recommended approach because command numbers can change between After Effects versions. For example: id = app.findMenuCommandId("copy"); or app.executeCommand(app.findMenuCommandId("Close Project"));

```cpp
id = app.findMenuCommandId("copy");
app.executeCommand(app.findMenuCommandId("Close Project"));
```

*Tags: `scripting`, `extendscript`, `ui`, `cross-platform`*

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

## What language and tools are needed to create After Effects plugins?

After Effects plugins are written in C/C++. While you can work with other languages, all calls to the AE API must be in C/C++. The SDK is shipped with Visual Studio project files for Windows and Xcode project files for Mac. Effect controls windows and composition window overlay graphics are created using the DrawBot suite supplied by the AE SDK.

*Tags: `sdk`, `c/c++`, `windows`, `macos`, `ui`, `build`*

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

## Why don't slave parameters update when the master parameter is animated and scrubbed through the timeline?

Since CC2015, the render thread is not allowed to modify the project in any way, which prevents parameter updates during rendering. Additionally, with MFR (Multiple Frame Rendering), multiple frames are rendered simultaneously, making it ambiguous what value a parameter should have. If the slave parameter only affects the display UI and doesn't influence the actual render, you can work around this by: (1) creating a custom UI that redraws on time changes using its draw event, or (2) creating an AEGP with an idle_hook that scans the current composition for your effect and modifies it as needed. However, if the slave parameter needs to affect the render or be read by expressions in other effects/layers, this design pattern is not supported by After Effects' workflow.

*Tags: `params`, `render-loop`, `mfr`, `ui`, `aegp`*

---

## How do I draw custom UI in the title area of a parameter to position it as far left as possible?

When defining the parameter, set PF_PUI_TOPIC instead of PF_PUI_CONTROL. You'll need to respond to PF_EA_PARAM_TITLE instead of PF_EA_CONTROL during the event call. If you need UI in both the title area and control area, you can define both for the same parameter, but note that the two areas won't join into one continuous segment.

*Tags: `ui`, `pipl`, `params`*

---

## How can I create a plugin that adds image layers and regenerates them when properties change?

This is possible by combining an AEGP panel plugin with an effect plugin. The AEGP panel (implemented similar to the SDK's 'Panelator' sample) creates and manages layers with an inspector window for user controls. The effect plugin handles rendering and stores configuration data using either hidden parameters (which can be read/written by the AEGP window) or sequence data (a memory chunk that stores arbitrary data without UI representation, undo stack entries, or reset behavior). The effect plugin can regenerate images whenever properties change, with the decisions made in the panel interface driving the effect's rendering behavior.

*Tags: `aegp`, `params`, `sequence-data`, `ui`, `reference`*

---

## What is the recommended sample project for learning how to create a dockable panel in After Effects?

The SDK sample project 'Panelator' demonstrates how to create a dockable panel in After Effects. This sample shows the fundamentals of building a panel interface that can be populated with custom controls and integrated into the AE workspace.

*Tags: `aegp`, `ui`, `reference`, `sdk`*

---

## Where should AEGP_ExecuteScript be called in a C++ plugin to display a dialog each time the effect is applied?

AEGP_ExecuteScript should not be placed in GlobalSetup or ParamsSetup, as both occur only once per session. Instead, use sequence_setup, which is called once when a new instance is applied to a layer (not when duplicated or copy/pasted), or update_params_ui, which requires tracking to avoid launching the dialog multiple times for the same instance.

*Tags: `aegp`, `scripting`, `ui`, `params`*

---

## How can I create a custom text input field in the Effect Control Window instead of using a JavaScript popup?

Custom UI elements in the ECW or comp window must be handled exclusively via DrawBot. You cannot add OS-level text fields directly to the UI. Instead, you need to either: (1) translate user input through AE's event system onto an offscreen text controller and copy its image buffer, or (2) create a text editor from scratch using the event system and DrawBot. Alternatively, use a JavaScript window for input and display the result as non-interactive text in the ECW that launches the JavaScript window when clicked.

*Tags: `ui`, `drawbot`, `aegp`, `scripting`*

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

## Where did the 'About' button go for plugins in After Effects 2020?

The 'About' button was deliberately hidden in After Effects 2020. It is still accessible by right-clicking the effect name and choosing 'About' from the context menu. As an alternative, developers can use PF_SetOptionsButtonName() to implement an options button, which will be displayed next to the 'Reset' button.

*Tags: `ui`, `macos`, `pipl`, `debugging`*

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

## How can I change the current time (CTI) in After Effects using an AEGP plugin?

To change the current time (CTI) with a plugin, first get the comp's item from the effect layer, then use AEGP_SetItemCurrentTime. The process involves: (1) using AEGP_GetEffectLayer to get the layer handle, (2) using AEGP_GetLayerParentComp to get the composition handle, (3) using AEGP_GetItemFromComp to get the item handle from the composition, and (4) finally calling AEGP_SetItemCurrentTime with the item handle and desired time value.

```cpp
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &m_layerH);
suites.LayerSuite8()->AEGP_GetLayerParentComp(m_layerH, &m_compH);
suites.CompSuite11()->AEGP_GetItemFromComp(m_compH, &compItemH);
suites.ItemSuite9()->AEGP_SetItemCurrentTime(compItemH, &compTime);
```

*Tags: `aegp`, `cti`, `scripting`, `ui`, `reference`*

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

## How can I force an overlay to refresh when ARB parameter data changes without requiring mouse movement?

Use the PF_Event_IDLE event which is automatically sent to composition UIs. In the idle event handler, check system time or other factors to determine if a refresh is needed. If so, set PF_OutFlag_REFRESH_UI or call PF_InvalidateRect(), which will trigger a PF_Event_DRAW call to redraw the overlay and reflect the updated ARB data.

*Tags: `ui`, `arb-data`, `aegp`, `render-loop`*

---

## How can you collapse a parameter in the After Effects timeline using the SDK?

An alternative way to collapse a parameter is to use AEGP_SetStreamFlags() to hide and then unhide the parameter, which may collapse the parameter in the timeline as well. This approach can be used in addition to PF_ParamFlag_START_COLLAPSED and PF_ParamFlag_COLLAPSE_TWIRLY, which work for collapsing twirlies in the Effect Control Window (ECW).

*Tags: `sdk`, `params`, `ui`, `aegp`*

---

## Can I call a command selector like PF_Cmd_DO_DIALOG in the render function?

No, since CC2015 the render thread cannot perform operations that change the project. Instead, you should use alternative approaches: (1) Use an idle hook to send a message from the render thread to an AEGP, which then creates a window during the idle hook call when it's safe. (2) Use SEQUENCE_SETUP, which is called whenever a new instance of your effect is applied. (3) Use GLOBAL_SETUP, which is called once per session when the effect is first applied. These approaches allow you to open dialogs for user input (like username/password) without trying to trigger commands from the render thread.

*Tags: `render-loop`, `aegp`, `threading`, `ui`*

---

## How do you capture when a popup parameter selection changes in an After Effects plugin?

To capture popup menu selection changes, the parameter must be created with the PF_ParamFlag_SUPERVISE flag. When such a parameter is changed, the plugin receives a USER_CHANGED_PARAM call. All plugin calls go through the main entry point function (EffectMain), which should implement the selector to handle this notification.

*Tags: `params`, `ui`, `aegp`, `plugin-development`*

---

## How can I find the command ID for Numpad+0 (play preview from start) in After Effects?

Setup a command hook with the 'all commands' flag. Once you've made some preliminary calls, you can press 0 and catch the command ID that is triggered. This method works because the Numpad+0 command doesn't have a corresponding menu item, so app.findMenuCommandId() won't work directly.

*Tags: `scripting`, `aegp`, `ui`, `debugging`*

---

## How do you add static text labels to an After Effects effect UI?

You can use PF_ADD_TOPIC and PF_END_TOPIC macros without putting any params in between to create a text section. Alternatively, you can use PF_ADD_NULL, declare a minimal custom UI size, and not implement it.

*Tags: `ui`, `pipl`, `params`, `reference`*

---

## How can you detect which mouse button (left, middle, or right) was pressed in an After Effects plugin?

You cannot rely on the AeData structure to determine which mouse button was pressed. Instead, you must use the operating system API. For Windows, use the GetKeyState() function with virtual key codes: VK_LBUTTON for left button, VK_MBUTTON for middle button, and VK_RBUTTON for right button. Check if the result masked with 0x100 is non-zero to determine if the button is currently pressed.

```cpp
case button::left: return (GetKeyState(VK_LBUTTON) & 0x100) != 0;
case button::middle: return (GetKeyState(VK_MBUTTON) & 0x100) != 0;
case button::right: return (GetKeyState(VK_RBUTTON) & 0x100) != 0;
```

*Tags: `ui`, `windows`, `sdk`, `aegp`*

---

## How can a plugin force an undo or redo action in After Effects?

Use AEGP_DoCommand() with command IDs: 16 for undo and 2035 for redo. Note that directly manipulating undo state is generally discouraged as a best practice.

```cpp
AEGP_DoCommand();
// undo = 16
// redo = 2035
```

*Tags: `aegp`, `ui`, `reference`*

---

## How can I make PF_Param_POINT and PF_Param_POINT_3D parameters maintain constant values independent of layer resizing?

PF_Param_POINT and PF_Param_POINT_3D parameters by default scale proportionally with layer dimensions. To maintain absolute coordinates invariant under layer size changes, shachar carmi recommends three workarounds: (1) Create a custom parameter with custom composition UI to have full control over behavior; (2) Add an expression to your point parameter programmatically that divides the current position by layer dimensions and multiplies by a scaling factor to maintain the same absolute coordinates when layer dimensions change; (3) Try the PF_ValueDisplayFlag_PIXEL flag, though its applicability to point parameters is undocumented and may have unexpected behavior.

*Tags: `params`, `ui`, `sdk`*

---

## How can I detect when the user performs undo or redo actions in my After Effects effect plugin so I can update my custom UI?

One effective approach is to save a random number or state identifier with your data, then poll the effect data (stored in a data parameter) once or twice per second to check if the state index differs from what your UI window expects. If there's a mismatch, the data has changed via undo/redo or project load. To optimize performance, you can save the random number to a separate parameter like a slider. When saving data, use StartUndoGroup() and EndUndoGroup() to bundle operations into a single undo entry. This technique works well for effects with custom external UIs that manage internal parameters in a structure, which is then serialized to an AEGP data parameter using AEGP_SetStreamValue().

*Tags: `undo`, `redo`, `ui`, `aegp`, `params`, `debugging`*

---

## How should you diagnose a plugin crash that only occurs in After Effects CC 2015+ but not CC 2014, particularly when it involves custom UI?

Ask the user to send over their old After Effects preferences folder and then delete the pref folder before relaunching AE. If resetting the preferences solves the issue, try reproducing the issue on your machine using the user's old preferences. This approach has a high probability of resolving the issue. If successful, the old prefs can help identify what setting or cached data was causing the crash.

*Tags: `ui`, `debugging`, `macos`, `preferences`*

---

## Is there an open-source example of using sequence_data and popup dialogs in an After Effects plugin?

The AE_tl_math plugin by crazylafo on GitHub demonstrates sequence data handling in an After Effects plugin. Available at: https://github.com/crazylafo/AE_tl_math (specifically the tl_math.cpp file). This example shows how to flatten sequences and work with data communication in plugins.

*Tags: `sequence-data`, `open-source`, `reference`, `ui`*

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

## Why is sequence_data different between PF_Cmd_RENDER and PF_Cmd_USER_CHANGED_PARAM in After Effects CC 2015 and later?

In After Effects CC 2015 and later, the render thread and UI thread have separate sequence_data handles that only synchronize at specific occasions. This is a design change from earlier versions like CS5.5. To force synchronization, set PF_OutFlag_FORCE_RERENDER during UserChangedParam, which forces a flatten call on the UI thread and an unflatten on the render thread, syncing the render thread's sequence data with the UI data.

*Tags: `sequence-data`, `threading`, `params`, `render-loop`, `ui`*

---

## How can data be shared between render and UI threads if sequence_data synchronization is not reliable?

Global data is shared between threads and can be used as an alternative solution for render-to-UI-thread communication when sequence_data synchronization is not reliable. However, relying on updating UI sequence_data from the render function is not supported by design in CC 2015 and later versions.

*Tags: `threading`, `memory`, `render-loop`, `ui`*

---

## Why does adding keyframes from a modeless dialog fail in After Effects CC2015 and later?

Since After Effects 2015, there is a known issue with calling AE's suites while a modal loop is running. To work around this, retrieve the information you need from After Effects before running the modal loop, rather than attempting to call AE suites from within the modal dialog.

*Tags: `aegp`, `ui`, `debugging`, `cross-platform`*

---

## How can an AEGP communicate with a CEP/ScriptUI panel to pass data back and forth?

You can use AEGP_ExecuteScript() to send data from the AEGP to your JavaScript panel, and retrieve data from the JavaScript side via the returned value. For layer selection detection, you can use comp.selectedLayers and comp.selectedProperties from JavaScript with a scheduled task to check for selection changes at intervals. Alternatively, use idle_hook in the AEGP to check periodically and execute scripts via AEGP_ExecuteScript(). For deeper integration, there is a JavaScript SDK that allows you to build an AEGP that gets called to execute custom JavaScript APIs.

*Tags: `aegp`, `scripting`, `ui`, `cep`*

---

## How can I draw custom curves outside the composition bounds in After Effects?

To draw custom UI elements like curves outside the composition, you need to use PF_Event and Drawbot APIs. Look up "PF_Event" and "Drawbot" in the After Effects SDK guide for implementation details.

*Tags: `ui`, `aegp`, `custom-ui`, `drawing`*

---

## Where can I find a sample project demonstrating Drawbot usage?

The "CCU" sample project in the After Effects SDK demonstrates the use of Drawbot for custom UI drawing.

*Tags: `ui`, `reference`, `sample`, `drawbot`*

---

## Why doesn't PF_Event_IDLE run after restoring a project until a control is touched?

PF_Event_IDLE is only called when an effect instance is the sole selected item. When you apply an effect from the menu, it automatically becomes selected alone, so idle events trigger. However, when loading a project, effects are not automatically selected, so idle events don't fire until a control is manually touched. This is expected behavior and cannot be overridden.

*Tags: `aegp`, `ui`, `debugging`, `params`*

---

## How can I handle UI synchronization tasks that need to run even when an effect isn't selected after project restore?

Use AEGP_RegisterIdleHook() to register a general-purpose idle hook that receives regular calls not tied to a specific effect instance. During these calls, you can scan the current comp or entire project for your effect instances using AEGP_GetActiveItem(), AEGP_GetFirstProjItem(), AEGP_GetNextProjItem(), AEGP_GetCompFromItem(), AEGP_GetCompLayerByIndex(), AEGP_GetLayerEffectByIndex(), and AEGP_GetInstalledKeyFromLayerEffect() to identify your effects. You can then programmatically select your effect using AEGP_NewCollection(), AEGP_CollectionPushBack(), and AEGP_SetSelection() so your regular effect process picks it up.

*Tags: `aegp`, `ui`, `params`, `debugging`*

---

## How can you keep UI and render threads in sync when automatically applying multiple effects in CC2015+?

When using Sequence Setup to automatically add effects to a layer, the UI and render copies of the project can become out of sync in CC2015+, causing verification failures and missing data effects. The solution is to use an AEGP (After Effects General Plug-in) with an idle hook instead of modifying the layer during Sequence Setup. The effect can set a flag when it's applied, and the AEGP's idle hook will then process the pending operation to add the additional effects. The AEGP can be bundled in the same binary as the effect plugins, but the AEGP's PiPL must be listed first in the bundle.

*Tags: `aegp`, `pipl`, `ui`, `smart-render`, `threading`, `plugin-architecture`*

---

## How can I add markers to a layer from an effect panel in After Effects?

The user reported finding a solution to add markers to layers from an effect panel, though the specific implementation details were not fully shared in this conversation. The discussion suggests using the After Effects SDK to manipulate layer markers programmatically from custom UI panels.

*Tags: `sdk`, `ui`, `params`, `aegp`*

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

## How can I share data between the render thread and UI event handlers?

Since After Effects 13.5, sequence_data modified in the render thread cannot be accessed in UI threads. Two workarounds are: (1) add a pointer to sequence_data that is shared between threads (requiring logic to distinguish live pointers from saved invalid ones), or (2) store data in the global_data handle which is shared between threads and instances (requiring logic to identify which data belongs to which instance). The recommended approach is caching values in sequence_data during render and reading them back in event handlers from the UI thread.

*Tags: `sequence-data`, `threading`, `arb-data`, `ui`, `render-loop`*

---

## How can I draw gradients in an After Effects effect UI using DrawBot?

DrawBot does not have a direct API function for drawing gradients. The recommended approach is to draw the gradient into a buffer using any available means, then convert that buffer to a DrawBot image using the NewImageFromBuffer() function. This allows you to create dynamic gradient fills like color wheels for custom effect panels.

*Tags: `ui`, `drawbot`, `aegp`, `reference`*

---

## How do you hide a layer parameter in After Effects without breaking parameter evaluation?

To hide a layer parameter while maintaining evaluation, you must set the ui_flags during parameter setup using PF_PUI_NO_ECW_UI. If you hide the parameter later using AEGP_SetStreamFlag(), the parameter will stop evaluating. The correct approach is to hide it at initialization time rather than at runtime.

```cpp
def.ui_flags = PF_PUI_NO_ECW_UI;
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## How can I detect when a user selects a layer in After Effects?

There is no direct layer selection event in the AEGP API. However, you can use the idle_hook to periodically check if the composition's layer collection has changed, which allows you to detect when layer selection has occurred.

*Tags: `aegp`, `ui`, `reference`*

---

## How do I port an AEIO plugin with a user options dialog from Windows to macOS?

To port an AEIO_UserOptionsDialog from Windows to macOS, you need to use native macOS UI frameworks instead of Windows dialogs. For Carbon implementation, refer to the Adobe forums thread at https://forums.adobe.com/thread/559946?tstart=0 which contains code examples and links for Cocoa. Alternatively, you can use AEGP_ExecuteScript() to open a dialog via JavaScript, which works cross-platform on both Windows and macOS without requiring separate implementations.

*Tags: `aeio`, `ui`, `cross-platform`, `macos`, `windows`*

---

## What is a good approach for creating user option dialogs in AEIO plugins that work on both Windows and macOS?

You can use AEGP_ExecuteScript() to implement user option dialogs in JavaScript, which provides a cross-platform solution that works identically on Windows and macOS. This avoids the need to maintain separate platform-specific dialog code. For implementation details, see the example at https://forums.adobe.com/message/3625857#3625857 and search the Adobe After Effects scripting community forum at https://forums.adobe.com/community/aftereffects_general_discussion/ae_scripting for additional script examples with checkboxes and dropdown lists.

*Tags: `aeio`, `scripting`, `ui`, `cross-platform`, `aegp`*

---

## Why does my PF_Cmd_DO_DIALOG case handler run constantly instead of only when the dialog button is clicked?

The issue is likely a missing break statement on the case above PF_Cmd_DO_DIALOG. Without a break, execution falls through to the dialog case, causing it to run repeatedly. Ensure each case statement in your switch block has a proper break statement to prevent fall-through execution.

```cpp
case PF_Cmd_DO_DIALOG:
  err = Register(in_data, out_data);
  break;
```

*Tags: `ui`, `debugging`, `sdk`*

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

## How can I programmatically detect when a user presses the play button in After Effects?

Use AEGP_RegisterCommandHook() to register a command hook that listens for play events. The relevant command numbers are: 2415 for Play (spacebar) and 2285 for RAM Preview. This allows you to be notified when the user triggers playback rather than implementing a custom onClick function.

```cpp
AEGP_RegisterCommandHook()
// Command numbers:
// 2285 - RAM Preview
// 2415 - Play (spacebar)
```

*Tags: `aegp`, `scripting`, `ui`, `reference`*

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

## What language and tools should I use to develop an After Effects plugin for controlling layer properties and exporting data?

For your use case of creating a GUI, controlling layers and layer properties (content, effects, transform), and exporting data as XML, you don't need to develop a C plugin. Instead, use JavaScript/ExtendScript to create a script that can be encrypted as a .jsxbin file. Refer to the After Effects Scripting Guide PDF for detailed information on manipulating layers, effects, and properties. The AE Scripting forum at https://forums.adobe.com/community/aftereffects_general_discussion/ae_scripting is also a valuable resource for scripting questions.

*Tags: `scripting`, `ui`, `deployment`, `reference`*

---

## How can I open a file browser dialog in an After Effects plugin to let users select a file?

There is no direct C API call in the SDK to open a file browser dialog. However, you can use AEGP_ExecuteScript() to execute a script that opens a file browser and retrieves the selected file path, then pass that information back to your plugin.

*Tags: `aegp`, `ui`, `scripting`, `reference`*

---

## How do you get input layer data with effects applied during an arbitrary draw event in After Effects?

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

*Tags: `sequence-data`, `caching`, `smart-render`, `ui`, `aegp`*

---

## How do you force After Effects to re-render when custom UI parameters change but the effect's cached result is stale?

Use PF_GetCurrentState and PF_HasParamChanged during a draw event to detect if the plug-in's parameter state has changed since the last draw call. If parameters have changed, set the evt_out_flags to include PF_OutFlag_FORCE_RERENDER to force a fresh render. Simply setting change_flags or PF_EO_HANDLED_EVENT alone is not sufficient to overcome AE's caching behavior.

*Tags: `caching`, `ui`, `smart-render`, `params`*

---

## How can I efficiently update many parameters without causing UI slowness from repeated PF_ChangeFlag_CHANGED_VALUE flags?

Instead of setting PF_ChangeFlag_CHANGED_VALUE for each of many parameters, use AEGP_SetStreamValue() from the AEGP Suite to update parameter values more efficiently. Alternatively, if the parameters don't need to be directly exposed as individual sliders in the UI, store them in sequence data or arbitrary data, which are automatically saved with the project without requiring change flags for each value.

```cpp
params[paramId]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `ui`, `performance`*

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

## Can you access the comp overlay buffer during a render call using Drawbot?

No, you cannot access the comp overlay buffer during a render call, regardless of whether you're using Drawbot. Instead, you must cache whatever you want to render during a "draw" event call, and then use that cache during the render call. Drawbot's data structures are opaque and don't offer tools for reading data back out. If you need to draw during render, consider using third-party drawing tools or OS-level drawing tools instead, such as OpenGL.

*Tags: `drawbot`, `render-loop`, `caching`, `ui`*

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

## What is the difference between DrawBot and OS drawing libraries for After Effects plugin development?

DrawBot is designed exclusively for drawing custom UI overlays on AE's interface in CS5 and later. It provides built-in drawing tools and operates only on AE's internal interface graphics buffers. OS drawing libraries (Quartz on macOS, GDI+ on Windows) are general-purpose graphics APIs used to render into effect output images. If you need to draw shapes, lines, or text into an effect's output buffer rather than a UI overlay, you must use OS-native drawing tools combined with the Memory Suite to manage the graphics context, not DrawBot.

*Tags: `drawbot`, `ui`, `macos`, `windows`, `reference`*

---

## How can you detect when a user sets parenting on a layer in After Effects?

There is no direct event to detect parenting changes. However, you can use several workarounds: (1) Keep a hidden slider that tracks the parent layer's index and detect changes via re-render calls when the index changes, (2) Register a command hook with the 'ALL COMMANDS' flag to monitor all commands sent by AE and look for parenting-related commands, or (3) Use expressions to recursively detect parenting changes through the layer chain. A slider with expressions is a practical approach for tracking parent changes.

*Tags: `aegp_api`, `params`, `ui`, `debugging`*

---

## What is an alternative approach to directly detecting parenting changes in After Effects effects?

Instead of trying to detect parenting events directly, you can use expressions to keyframe parenting behavior. One effective method is to use layer markers' comments as parent layer names within position, rotation, and scale expressions. This allows you to simulate dynamic parenting through expressions without needing to monitor parenting change events.

*Tags: `params`, `scripting`, `ui`*

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

## How should I port a plug-in from CS4 to CS6 when PF_Context's cgrafptr and PF_GET_CGRAF_DATA macro are removed?

The drawing mechanism changed between CS4 and CS5. Direct drawing into a context is no longer supported. Instead, you must use the DrawBot suite to draw. The quickest conversion approach is to wrap the DrawBot commands under your old drawing command names. This change also makes the code cross-platform compatible.

*Tags: `mfr`, `ui`, `windows`, `cross-platform`, `porting`*

---

## How can I make my After Effects plugin support multiple bit depths instead of just 8-bit?

You cannot explicitly restrict AE to only process at specific bit depths through plugin settings. Instead, check the bitdepth at runtime using `extra->input->bitdepth`, and if the bitdepth is not supported, copy the input to output without processing and inform the user via a message overlay or alert that the effect requires a specific bit depth.

*Tags: `params`, `smart-render`, `ui`, `sdk`*

---

## How can I detect when a composition's color depth or pixel format is changed by the user?

There is no direct event for color depth changes, but the frame is re-rendered when the comp depth is changed. You can check the current pixel format using PF_GetPixelFormat() during the render call. To update parameter UI in response, use PF_UpdateParamUI during the render call, and you can hide/unhide parameters at any time using AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN. Parameter property changes can be made when receiving events or PARAMS_UI_CHANGED.

*Tags: `params`, `ui`, `aegp`, `pixel_format`, `bit_depth`*

---

## What parameter type should be used to replace Pixel Bender percent sliders when porting to the After Effects SDK?

Use PF_ADD_FLOAT_SLIDERX instead of PF_ADD_PERCENT, since Pixel Bender implements percent sliders as float sliders. You can add the PF_ValueDisplayFlag_PERCENT flag to the PF_ADD_FLOAT_SLIDER call to display the float slider as a percentage in the UI without changing the underlying implementation.

```cpp
PF_ADD_FLOAT_SLIDERX("param_name", 0, 100, 0, 100, 1, PF_ValueDisplayFlag_PERCENT);
```

*Tags: `params`, `ui`, `pipl`*

---

## How do you handle coordinate system differences when drawing UI elements in Quartz on macOS After Effects plugins?

Quartz uses a different coordinate system than QuickDraw on macOS, with the Y-axis inverted and origin at bottom-left instead of top-left. To handle this, use CGContextConvertPointToUserSpace to get the flipped origin point, then adjust your Y coordinates accordingly. The solution involves getting the flip origin once per draw call and incorporating it into your transformation matrix: CGPoint zero = {0,0}; CGPoint flipOrigin = CGContextConvertPointToUserSpace(context, zero); flippedY = flipOrigin.y - (originaY - event_extraP->u.draw.update_rect.top);

```cpp
CGPoint zero = {0,0};
CGPoint flipOrigin = CGContextConvertPointToUserSpace(context, zero);
flippedY = flipOrigin.y - (originaY - event_extraP->u.draw.update_rect.top);
```

*Tags: `macos`, `ui`, `drawing`, `quartz`, `coordinate-systems`*

---

## How do you draw transformed circular handles in macOS After Effects plugins accounting for layer scale, rotation, and position?

Use Quartz context transformation matrices to apply layer-to-frame transformations. First get the layer2frame transformation matrix using get_layer2comp_xform and source_to_frame callbacks. Then create a CGAffineTransform that includes the Y-axis inversion correction, and apply it to the context using CGContextConcatCTM before drawing. The Y-inversion adjustment should negate the b and d components and adjust the ty translation by the flipped origin offset.

```cpp
float a = xform.mat[0][0];
float b = xform.mat[0][1];
float c = xform.mat[1][0];
float d = xform.mat[1][1];
float tx = xform.mat[2][0];
float ty = xform.mat[2][1];
CGPoint origin = CGPointMake(0,0);
CGPoint originFlipped = CGContextConvertPointToUserSpace(context, origin);
CGAffineTransform trans = CGAffineTransformMake (a, -b, c, -d, tx, originFlipped.y - (ty - event_extraP->u.draw.update_rect.top));
CGContextConcatCTM (context, trans);
CGRect rect = CGRectMake ( center.x - cr, center.y - cr, 2*cr, 2*cr);
CGContextAddEllipseInRect(context, rect);
CGContextStrokePath(context);
```

*Tags: `macos`, `ui`, `quartz`, `drawing`, `transformation`, `matrix`*

---

## Is there an API to draw standard After Effects controls like buttons and checkboxes outside the Effect Controls Window?

No, there is no API available to draw standard AE controls outside the ECW (Effect Controls Window). Effect controls can only be created during param_setup in effects and will display only in the ECW and timeline. Custom controls would need to be reimplemented with custom drawing code.

*Tags: `ui`, `params`, `aegp`, `reference`*

---

## How can I catch mouse wheel events in After Effects plugins?

After Effects does not supply effects with mouse wheel events through its standard API. The SDK only provides clicks, drags, and cursor enter/exit events for UI regions. If you need mouse wheel support on Windows, you can use the HWND descriptor approach, but this is only available for AEGP Palette plugins, not standard effects.

*Tags: `ui`, `aegp`, `windows`, `debugging`*

---

## How can I prevent PF_Param_POINT from intercepting mouse click and drag events near custom UI controls?

The issue is that After Effects' interface takes higher priority over plug-in interface and intercepts drag events. The solution is to disable the point parameter UI when needed by setting the PF_PUI_DISABLED flag using PF_UpdateParamUI. This makes the point parameter unselectable and prevents it from intercepting clicks and drags. You can set this flag selectively when the cursor is close to your custom control or inside the comp window.

```cpp
def.ui_flags = PF_PUI_DISABLED;
suites.ParamUtilsSuite1()->PF_UpdateParamUI(in_data->effect_ref, paramIndex, &def);
```

*Tags: `ui`, `params`, `debugging`*

---

## Why can't I draw a custom UI in PF_Window_LAYER even though it works in PF_Window_COMP?

This was a bug in After Effects CS3 and CS4 where custom UI drawing in layer windows was not supported. The plug-in would receive PF_Event_DRAW calls and return a buffer, but After Effects would ignore the rendered output. This issue was fixed in CS5. As a workaround, you may need to upgrade to CS5 or later, or reference how other plugins like 'meshwarp' or 'vector paint' handle this limitation.

*Tags: `ui`, `custom-ui`, `debugging`, `ae-versions`*

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

## How do you create a modal dialog window on macOS using Interface Builder for an After Effects plugin options button?

Create a .nib file in Interface Builder and set the window class type to 'movable modal' to keep it in front of After Effects. Add the .nib file to your Xcode project and add it to the 'Copy Bundle Resources' build phase. Link CoreFoundation.framework and IOKit.framework. Update your plugin's .plugin-info.plist with a unique CFBundleIdentifier. Then use the Carbon API to load the .nib file: create a CFBundleRef, use CreateNibReferenceWithCFBundle, CreateWindowFromNib, and install event handlers. Call RunApplicationEventLoop to display the dialog and handle user interactions before closing.

```cpp
CFBundleRef bundleRef = CFBundleGetBundleWithIdentifier(CFSTR("com.adobe.AfterEffects.YourPlugName"));
IBNibRef theNibRef = NULL;
OSStatus theErr = CreateNibReferenceWithCFBundle(bundleRef, CFSTR("YourNibFileName"), &theNibRef);
WindowRef ActivationWindow = NULL;
theErr = CreateWindowFromNib(theNibRef, CFSTR("Window"), &ActivationWindow);
DisposeNibReference(theNibRef);
RepositionWindow(ActivationWindow, NULL, kWindowCascadeOnMainScreen);
ShowWindow(ActivationWindow);
theErr = InstallWindowEventHandler(ActivationWindow, EventHandlerFunction, GetEventTypeCount(kEvents), kEvents, (void*)ActivationWindow, NULL);
RunApplicationEventLoop();
```

*Tags: `ui`, `macos`, `build`*

---

## What is an alternative to Interface Builder for creating simple dialogs on macOS in After Effects plugins?

Use the CFUserNotification API, which is C-based and simpler than Interface Builder and Objective-C. It's suitable for basic dialogs with string input and 1-2 text fields. Refer to Apple's CFUserNotification documentation at http://developer.apple.com/mac/library/documentation/CoreFoundation/Reference/CFUserNotificationRef/Reference/reference.html

*Tags: `ui`, `macos`, `reference`*

---

## How can I create an After Effects plugin that samples the current frame and displays information graphically in a separate window?

You should build an AEGP (After Effects General Plugin) rather than an effect plugin. Use the 'panelator' sample as a base to create a dockable panel for displaying your graphical output. To access frame data, you have two approaches: (1) Use the 'EMP' (external monitor preview) sample which provides the MyBlit() function to receive the currently viewed composition buffer, registered during entryPoint(). (2) Use AEGP functions to render project items on demand: AEGP_GetMostRecentlyUsedComp, AEGP_RenderAndCheckoutFrame, and AEGP_GetReceiptWorld. The second approach can leverage AE's render cache to avoid re-rendering if the frame has already been processed.

*Tags: `aegp`, `ui`, `output-rect`, `reference`*

---

## What is the panelator sample plugin used for?

The 'panelator' sample is an AEGP that demonstrates how to create a dockable panel in After Effects, similar to built-in palettes like the Info palette. It serves as a template for AEGP plugins that need to display custom user interfaces and allows you to draw arbitrary content within the panel.

*Tags: `aegp`, `ui`, `reference`, `sample`*

---

## How can you create dynamic stroke parameters similar to the Paint effect in After Effects plugins?

Strokes are handled by the dynamicStreamSuite, which allows you to add, remove, and alter strokes and text layer parameters. However, custom parameters that behave like strokes cannot be created through the API. Two workarounds are: (1) Create your effect with hidden parameter groups (e.g., 50 groups, each with width, length, start, and end sliders) and unhide them as needed—this is simpler but limited by the number of compiled groups. (2) Create a dummy effect with the desired sliders and add new instances of it to the layer whenever more strokes are needed—this is more flexible and allows unlimited strokes but is significantly more complex to implement.

*Tags: `params`, `ui`, `sdk`, `aegp`, `pipl`*

---

## How do you use PF_AppGetColor() to retrieve After Effects application colors in a custom effect?

Use the AppSuite3() to call PF_AppGetColor() with the desired color type and a pointer to store the result. The syntax is: suites.AppSuite3()->PF_AppGetColor(PF_App_Color_HOT_TEXT, &appColor1); where PF_App_Color_HOT_TEXT is the color type you want to retrieve and appColor1 is the variable that will store the retrieved color.

```cpp
suites.AppSuite3()->PF_AppGetColor(PF_App_Color_HOT_TEXT, &appColor1);
```

*Tags: `ui`, `aegp`, `params`, `reference`*

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

## How can you hide programmatically created layers like cameras from the AE timeline panel?

Use AEGP_GetNewStreamRefForLayer to get a stream reference for the camera layer, then call AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to hide it from the timeline. However, this approach has risks—a better alternative is to keep the layer visible but locked, similar to how plugins like Particular handle added elements, to prevent accidental deletion by users.

```cpp
AEGP_StreamRefH streamPH = NULL;
err = suites.DynamicStreamSuite3()->AEGP_GetNewStreamRefForLayer(NULL, layerPH, &streamPH);
err = suites.DynamicStreamSuite3()->AEGP_SetDynamicStreamFlag(streamPH, AEGP_DynStreamFlag_HIDDEN, false, true);
```

*Tags: `aegp`, `ui`, `layer-checkout`*

---

## How can I hide an effect from the Effects menu while still allowing it to be added programmatically?

Set the PF_OutFlag_I_AM_OBSOLETE flag during global setup, and the effect will not appear in the effects list but can still be added using addEffect from code.

*Tags: `ui`, `pipl`, `aegp`*

---

## How do you get resize callbacks for a dockable plugin window in macOS?

To receive resize callbacks for a dockable plugin window, you need to register for the kEventControlBoundsChanged event in your event handler specification. Set up your event handler with an EventTypeSpec array that includes {kEventClassControl, kEventControlBoundsChanged}, which will trigger whenever the control is resized. You may also want to register for kEventControlOwningWindowChanged to detect when the control's owning window changes.

```cpp
static const EventTypeSpec kControlEventSpec[] = {
  {kEventClassCommand, kEventProcessCommand},
  {kEventClassControl, kEventControlDraw},
  {kEventClassControl, kEventControlOwningWindowChanged},
  {kEventClassControl, kEventControlBoundsChanged}
};
OSErr err = InstallEventHandler(GetControlEventTarget(i_refH),
  NewEventHandlerUPP(S_EventHandler),
  GetEventTypeCount(kControlEventSpec),
  kControlEventSpec,
  (void*)this,
  NULL);
```

*Tags: `ui`, `macos`, `debugging`*

---

## Can I draw custom graphics and keyframe indicators on the After Effects TimeGraph?

No, the After Effects SDK does not provide buffers or APIs for drawing on the TimeGraph or timeline. You can only draw into buffers that AE provides for the composition window, layer window, and effects controls window. Custom parameter UIs created in the Effects Controls Window will not appear in the timeline. As an alternative, you can create a custom UI using an arbitrary data type parameter that displays a synchronized alternative timeline view centered on the current time cursor, allowing users to edit your special keyframes with custom interpolation behavior.

*Tags: `ui`, `params`, `arb-data`, `aegp`, `sdk`*

---

## What is the best way to create interactive UI elements like draggable masks in After Effects plugins?

Use a custom UI overlay rather than rendering the mask as part of the image. Rendering the mask in the image has several drawbacks: it gets affected by subsequent effects and display channels, is confined to the layer's size, and forces re-renders when toggling visibility. A custom UI is necessary for interactivity since After Effects won't report click events otherwise. The CCU (Custom Composition UI) sample in the After Effects SDK provides an excellent starting point for creating interactive interfaces in the composition window.

*Tags: `ui`, `custom-ui`, `interactive`, `reference`, `sdk`*

---

## Where can I find example code for building custom interactive UI in After Effects composition windows?

The After Effects SDK includes a CCU (Custom Composition UI) sample that demonstrates how to create a simple interactive interface directly in the composition window. This sample is the recommended starting point for developers building custom UIs with interactive elements like draggable shapes and masks.

*Tags: `ui`, `custom-ui`, `sdk`, `reference`, `open-source`*

---
