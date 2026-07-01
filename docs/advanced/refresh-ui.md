# Refresh Ui

> 2 entries · sources: AE plugin dev community Discord; VFX Tools production incident

### Why doesn't PF_Cmd_UPDATE_PARAMS_UI properly update parameter values?

Two issues: (1) During UPDATE_PARAMS_UI, you should only change appearance (hidden, disabled, etc.), not values. Value changes don't play well with AE's instance application scheme. Set proper default values during PARAM_SETUP. (2) Never modify values directly on the original params array. Instead, make a copy of the param struct, modify the copy, and pass it back via PF_UpdateParamUI. Also ensure out_data->out_flags includes PF_OutFlag_REFRESH_UI. See the 'Supervisor' SDK sample project for correct implementation.

*Tags: `param-update`, `pf-update-param-ui`, `refresh-ui`, `supervisor-sample`, `update-params-ui`*

---

### Animating a custom UI by requesting PF_AppRefreshAllWindows from the DRAW handler creates a self-refresh storm (RAM balloons, AE becomes unkillable)

> source: VFX Tools production incident (2026-07)

Symptom: with a custom-UI panel merely exposed (playhead parked, no user input), CPU stays pegged, RAM climbs without bound, and eventually AE runs out of memory and wedges - unkillable, because the main thread is stuck in the loop. On a beta/debug build the heap gives out as a "Press OK after attaching the debugger" assert.

Cause: a `PF_Event_DRAW` handler that asks for a full UI refresh to drive continuous animation - either calling `PF_AppSuite`'s `PF_AppRefreshAllWindows` directly, or setting a flag that an AEGP idle hook turns into that call. The refresh repaints every window, which re-fires your DRAW, which requests another refresh. It free-runs at your throttle rate (often ~20 fps) forever, force-feeding AE's frame/UI caches until memory is gone.

Why the obvious guard fails: the natural fix is "ignore draws that my own refresh caused" - set an `in_refresh` bool around the refresh call and skip re-arming while it is set. This does NOT work. `PF_AppRefreshAllWindows` only invalidates/posts the repaint (WM_PAINT on Windows is posted, not sent), so your DRAW is delivered on a LATER event-loop turn, after `in_refresh` has already cleared. The self-generated draw then looks like a genuine user draw, keeps any "drawn recently" heartbeat fresh, and the loop never terminates. Any stop condition a self-draw can re-satisfy is broken.

Fix - gate on a signal your own refresh cannot regenerate: stamp a `last_interaction` timestamp ONLY from genuine input (`PF_Cmd_USER_CHANGED_PARAM`, and/or real `DO_CLICK`/`DRAG`/keyboard events in your UI), never from `PF_Event_DRAW`. In the DRAW handler, request a refresh only while `now - last_interaction < window` (e.g. 1.5 s). Because draws never advance the timestamp, the animation plays while the user adjusts a control and stops within `window` after they stop, and the loop cannot sustain itself. If you do not actually need idle animation, the simplest fix is to request no refresh from DRAW at all and repaint only on the invalidations AE already sends.

Rule of thumb: `PF_AppRefreshAllWindows` is for reflecting a state change you just made, not an animation clock. Never call it (or arm an idle hook that calls it) unconditionally from a DRAW handler.

*Tags: `pf-refresh-all-windows`, `pf-app-suite`, `custom-ecw-ui`, `drawbot`, `idle-hook`, `wm-paint`, `animation`, `memory-leak`, `unkillable`*

---
