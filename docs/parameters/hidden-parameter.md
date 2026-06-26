# Hidden Parameter

> 1 Q&A · source: AE plugin dev community Discord

### How do you display a progress banner (like 'Computing: X%') that updates during SmartRender in After Effects?

The technique involves using a hidden parameter to trigger re-renders. When the user presses a button, code changes a hidden parameter which triggers AE to call SmartRender again. Each SmartRender call checks the current state and draws the appropriate banner. Adobe's internal 3D Camera Tracker uses PF_Cmd_RESERVED3 (an undocumented idle command selector that AE calls repeatedly) to check background work status and sets PF_OutFlag_FORCE_RERENDER to trigger SmartRender calls. Parameter values can only be changed during PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_EVENT. An alternative is to use AEGP_RegisterIdleHook with hidden parameter changes, though you don't have access to PF_OutData (and thus PF_OutFlag_FORCE_RERENDER) in an idle hook. In Premiere Pro, PF_OutFlag_FORCE_RERENDER doesn't work, so the hidden parameter trick is the recommended approach.

*Tags: `force-rerender`, `hidden-parameter`, `idle-hook`, `pf-cmd-reserved3`, `progress-bar`, `re-render`, `smartrender`*

---
