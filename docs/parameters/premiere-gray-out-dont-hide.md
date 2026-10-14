---
title: Premiere: gray out params, never hide or show them at runtime
contributor: jonahclarsen
date: 2026-10-14
source: contributed via the /ae-plugin skill
status: unreviewed
---

# Premiere: Gray Out Params, Never Hide or Show Them at Runtime

> Your effect hides and shows controls from `PF_Cmd_UPDATE_PARAMS_UI` with
> `PF_UpdateParamUI` + `PF_PUI_INVISIBLE`. It works in After Effects, and it
> seems to work in Premiere Pro, until a user applies the effect to a clip for
> the first time. On that **first instance** the controls never hide (or never
> come back). A second instance of the same effect behaves.

## Symptom

- Premiere Pro only. The first instance of the effect ignores every runtime
  change to `PF_PUI_INVISIBLE`. Later instances follow it.
- Nothing fails: `PF_UpdateParamUI` returns `PF_Err_NONE`.
- Testing tends to miss it, because the developer's test project usually
  already has an instance applied.

## Cause

An Adobe bug: Premiere ignores changes to `PF_PUI_INVISIBLE` for the first
instance of an effect (Adobe DVARC-3737,
[bug report](https://community.adobe.com/bug-reports-733/premiere-pro-25-to-25-2-beta-ignores-changes-to-pf-pui-invisible-for-first-instance-of-effect-906298),
reported on Premiere Pro 25.0 through the 25.2 beta).

`PF_PUI_DISABLED` is the flag the SDK documents as changeable through
`PF_UpdateParamUI` (`AE_EffectSuites.h`: "ui_flags: PF_PUI_ECW_SEPARATOR,
PF_PUI_DISABLED only"), and Premiere honors it on every instance.

## Fix

In Premiere, never toggle visibility at runtime. Gray the param out instead.
Keep hiding in After Effects (AEGP `AEGP_DynStreamFlag_HIDDEN`) and in OFX hosts.

```cpp
// From PF_Cmd_UPDATE_PARAMS_UI.
static PF_Err SetParamShown(PF_InData* in_data, PF_ParamDef* params[],
                            PF_ParamIndex index, bool shown) {
    PF_ParamDef copy = *params[index];          // never modify params[] directly
    if (in_data->appl_id == 'PrMr') {
        // Leave PF_PUI_INVISIBLE exactly as it was registered.
        if (shown) copy.ui_flags &= ~PF_PUI_DISABLED;
        else       copy.ui_flags |=  PF_PUI_DISABLED;
    } else {
        if (shown) copy.ui_flags &= ~PF_PUI_INVISIBLE;
        else       copy.ui_flags |=  PF_PUI_INVISIBLE;
        // In After Effects, also set AEGP_DynStreamFlag_HIDDEN on the stream.
    }
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    return suites.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref, index, &copy);
}
```

Rules that follow from this:

- **A param that must always stay hidden in Premiere** (internal storage,
  an AE-only control, a popup replaced by custom UI) has to be registered with
  `PF_PUI_INVISIBLE` (plus `PF_PUI_NO_ECW_UI` to drop the row) in
  `PF_Cmd_PARAMS_SETUP`, using `in_data->appl_id` to make it host-specific. A
  runtime call cannot hide it reliably.
- **A param that starts hidden in After Effects** should register with
  `PF_PUI_DISABLED` instead of `PF_PUI_INVISIBLE` in Premiere. Otherwise it can
  never appear there, because the runtime code no longer clears `INVISIBLE`.
- `PF_ADD_FLOAT_SLIDERX`, `PF_ADD_CHECKBOXX`, `PF_ADD_TOPICX` and
  `PF_ADD_POPUPX` clear `def` themselves, so set `def.ui_flags` with the non-X
  macro (`PF_ADD_FLOAT_SLIDER` and so on) when registering a param hidden.

## Docs that contradict this

These Q&A entries recommend toggling `PF_PUI_INVISIBLE` at runtime in Premiere
and should point here instead:

- "How can you hide parameters in Premiere within UpdateParamsUI?"
  (`advanced/premiere.md`, `qa/premiere.md`, `qa/params.md`, `qa/ui.md`,
  `parameters/params.md`, `parameters/ui.md`)
- "How do you hide parameters in Premiere Pro using PF_UpdateParamUI?" (same files)
- `platform/premiere-compatibility.md`: "In Premiere, use `PF_UpdateParamUI`
  with `PF_PUI_INVISIBLE`"
