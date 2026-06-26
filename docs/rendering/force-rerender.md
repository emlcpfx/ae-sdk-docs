# Force Rerender

> 3 Q&As · source: AE plugin dev community Discord

### How do you use GuidMixInPtr to force re-rendering of cached frames?

Set the flag PF_OutFlag2_I_MIX_GUID_DEPENDENCIES, then in PreRender call extraP->cb->GuidMixInPtr with changed data. When the mixed-in data changes, AE invalidates the cache for that frame. Check for null pointers on extraP, cb, and GuidMixInPtr. Also check against 0xabababababababab (uninitialized memory pattern). You can mix in timestamps, parameter values, or any data that should trigger re-rendering.

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data, PF_PreRenderExtra* extraP) {
    if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION &&
        extraP && extraP->cb && extraP->cb->GuidMixInPtr &&
        (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab) {
        int r = time(0) + in_data->current_time;
        extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(r), reinterpret_cast<void*>(&r));
    }
    // ...
}
```

*Tags: `cache-invalidation`, `force-rerender`, `guid-mixin`, `pre-render`*

---

### How can I force a re-render from a separate thread (e.g., when receiving WebSocket messages)?

It's only possible via idle_hook, which runs on the UI thread (30-50 times per second). Check your other thread for messages during idle events. Options to force re-render: (1) Change a hidden/invisible parameter value - this forces AE to re-render. (2) Call the command number to purge the cache. (3) Call AEGP_EffectCallGeneric to have the effect instance act on itself. You cannot set out_data->out_flags from a separate thread - it must happen on the main UI thread during an AE callback.

*Tags: `force-rerender`, `hidden-param`, `idle-hook`, `threading`, `ui-thread`, `websocket`*

---

### How do you display a progress banner (like 'Computing: X%') that updates during SmartRender in After Effects?

The technique involves using a hidden parameter to trigger re-renders. When the user presses a button, code changes a hidden parameter which triggers AE to call SmartRender again. Each SmartRender call checks the current state and draws the appropriate banner. Adobe's internal 3D Camera Tracker uses PF_Cmd_RESERVED3 (an undocumented idle command selector that AE calls repeatedly) to check background work status and sets PF_OutFlag_FORCE_RERENDER to trigger SmartRender calls. Parameter values can only be changed during PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_EVENT. An alternative is to use AEGP_RegisterIdleHook with hidden parameter changes, though you don't have access to PF_OutData (and thus PF_OutFlag_FORCE_RERENDER) in an idle hook. In Premiere Pro, PF_OutFlag_FORCE_RERENDER doesn't work, so the hidden parameter trick is the recommended approach.

*Tags: `force-rerender`, `hidden-parameter`, `idle-hook`, `pf-cmd-reserved3`, `progress-bar`, `re-render`, `smartrender`*

---
