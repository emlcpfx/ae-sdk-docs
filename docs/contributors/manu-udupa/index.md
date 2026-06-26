# Manu Udupa

**2 contributions** to AE SDK community knowledge.

Top topics: `masks`, `pf-path`, `outflags`, `render-invalidation`, `open-path`, `smartrender`, `progress-bar`, `re-render`, `hidden-parameter`, `force-rerender`

---

## How do you display a progress banner (like 'Computing: X%') that updates during SmartRender in After Effects?

The technique involves using a hidden parameter to trigger re-renders. When the user presses a button, code changes a hidden parameter which triggers AE to call SmartRender again. Each SmartRender call checks the current state and draws the appropriate banner. Adobe's internal 3D Camera Tracker uses PF_Cmd_RESERVED3 (an undocumented idle command selector that AE calls repeatedly) to check background work status and sets PF_OutFlag_FORCE_RERENDER to trigger SmartRender calls. Parameter values can only be changed during PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_EVENT. An alternative is to use AEGP_RegisterIdleHook with hidden parameter changes, though you don't have access to PF_OutData (and thus PF_OutFlag_FORCE_RERENDER) in an idle hook. In Premiere Pro, PF_OutFlag_FORCE_RERENDER doesn't work, so the hidden parameter trick is the recommended approach.

*Source: aescripts discord · 2026-03-05 · Tags: `smartrender`, `progress-bar`, `re-render`, `hidden-parameter`, `force-rerender`, `pf-cmd-reserved3`, `idle-hook` · [View in Q&A](../qa/smartrender/)*

---

## Why does the render not refresh automatically when drawing/deleting an open path/mask with PF_Path suites?

You need to set the flag PF_OutFlag2_DEPENDS_ON_UNREFERENCED_MASKS. This tells After Effects to invalidate your output when a mask is modified that doesn't appear to be referenced by your effect. Closed masks automatically refresh because they alter the input layer pixels before they reach your plugin (AE must call the plugin again since input changed). Open masks don't alter anything normally, so AE thinks there's no reason to re-render.

*Source: aescripts discord · 2025-09-03 · Tags: `masks`, `pf-path`, `outflags`, `render-invalidation`, `open-path` · [View in Q&A](../qa/masks/)*

---
