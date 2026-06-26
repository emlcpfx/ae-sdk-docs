# Q&A: arb-param

**7 entries** tagged with `arb-param`.

---

## How can I continuously update an ECW (Effect Control Window) custom UI drawing while dragging a 2D point parameter?

ECW custom UIs get idle calls on regular intervals only when the cursor is within their perimeter. However, if the point parameter is supervised, you can call PF_RefreshAllWindows during interactions, which will force a redraw. It's somewhat wasteful, but it works.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-11-01 · Tags: `ecw`, `custom-ui`, `arb-param`, `refresh`, `drag-interaction`*

---

## How can I get the string value of an effect's ARBITRARY parameter data through ExtendScript?

One approach: (1) Add a hidden supervised checkbox parameter. (2) From JavaScript, leave a message in global scope, then toggle the checkbox to trigger USER_CHANGED_PARAM. (3) From C side, use AEGP_ExecuteScript() to check for the JavaScript message, read the string from the arb, and pass it back via AEGP_ExecuteScript(). (4) The string value is then available in JavaScript global scope. An alternative trick is to use hidden UI parameters and encode the string into their controller names, then read the property names from ExtendScript.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/), [**Javier Yang**](../contributors/javier-yang/) · Source: adobe-forum-sdk · 2025-09-01 · Tags: `arb-param`, `extendscript`, `aegp-execute-script`, `string`, `interop`*

---

## How can two different plugin types share arbitrary (ARB) parameter data?

AE blocks expression connections between ARB parameters of different effect types, assuming they can't be guaranteed to have the same structure. On the C side, use AEGP_GetNewStreamValue to read ARB values from other effects. To find the other effect, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH. For consistent identification of a specific effect instance, store an ID in sequence data and access it via AEGP_EffectCallGeneric, or rename an invisible param (though that doesn't survive project reload).

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `arb-param`, `inter-plugin`, `aegp`, `stream-value`, `effect-identification`*

---

## Why doesn't my Arbitrary parameter respect CANNOT_TIME_VARY and COLLAPSE_TWIRL flags?

The flags are placed in the wrong argument position in PF_ADD_ARBITRARY2. PF_ParamFlag_CANNOT_TIME_VARY and PF_ParamFlag_COLLAPSE_TWIRLY are param flags, not param UI flags. They should go in the param_flags argument (one position before where you placed them), not in the ui_flags argument where PF_PUI_CONTROL belongs.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `arb-param`, `param-flags`, `ui-flags`, `cannot-time-vary`, `collapse-twirl`*

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
