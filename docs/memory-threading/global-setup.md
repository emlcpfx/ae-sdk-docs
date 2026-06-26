# Global Setup

> 5 Q&As · source: AE plugin dev community Discord

### When is PF_Cmd_GLOBAL_SETUP called - during AE loading or when applying the effect?

In After Effects, GlobalSetup is called the first time the user applies the effect to a layer, not during the AE loading screen (during loading, AE only scans PiPLs). In Premiere Pro, GlobalSetup is called when the app is loading. You can show dialogs during GlobalSetup in AE since it happens after the app is fully loaded.

*Tags: `global-setup`, `initialization`, `plugin-lifecycle`, `premiere`*

---

### How do you store a struct in global_data?

Create a pointer to the struct in your global data type, then in GlobalSetup allocate it with new: globaldata->structPointer = new StructName(). Use new/malloc (not smart pointers) since AE does special things with the global_data pointer. Optionally delete it in GlobalSetdown, but it's not strictly necessary since GlobalSetdown only gets called when AE closes and the OS will clean up.

*Tags: `global-data`, `global-setdown`, `global-setup`, `memory-management`*

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

### How should I handle color spaces in Premiere plugins, and can I avoid dealing with multiple color modes?

You can use basic render in 8/16-bit without handling all color spaces -- it works but is not optimized, and you won't get the 32-bit icon next to the plugin. You can choose to support only BGRA and skip VUYA. In GlobalSetup, you select which colorspaces to support. However, 32-bit float is required for color grading workflows to access values outside the 0-1 range. The SDKNoise example in the AE SDK demonstrates both BGRA and VUYA colorspace handling for Premiere.

*Tags: `32-bit`, `bgra`, `color-grading`, `colorspace`, `global-setup`, `premiere`, `vuya`*

---

### How can I programmatically set keyframes on effect parameters when the effect is first applied?

Don't set keyframes during GlobalSetup - there's no effect instance yet (effect_ref is garbage). Instead, try setting keyframes on the first call to UPDATE_PARAMS_UI. To set a value, first get the current stream value using AEGP_GetNewStreamValue, modify the value in the returned AEGP_StreamValue2 struct (e.g., valueP.val.one_d = 20), then use the keyframe suite to add keyframes. If the effect is added via script, it's simpler to set keyframes from the script instead.

*Tags: `global-setup`, `initialization`, `keyframes`, `stream-value`, `update-params-ui`*

---
