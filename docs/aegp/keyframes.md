# Keyframes

> 4 Q&As · source: AE plugin dev community Discord

### Why do keyframed parameters on adjustment layers in Premiere always return the value at the first keyframe regardless of current time?

This is caused by the PF_OutFlag_NON_PARAM_VARY flag. When this flag is set, keyframes on adjustment layers in Premiere always return the first keyframe value during checkout param, regardless of current time. This affects sliders, angles, checkboxes, and 2D points. The bug occurs in Premiere v23 and v24. Removing the NON_PARAM_VARY flag fixes the issue. In AE, this flag can be replaced using MIX_GUID during the pre_render thread. Note that apart from this bug, the flag does nothing useful in Premiere.

*Tags: `adjustment-layer`, `bug`, `checkout-param`, `keyframes`, `non-param-vary`, `premiere`*

---

### How can I synchronize two parameters so that changing one updates the other (e.g., radius and area)?

Full sync is not entirely possible due to AE's architecture. Problems include: (1) keyframable params may have contradictory interpolation settings, (2) 'current time' is ambiguous with nested comps and multi-frame rendering, (3) since AE 2015, you cannot change param values from the render thread. Workarounds: (1) Use an expression on one param, set programmatically via AEGP_SetExpression during UPDATE_PARAMS_UI. (2) Use PF_PUI_STD_CONTROL_ONLY to make a param non-keyframable. (3) Use PF_PUI_DISABLED to prevent manual changes while still allowing expressions. To get param values without downsample scaling, use AEGP_GetNewStreamValue instead of regular checkout.

*Tags: `downsample`, `expressions`, `keyframes`, `parameter-sync`, `update-params-ui`*

---

### How can I programmatically set keyframes on effect parameters when the effect is first applied?

Don't set keyframes during GlobalSetup - there's no effect instance yet (effect_ref is garbage). Instead, try setting keyframes on the first call to UPDATE_PARAMS_UI. To set a value, first get the current stream value using AEGP_GetNewStreamValue, modify the value in the returned AEGP_StreamValue2 struct (e.g., valueP.val.one_d = 20), then use the keyframe suite to add keyframes. If the effect is added via script, it's simpler to set keyframes from the script instead.

*Tags: `global-setup`, `initialization`, `keyframes`, `stream-value`, `update-params-ui`*

---

### How can an After Effects plugin programmatically create keyframes on its own parameters when a button is pressed?

During the USER_CHANGED_PARAM event for the button parameter, use AEGP_GetNewEffectForEffect to get the effect reference, then use AEGP_GetNewEffectStreamByIndex with the target parameter index to get the parameter stream. Once you have the stream reference, you can use keyframe manipulation functions to set keyframes as needed.

*Tags: `aegp`, `keyframes`, `params`, `sdk`*

---
