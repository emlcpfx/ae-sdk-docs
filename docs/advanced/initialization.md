# Initialization

> 4 Q&As · source: AE plugin dev community Discord

### What causes uninitialized PF_Err variables and subtle bugs?

A common C++ syntax error: 'PF_Err err, err2 = PF_Err_NONE;' only initializes err2, leaving err undefined (often ends up as 1). The correct form is 'PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;'. This can cause if(!err) checks to fail unexpectedly and lead to hard-to-track bugs like AEGP_GetNewEffectStreamByIndex returning 0x0.

```cpp
// WRONG - err is uninitialized:
PF_Err err, err2 = PF_Err_NONE;

// CORRECT - both initialized:
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `c-plus-plus`, `common-mistake`, `debugging`, `initialization`, `pf-err`*

---

### When is PF_Cmd_GLOBAL_SETUP called - during AE loading or when applying the effect?

In After Effects, GlobalSetup is called the first time the user applies the effect to a layer, not during the AE loading screen (during loading, AE only scans PiPLs). In Premiere Pro, GlobalSetup is called when the app is loading. You can show dialogs during GlobalSetup in AE since it happens after the app is fully loaded.

*Tags: `global-setup`, `initialization`, `plugin-lifecycle`, `premiere`*

---

### How can I set a parameter value when an effect is first applied (during ParamSetup or SequenceSetup)?

ParamSetup is called once per session per effect type, not per instance, so you can't set instance-specific values there. SequenceSetup also lacks a specific instance association - the params array contains junk data and acquiring an AEGP_EffectRef will crash. The solution is to set a flag in new sequence data saying 'this is a brand new instance', then check that flag during idle_hook or UPDATE_PARAMS_UI and make changes accordingly. For unique instance IDs, use sequence data as each sequence setup call is triggered only once per instance when created.

*Tags: `initialization`, `instance-id`, `param-setup`, `sequence-data`, `sequence-setup`*

---

### How can I programmatically set keyframes on effect parameters when the effect is first applied?

Don't set keyframes during GlobalSetup - there's no effect instance yet (effect_ref is garbage). Instead, try setting keyframes on the first call to UPDATE_PARAMS_UI. To set a value, first get the current stream value using AEGP_GetNewStreamValue, modify the value in the returned AEGP_StreamValue2 struct (e.g., valueP.val.one_d = 20), then use the keyframe suite to add keyframes. If the effect is added via script, it's simpler to set keyframes from the script instead.

*Tags: `global-setup`, `initialization`, `keyframes`, `stream-value`, `update-params-ui`*

---
