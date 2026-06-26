# Idle Event

> 1 Q&A · source: AE plugin dev community Discord

### What is the correct way to change a parameter value during PF_Event_IDLE?

According to Adobe SDK documentation, parameter values can only be modified during PF_Cmd_USER_CHANGED_PARAM and specific PF_Cmd_EVENT types (PF_Event_DO_CLICK, PF_Event_DRAG, PF_Event_KEYDOWN). To change parameters outside these events, use AEGP_SetStreamValue() from the AEGP suite. If the parameter has keyframes, use the keyframe suite instead, though AEGP_SetStreamValue() will still work despite what the documentation states. AEGP_SetStreamValue() is available in all After Effects versions since 2000.

```cpp
params[someindex]->u.fs_d.value = somevalue;
params[someindex]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `aegp`, `idle-event`, `params`, `sdk`*

---
