# Redraw

> 2 Q&A · source: AE plugin dev community Discord, PluginPort SuperClone

### How can I ensure an ECW element receives draw calls and redraws consistently?

ECW UI elements receive idle calls only when the cursor is within their borders. If that's sufficient, you can use the native ECW idle calls. For redrawing regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows(). For softer redraw methods when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `ecw`, `idle-calls`, `redraw`, `ui`*

---

### My comp-viewer custom-UI overlay (brush cursor, etc.) tracks smoothly during a button-down DRAG but sits frozen on plain hover. Why, and how do I fix it?

Symptom: a custom-UI overlay drawn in `PF_Event_DRAW`, driven by a `PF_Event_ADJUST_CURSOR` handler that updates the overlay position and sets `event_extra->evt_out_flags = PF_EO_UPDATE_NOW`, redraws every tick while you drag with the button down, but on plain hover (mouse move, no button) the overlay appears frozen and only jumps occasionally.

Root cause: **`PF_EO_UPDATE_NOW` is only a *hint*, and AE coalesces it on hover.** Tracing the event flow shows `ADJUST_CURSOR` firing on every hover move with the position updated and `UPDATE_NOW` set every time, yet `PF_Event_DRAW` fires only sparsely (dozens of cursor moves can pass with zero `DRAW` between them). When the host is busy (e.g. a slow GPU/SmartRender pipeline) it simply drops most of the requested redraws. During a drag a different, per-tick redraw path runs, which is why drag looks smooth and hover does not.

Fix: **force the redraw with `PF_InvalidateRect` instead of relying on `UPDATE_NOW`.** In the `ADJUST_CURSOR` handler (when the overlay should be visible), call:

```cpp
AEGP_SuiteHandler suites(in_data->pica_basicP);
PF_Rect inval = {0, 0, 32000, 32000};  // full comp; or just the brush bbox
suites.AppSuite4()->PF_InvalidateRect(event_extra->contextH, &inval);
```

`PF_InvalidateRect` triggers a `PF_Event_DRAW` to repaint the overlay over the **cached** rendered frame -- it does NOT re-run SmartRender, so it is cheap enough to call on every `ADJUST_CURSOR`. Keep setting `PF_EO_UPDATE_NOW` as well. If full-comp invalidation feels heavy, invalidate just the bounding box around the brush/cursor instead of `{0,0,32000,32000}`. (`PF_OutFlag_REFRESH_UI` on `out_data->out_flags` is a lighter alternative that also triggers a `DRAW`.)

*Tags: `custom-ui`, `adjust-cursor`, `draw`, `hover`, `redraw`, `invalidate-rect`, `update-now`*

---
