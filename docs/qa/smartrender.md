# Q&A: smartrender

**8 entries** tagged with `smartrender`.

---

## Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Contributors: [**Mirza Kadic (EFEKT)**](../contributors/mirza-kadic-efekt/), [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2025-07-10 · Tags: `smartfx`, `prerender`, `smartrender`, `code-organization`, `cpp`, `dll`*

---

## Can the same .aex plugin work in both After Effects and Premiere Pro?

Yes, but it requires correct coding. Premiere does many things differently than AE and not all features are supported in both hosts. The AE SDK documentation has a dedicated section for Premiere Pro compatibility (https://ae-plugins.docsforadobe.dev/). Notably, Premiere does not support SmartRender, which is a significant difference. There are also long-standing bugs in Premiere's implementation of AE plugin APIs (present since at least 2006) that are regularly reported to but largely ignored by Adobe.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2025-12-21 · Tags: `premiere-pro`, `aex`, `cross-host`, `compatibility`, `smartrender`*

---

## How do you display a progress banner (like 'Computing: X%') that updates during SmartRender in After Effects?

The technique involves using a hidden parameter to trigger re-renders. When the user presses a button, code changes a hidden parameter which triggers AE to call SmartRender again. Each SmartRender call checks the current state and draws the appropriate banner. Adobe's internal 3D Camera Tracker uses PF_Cmd_RESERVED3 (an undocumented idle command selector that AE calls repeatedly) to check background work status and sets PF_OutFlag_FORCE_RERENDER to trigger SmartRender calls. Parameter values can only be changed during PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_EVENT. An alternative is to use AEGP_RegisterIdleHook with hidden parameter changes, though you don't have access to PF_OutData (and thus PF_OutFlag_FORCE_RERENDER) in an idle hook. In Premiere Pro, PF_OutFlag_FORCE_RERENDER doesn't work, so the hidden parameter trick is the recommended approach.

*Contributors: [**Tim Constantinov**](../contributors/tim-constantinov/), [**Manu Udupa**](../contributors/manu-udupa/), [**Jonah (Haligonian/Baskl)**](../contributors/jonah-haligonian-baskl/), [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2026-03-05 · Tags: `smartrender`, `progress-bar`, `re-render`, `hidden-parameter`, `force-rerender`, `pf-cmd-reserved3`, `idle-hook`*

---

## How do you generate unique checkout IDs for checkout_layer/checkout_layer_pixels in SmartFX with multiple layers and instances?

This is a known challenge when dealing with multiple layers, multiple instances, and nested loops in SmartFX PreRender/SmartRender. The checkout ID must be unique across all checkouts. No definitive solution was provided in the discussion, but the issue typically arises with many cloned plugin instances resulting in 'checkout id is not unique' errors.

*Source: aescripts discord · 2025-09-13 · Tags: `smartfx`, `checkout-layer`, `unique-id`, `prerender`, `smartrender`, `multiple-instances`*

---

## Is it possible for one effect to render from another effect during smartrender?

Direct effect-to-effect rendering during smartrender is not possible because After Effects will not call smartrender on other layers while a current smartrender is in progress—it waits until the current smartrender is done. Workarounds include: (1) using AEGP outside of render calls but this has poor UI; (2) calling EffectCallGeneric on another plugin if that plugin implements a passthrough code for SmartRender; (3) using AEGP_RenderAndCheckoutLayerFrame_Async which works on any thread but can be buggy; (4) delegating AEGP calls to the UI thread for the next call, though smartrender happens on the render thread and cannot directly call UI thread methods.

*Tags: `smartrender`, `render-loop`, `aegp`, `threading`*

---

## What is AEGP_RenderAndCheckoutLayerFrame_Async and can it be called from smart_render command?

AEGP_RenderAndCheckoutLayerFrame_Async is an asynchronous layer rendering and checkout function that can work on any thread, making it potentially usable from smartrender command, though it is known to be buggy.

*Tags: `smartrender`, `aegp`, `layer-checkout`, `threading`*

---

## What error occurs when trying to adjust effect parameters using AEGP during smartrender?

After Effects gives an error box saying 'error: effect attempting to modify a locked project' when trying to adjust the parameter of an effect using AEGP during a smartrender command.

*Tags: `smartrender`, `aegp`, `params`, `debugging`*

---

## How can I access sequence_data in SmartRender when it appears as nullptr?

In After Effects v22 and later, you must use the PF_EffectSequenceDataSuite to read sequence data in SmartRender instead of directly accessing in_data->sequence_data. Use AEFX_SuiteScoper to obtain the suite, then call PF_GetConstSequenceData to retrieve a const handle. This handle can then be locked with PF_LOCK_HANDLE to access the sequence data structure. Note that this method provides read-only access.

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

*Tags: `smartrender`, `sequence-data`, `aegp`, `mfr`*

---
