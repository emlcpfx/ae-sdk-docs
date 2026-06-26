# Cannot Time Vary

> 1 Q&A · source: AE plugin dev community Discord

### Why doesn't my Arbitrary parameter respect CANNOT_TIME_VARY and COLLAPSE_TWIRL flags?

The flags are placed in the wrong argument position in PF_ADD_ARBITRARY2. PF_ParamFlag_CANNOT_TIME_VARY and PF_ParamFlag_COLLAPSE_TWIRLY are param flags, not param UI flags. They should go in the param_flags argument (one position before where you placed them), not in the ui_flags argument where PF_PUI_CONTROL belongs.

*Tags: `arb-param`, `cannot-time-vary`, `collapse-twirl`, `param-flags`, `ui-flags`*

---
