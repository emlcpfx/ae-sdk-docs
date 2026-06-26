# Q&A: custom-ui

**20 entries** tagged with `custom-ui`.

---

## How do you toggle parameter visibility (show/hide) in an AE plugin?

Use AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to toggle visibility. The approach using PF_PUI_INVISIBLE with ui_flags and PF_UpdateParamUI is unreliable. Instead, use the AEGP stream suites: get the effect ref via AEGP_GetNewEffectForEffect, get the stream via AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag. Note this only works in AE (check in_data->appl_id != 'PrMr'), not Premiere.

```cpp
static PF_Err
changeParamVisibility(PF_InData *in_data, PF_OutData *out_data,
                      PF_ParamDef *paramsDef, PF_ParamIndex paramIndex,
                      PF_Boolean paramVisibleB)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    global_dataP globP = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr') {
        AEGP_EffectRefH meH = NULL;
        AEGP_StreamRefH currStreamH = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex, &currStreamH));
        if (meH && currStreamH) {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
    }
    return err;
}
```

*Contributors: [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2022-12-07 · Tags: `param-visibility`, `aegp`, `dynamic-stream`, `custom-ui`, `pipl`*

---

## Why does DrawBot draw colors at ~90% brightness compared to what you specify?

This can be caused by using DrawImage with opacity less than 1.0, which makes the drawing darker while alpha still appears as 1.0 (since AE's UI is behind with solid alpha). It could also be related to AE's UI brightness preferences or the project's color space settings, as AE's drawing suites may compensate for those things.

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2023-06-24 · Tags: `drawbot`, `custom-ui`, `color-accuracy`, `brightness`*

---

## How do you implement a text editor in the Effect Control Window of an AE plugin?

Instead of trying to handle PF_Event_KEYDOWNs directly (which has limitations like missing Tab key and no param index), use a button that runs a script via AEGP_ExecuteScript(). The script string can display a text box dialog. Pass data to it by find+replacing in the script string before execution. Store the text in an arb data parameter. The script can return data back to AE.

*Contributors: [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2023-07-31 · Tags: `custom-ui`, `text-input`, `aegp-execute-script`, `arb-data`, `button-param`*

---

## How do you handle Custom UI mouse-leave events to un-hover buttons?

AE has a known bug where no PF_Event is sent when the mouse leaves the Custom UI area, causing hover states to persist. A workaround is to use a debounce pattern: create a timer callback that triggers after a delay (e.g., 2 seconds). On every mouse-over event, reset the timer. When the timer fires (meaning no mouse-over received recently), trigger an UpdateUI on the main thread to force a Custom UI re-render that resets the hover state.

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
```

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2025-07-25 · Tags: `custom-ui`, `mouse-events`, `hover`, `debounce`, `workaround`*

---

## How does AE's Custom UI / arb params bug in AE 2025 manifest?

In AE 2025 (25.0.1), custom UI and arb params display glitches: one arb param's UI can corrupt other arb UIs, causing flashes or scrambled/duplicated content. The likely culprit is NewImageFromBuffer in DrawBot - every call may be setting pixels of all retained DRAWBOT_ImageRef within the same context. Adobe acknowledged fixing something in arb handling, but it may have introduced this unintended consequence. The issue affects multiple third-party plugins including Trapcode.

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**rowbyte**](../contributors/rowbyte/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2024-12-10 · Tags: `custom-ui`, `arb-data`, `drawbot`, `ae-2025`, `bug`, `regression`*

---

## How can I continuously update an ECW (Effect Control Window) custom UI drawing while dragging a 2D point parameter?

ECW custom UIs get idle calls on regular intervals only when the cursor is within their perimeter. However, if the point parameter is supervised, you can call PF_RefreshAllWindows during interactions, which will force a redraw. It's somewhat wasteful, but it works.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-11-01 · Tags: `ecw`, `custom-ui`, `arb-param`, `refresh`, `drag-interaction`*

---

## How does the AddArc function work in the Drawbot Path Suite, and why do two consecutive arcs produce unexpected shapes?

When drawing multiple arcs/circles in the same path, you need to call close() on the path after each circle. Without closing, the entire sequence is treated as one continuous path, and the renderer draws a connecting line from the end point of one arc back to the beginning of the next, creating unexpected shapes. You can have multiple closed shapes in one path, or alternatively draw each shape in separate paths with separate drawing calls.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `drawbot`, `path-suite`, `addarc`, `custom-ui`, `drawing`*

---

## How can I create a progress bar or continuously animated element in the ECW (Effect Control Window)?

ECW UI elements get idle calls only when the cursor is within their borders. For updates regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows() during that event. It's a brute-force approach but gets the job done. For softer redraws when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `ecw`, `progress-bar`, `idle-hook`, `refresh`, `custom-ui`, `animation`*

---

## How do I get the mouse position in the comp window from a plugin?

The mouse coordinates are part of the data passed to the plugin on a UI event call. Look at the 'CCU' SDK sample project - it shows how to get the mouse click location in comp window coordinates and how to convert those into layer source coordinates. This is only available during custom UI event callbacks.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2022-01-01 · Tags: `mouse-position`, `comp-window`, `custom-ui`, `ccu-sample`, `coordinates`*

---

## How do I work with multiple arbitrary (ARB) parameters in an AE effect?

Each ARB param is its own thing with its own routines. Use ARB_REFCON to route handling: it's pointer-sized and can point to a function or class instance that handles that specific param. This eliminates the need for switch statements. Each ARB needs a unique disk ID and a separately allocated default handle. PF_CustomUIInfo registration only needs to be done once per PARAM_SETUP call (not per param). For a scalable approach, create a base class for generic ARB handling and override specific methods per param type. The refconPV content is entirely up to you - AE is indifferent to it.

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

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `arb-param`, `multiple-params`, `refcon`, `custom-ui`, `class-design`, `param-routing`*

---

## What is the correct workflow for storing custom data in an arbitrary (ARB) parameter for custom UI interactions?

The workflow should be: user interacts with UI -> data is stored in a LOCAL array -> local array is serialized and stored in an ARB -> AE sees new value and invalidates cached frames. Before displaying UI, read the value from the ARB, deserialize into your local array, and draw accordingly (this ensures undo/redo reflects correctly). Serializing means writing all data into one continuous block of AE-allocated memory. You don't need to convert to a string - use whichever method works for collecting data in and reconstructing out. This is referred to as 'flattening' and 'unflattening' in the AE SDK.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `arb-param`, `serialization`, `custom-ui`, `undo-redo`, `flattening`, `ecw`*

---

## What is the best practice to spawn a window with text input from an effect's parameters?

Two separate concerns: (1) Getting user input: Use AEGP_ExecuteScript to launch a JavaScript prompt - it takes text input and returns whether the user hit OK or Cancel. For fancier UI, use OS-level windows (no third-party library needed). (2) Storing the data: Use an ARB parameter which allows undo/redo. You can have a button param launch the prompt and store the string in a hidden ARB, or combine both into one ARB with a custom UI that shows the current string AND is clickable.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `text-input`, `dialog`, `aegp-execute-script`, `arb-param`, `custom-ui`, `button-param`*

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

## How can you implement a custom text editor in the Effect Control Window that captures key events like Tab?

When implementing a custom text editor in the ECW using Custom UI events, PF_Event_KEYDOWN events are available but don't include an index for the particular param. Additionally, some keys like Tab don't reach the KEYDOWN handler. To solve this, you need to claim key-focus similar to how built-in slider numeric value editing works, which allows the plugin to intercept all keyboard input including Tab and other special keys that would normally be consumed by the host.

*Tags: `ui`, `custom-ui`, `aegp`, `debugging`*

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

## How can I draw custom curves outside the composition bounds in After Effects?

To draw custom UI elements like curves outside the composition, you need to use PF_Event and Drawbot APIs. Look up "PF_Event" and "Drawbot" in the After Effects SDK guide for implementation details.

*Tags: `ui`, `aegp`, `custom-ui`, `drawing`*

---

## Why can't I draw a custom UI in PF_Window_LAYER even though it works in PF_Window_COMP?

This was a bug in After Effects CS3 and CS4 where custom UI drawing in layer windows was not supported. The plug-in would receive PF_Event_DRAW calls and return a buffer, but After Effects would ignore the rendered output. This issue was fixed in CS5. As a workaround, you may need to upgrade to CS5 or later, or reference how other plugins like 'meshwarp' or 'vector paint' handle this limitation.

*Tags: `ui`, `custom-ui`, `debugging`, `ae-versions`*

---

## What is the best way to create interactive UI elements like draggable masks in After Effects plugins?

Use a custom UI overlay rather than rendering the mask as part of the image. Rendering the mask in the image has several drawbacks: it gets affected by subsequent effects and display channels, is confined to the layer's size, and forces re-renders when toggling visibility. A custom UI is necessary for interactivity since After Effects won't report click events otherwise. The CCU (Custom Composition UI) sample in the After Effects SDK provides an excellent starting point for creating interactive interfaces in the composition window.

*Tags: `ui`, `custom-ui`, `interactive`, `reference`, `sdk`*

---

## Where can I find example code for building custom interactive UI in After Effects composition windows?

The After Effects SDK includes a CCU (Custom Composition UI) sample that demonstrates how to create a simple interactive interface directly in the composition window. This sample is the recommended starting point for developers building custom UIs with interactive elements like draggable shapes and masks.

*Tags: `ui`, `custom-ui`, `sdk`, `reference`, `open-source`*

---
