# Progress Bar

> 3 Q&As · source: AE plugin dev community Discord

### How can I create a progress bar or continuously animated element in the ECW (Effect Control Window)?

ECW UI elements get idle calls only when the cursor is within their borders. For updates regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows() during that event. It's a brute-force approach but gets the job done. For softer redraws when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `animation`, `custom-ui`, `ecw`, `idle-hook`, `progress-bar`, `refresh`*

---

### Is there a way to show a progress bar in Premiere Pro for video filter / AE effect rendering?

In the AE SDK, there is a function for reporting render progress back to the host, which results in the standard AE progress bar being displayed/updated. It's unclear whether Premiere supports this. There is also an unofficial way used by some of Adobe's native effects to display custom progress, but the details may not be publicly disclosed.

*Tags: `premiere-pro`, `progress-bar`, `rendering`, `ui-feedback`*

---

### How do you display a progress banner (like 'Computing: X%') that updates during SmartRender in After Effects?

The technique involves using a hidden parameter to trigger re-renders. When the user presses a button, code changes a hidden parameter which triggers AE to call SmartRender again. Each SmartRender call checks the current state and draws the appropriate banner. Adobe's internal 3D Camera Tracker uses PF_Cmd_RESERVED3 (an undocumented idle command selector that AE calls repeatedly) to check background work status and sets PF_OutFlag_FORCE_RERENDER to trigger SmartRender calls. Parameter values can only be changed during PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_EVENT. An alternative is to use AEGP_RegisterIdleHook with hidden parameter changes, though you don't have access to PF_OutData (and thus PF_OutFlag_FORCE_RERENDER) in an idle hook. In Premiere Pro, PF_OutFlag_FORCE_RERENDER doesn't work, so the hidden parameter trick is the recommended approach.

*Tags: `force-rerender`, `hidden-parameter`, `idle-hook`, `pf-cmd-reserved3`, `progress-bar`, `re-render`, `smartrender`*

---
