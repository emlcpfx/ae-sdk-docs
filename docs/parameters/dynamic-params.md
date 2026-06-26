# Dynamic Params

> 1 Q&A · source: AE plugin dev community Discord

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
