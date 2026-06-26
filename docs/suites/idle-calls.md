# Idle Calls

> 1 Q&A · source: AE plugin dev community Discord

### How can I ensure an ECW element receives draw calls and redraws consistently?

ECW UI elements receive idle calls only when the cursor is within their borders. If that's sufficient, you can use the native ECW idle calls. For redrawing regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows(). For softer redraw methods when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `ecw`, `idle-calls`, `redraw`, `ui`*

---
