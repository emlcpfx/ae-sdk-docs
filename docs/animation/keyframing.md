# Keyframing

> 2 Q&As · source: AE plugin dev community Discord

### How do you restrict two float sliders relative to each other (e.g., Start cannot be >= End)?

Modifying param values in UserChangedParam works but causes problems with keyframing: a keyframe with an acceptable value can later become unacceptable, locking the value and preventing keyframe deletion. A better approach is to use the min()/max() pattern: treat the parameters as generic 'endpoints' and in your render code use min(A,B) as Start and max(A,B) as End.

*Tags: `keyframing`, `param-constraints`, `sliders`, `user-changed-param`*

---

### How can you create dependent parameters that work correctly with keyframing and expressions in After Effects plugins?

When a parameter is keyframed or uses expressions, the PF_Cmd_USER_CHANGED_PARAM callback is not called during timeline scrubbing, making it difficult to synchronize dependent parameters. The PF_ParamFlag_SUPERVISE flag works for user-initiated changes but not for time-varying data. If the dependent parameter doesn't affect rendering (PUI_ONLY flag), you can use it. Otherwise, one practical solution is to use Motion Script expressions to express dependencies between sliders and attach these expressions during sequence setup, allowing the effect to use the computed parameters during render.

```cpp
params[SLIDER2]->u.fs_d.value = params[SLIDER1]->u.fs_d.value;
params[SLIDER2]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `aegp`, `expressions`, `keyframing`, `params`, `scripting`*

---
