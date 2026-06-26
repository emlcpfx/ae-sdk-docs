# Instance Id

> 2 Q&As · source: AE plugin dev community Discord

### How can I set a parameter value when an effect is first applied (during ParamSetup or SequenceSetup)?

ParamSetup is called once per session per effect type, not per instance, so you can't set instance-specific values there. SequenceSetup also lacks a specific instance association - the params array contains junk data and acquiring an AEGP_EffectRef will crash. The solution is to set a flag in new sequence data saying 'this is a brand new instance', then check that flag during idle_hook or UPDATE_PARAMS_UI and make changes accordingly. For unique instance IDs, use sequence data as each sequence setup call is triggered only once per instance when created.

*Tags: `initialization`, `instance-id`, `param-setup`, `sequence-data`, `sequence-setup`*

---

### Why is PF_Cmd_SEQUENCE_FLATTEN called when an effect is first applied, and why are Sequence Resetup/Setdown called multiple times on different threads?

This is correct behavior when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set. AE flattens sequence data so it can be copied (for render thread copies). Since AE 2015, rendering uses separate threads with separate project copies. Resetup may receive unflat data, so tag your data to indicate flat/unflat state. Use PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA to allow AE to get the flat copy without losing the original unflat one, reducing unnecessary resetup calls. During sequence setup/resetup, don't assume the plugin can find its own layer/comp - AE sometimes constructs the instance before associating it to a layer. For unique instance IDs, flag sequence data as 'requires checking' on resetup, then scan for duplicates when the instance can be located.

*Tags: `flatten`, `instance-id`, `mfr`, `resetup`, `sequence-data`, `threading`*

---
