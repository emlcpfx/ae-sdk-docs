# Popup Param

> 1 Q&A · source: AE plugin dev community Discord

### Why does the dynamic dropdown list in AE CC2025.2 show as empty and crash when clicked?

The issue is likely related to how the dropdown parameter is being updated. Instead of using strncpy_s directly during the param_ui thread, use the AEGP suite functions to properly update dropdown parameters. Use AEGP_GetNewEffectStreamByIndex and AEGP_SetStreamValue through the StreamSuite to update dropdown values, which ensures proper AE integration.

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

*Tags: `aegp`, `debugging`, `popup param`, `ui`, `windows`*

---
