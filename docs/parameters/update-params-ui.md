# Update Params Ui

> 6 Q&As · source: AE plugin dev community Discord

### How do you use dynamic dropdown lists (POPUP params) with changing options?

Modifying popup options at runtime requires updating the names pointer string during the UpdateParamsUI thread. One approach uses strncpy_s to modify param_union.pd.u.namesptr. However, this broke in AE CC2025.2 (empty dropdown, crash on click). An alternative approach uses AEGP stream suites to set the dropdown value programmatically via AEGP_SetStreamValue, though this causes undo history issues. This is done in UpdateParamsUI.

```cpp
PF_Err setFloatOrBoolOrDropdownParamViaAEGP(uint16_t i_param_to_change, float val,
    AEGP_PluginID my_id, AEGP_EffectRefH *effect_refHP, AEGP_SuiteHandler *suitesP)
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

*Tags: `aegp-stream`, `dropdown`, `dynamic-params`, `popup`, `update-params-ui`*

---

### How can I hide an effect parameter in After Effects SDK?

Set the AEGP_DynStreamFlag_HIDDEN flag during PF_Cmd_UPDATE_PARAMS_UI. It should get called when a project loads and the effect is shown for the first time. Note that PF_PUI_NO_ECW_UI in ui_flags only removes it from the effect window but not from the timeline. There is a known AE bug where duplicating an effect may cause the hidden parameter's owner-drawn portion to still appear in some cases.

*Tags: `dynamic-stream`, `hidden`, `parameter`, `ui-flags`, `update-params-ui`*

---

### How can I synchronize two parameters so that changing one updates the other (e.g., radius and area)?

Full sync is not entirely possible due to AE's architecture. Problems include: (1) keyframable params may have contradictory interpolation settings, (2) 'current time' is ambiguous with nested comps and multi-frame rendering, (3) since AE 2015, you cannot change param values from the render thread. Workarounds: (1) Use an expression on one param, set programmatically via AEGP_SetExpression during UPDATE_PARAMS_UI. (2) Use PF_PUI_STD_CONTROL_ONLY to make a param non-keyframable. (3) Use PF_PUI_DISABLED to prevent manual changes while still allowing expressions. To get param values without downsample scaling, use AEGP_GetNewStreamValue instead of regular checkout.

*Tags: `downsample`, `expressions`, `keyframes`, `parameter-sync`, `update-params-ui`*

---

### Why doesn't PF_Cmd_UPDATE_PARAMS_UI properly update parameter values?

Two issues: (1) During UPDATE_PARAMS_UI, you should only change appearance (hidden, disabled, etc.), not values. Value changes don't play well with AE's instance application scheme. Set proper default values during PARAM_SETUP. (2) Never modify values directly on the original params array. Instead, make a copy of the param struct, modify the copy, and pass it back via PF_UpdateParamUI. Also ensure out_data->out_flags includes PF_OutFlag_REFRESH_UI. See the 'Supervisor' SDK sample project for correct implementation.

*Tags: `param-update`, `pf-update-param-ui`, `refresh-ui`, `supervisor-sample`, `update-params-ui`*

---

### How can I programmatically set keyframes on effect parameters when the effect is first applied?

Don't set keyframes during GlobalSetup - there's no effect instance yet (effect_ref is garbage). Instead, try setting keyframes on the first call to UPDATE_PARAMS_UI. To set a value, first get the current stream value using AEGP_GetNewStreamValue, modify the value in the returned AEGP_StreamValue2 struct (e.g., valueP.val.one_d = 20), then use the keyframe suite to add keyframes. If the effect is added via script, it's simpler to set keyframes from the script instead.

*Tags: `global-setup`, `initialization`, `keyframes`, `stream-value`, `update-params-ui`*

---

### How can I prevent my effect from being added more than once to the same layer?

There's no outflag for this. The approach: (1) During global setup, find your effect's AEGP_InstalledEffectKey by iterating installed effects with AEGP_GetNumInstalledEffects, AEGP_GetNextInstalledEffect, and AEGP_GetEffectMatchName. (2) During UPDATE_PARAMS_UI (guaranteed to be called after applying), scan the effect's layer and check each effect's install key using AEGP_GetInstalledKeyFromLayerEffect. (3) If duplicates exist, alert the user, delete/disable the redundant, or skip rendering by copying input to output. You can also try AEGP_DisableCommand, but users can still duplicate or copy/paste the effect.

*Tags: `effect-detection`, `install-key`, `matchname`, `single-instance`, `update-params-ui`*

---
