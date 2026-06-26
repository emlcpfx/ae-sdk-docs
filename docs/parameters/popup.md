# Popup

> 4 Q&As · source: AE plugin dev community Discord

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

### Why can I only select the first two items in a PF_ADD_POPUPX dropdown with 3 elements in Premiere, and the third snaps back to 2?

This can be caused by opening a Premiere project with the transition already applied (stale cached state). Applying the transition plugin fresh resolves the issue and the dropdowns work normally. Also ensure that each popup choice string has the pipe separator '|' at the end of each line (except the last), and that you have a proper AEFX_CLR_STRUCT(def) before the popup definition.

```cpp
AEFX_CLR_STRUCT(def);

PF_ADD_POPUPX("Color", 3, 2, 
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Tags: `cache-bug`, `dropdown`, `parameter`, `popup`, `premiere`, `transition`*

---

### How should dynamic dropdown lists be updated in After Effects plugins as of CC2025.2 to avoid crashes and empty display?

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

*Tags: `aegp`, `params`, `popup`, `ui`, `windows`*

---

### How should dynamic dropdown lists be updated in After Effects plugins to avoid crashes in CC 2025.2?

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

*Tags: `aegp`, `params`, `popup`, `ui`, `windows`*

---
