# Param Visibility

> 1 Q&A · source: AE plugin dev community Discord

### How do you toggle parameter visibility (show/hide) in an AE plugin?

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

*Tags: `aegp`, `custom-ui`, `dynamic-stream`, `param-visibility`, `pipl`*

---
