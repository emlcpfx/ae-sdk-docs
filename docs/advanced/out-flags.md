# Out Flags

> 2 Q&As · source: AE plugin dev community Discord

### How do you keep PiPL out_flags in sync with GlobalSetup flags?

Define the flags as macros in your plugin header file (e.g., #define FX_OUT_FLAGS (...)) and use those same macros in both your PiPL .r file and GlobalSetup code. The PiPL should include the header with all bit flags defined so it auto-updates when compiling. However, note that this approach may fail with higher bits/newest flags due to overflow in Adobe's PiPL compiler. An online tool for calculating flag bitmasks is available at https://reduxfx.com/aeoutflags.htm.

```cpp
#define FX_OUT_FLAGS (PF_OutFlag_FORCE_RERENDER + \
                       PF_OutFlag_USE_OUTPUT_EXTENT + \
                       PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                       PF_OutFlag_DEEP_COLOR_AWARE + \
                       PF_OutFlag_CUSTOM_UI + \
                       PF_OutFlag_I_DO_DIALOG)

// In PiPL:
AE_Effect_Global_OutFlags { FX_OUT_FLAGS },
AE_Effect_Global_OutFlags_2 { FX_OUT_FLAGS2 },
```

*Tags: `build-system`, `macros`, `out-flags`, `pipl`*

---

### How do you set per-host out_flags when supporting both AE and Premiere?

PiPL doesn't support per-host flags. Instead, set different flags in GlobalSetup by checking in_data->appl_id. For example, to use PF_OutFlag_NON_PARAM_VARY only in AE: check if(in_data->appl_id != 'PrMr') before setting the flag. Note: PiPLs are no longer used by Premiere Pro, so you can write PiPL for AE only. Premiere uses PluginDataEntryFunction / PF_REGISTER_EFFECT instead.

```cpp
if (in_data->appl_id != 'PrMr') {
    out_data->out_flags = AE_OUT_FLAGS;
} else {
    out_data->out_flags = PR_OUT_FLAGS;
}
```

*Tags: `global-setup`, `out-flags`, `per-host`, `pipl`, `premiere`*

---
