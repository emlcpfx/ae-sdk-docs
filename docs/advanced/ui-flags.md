# Ui Flags

> 2 Q&As · source: AE plugin dev community Discord

### How can I hide an effect parameter in After Effects SDK?

Set the AEGP_DynStreamFlag_HIDDEN flag during PF_Cmd_UPDATE_PARAMS_UI. It should get called when a project loads and the effect is shown for the first time. Note that PF_PUI_NO_ECW_UI in ui_flags only removes it from the effect window but not from the timeline. There is a known AE bug where duplicating an effect may cause the hidden parameter's owner-drawn portion to still appear in some cases.

*Tags: `dynamic-stream`, `hidden`, `parameter`, `ui-flags`, `update-params-ui`*

---

### Why doesn't my Arbitrary parameter respect CANNOT_TIME_VARY and COLLAPSE_TWIRL flags?

The flags are placed in the wrong argument position in PF_ADD_ARBITRARY2. PF_ParamFlag_CANNOT_TIME_VARY and PF_ParamFlag_COLLAPSE_TWIRLY are param flags, not param UI flags. They should go in the param_flags argument (one position before where you placed them), not in the ui_flags argument where PF_PUI_CONTROL belongs.

*Tags: `arb-param`, `cannot-time-vary`, `collapse-twirl`, `param-flags`, `ui-flags`*

---
