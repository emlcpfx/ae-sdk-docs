# Param Setup

> 2 Q&As · source: AE plugin dev community Discord

### How does PF_Arbitrary_NEW_FUNC work, and when is it called?

PF_Arbitrary_NEW_FUNC is never called in post-CS6 AE. The way arb data works now: you allocate a default arb handle during ParamSetup. For every new instance, AE sends PF_Arbitrary_COPY_FUNC with your default arb handle. Your arb data won't be zero length because it copies from the default you set up. Follow the ColorGrid SDK example which uses both NEW_FUNC and ParamSetup for this.

*Tags: `arb-data`, `arbitrary-data`, `copy-func`, `new-func`, `param-setup`*

---

### How can I set a parameter value when an effect is first applied (during ParamSetup or SequenceSetup)?

ParamSetup is called once per session per effect type, not per instance, so you can't set instance-specific values there. SequenceSetup also lacks a specific instance association - the params array contains junk data and acquiring an AEGP_EffectRef will crash. The solution is to set a flag in new sequence data saying 'this is a brand new instance', then check that flag during idle_hook or UPDATE_PARAMS_UI and make changes accordingly. For unique instance IDs, use sequence data as each sequence setup call is triggered only once per instance when created.

*Tags: `initialization`, `instance-id`, `param-setup`, `sequence-data`, `sequence-setup`*

---
