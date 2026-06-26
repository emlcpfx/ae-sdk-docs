# Refresh

> 2 Q&As · source: AE plugin dev community Discord

### How can I continuously update an ECW (Effect Control Window) custom UI drawing while dragging a 2D point parameter?

ECW custom UIs get idle calls on regular intervals only when the cursor is within their perimeter. However, if the point parameter is supervised, you can call PF_RefreshAllWindows during interactions, which will force a redraw. It's somewhat wasteful, but it works.

*Tags: `arb-param`, `custom-ui`, `drag-interaction`, `ecw`, `refresh`*

---

### How can I create a progress bar or continuously animated element in the ECW (Effect Control Window)?

ECW UI elements get idle calls only when the cursor is within their borders. For updates regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows() during that event. It's a brute-force approach but gets the job done. For softer redraws when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `animation`, `custom-ui`, `ecw`, `idle-hook`, `progress-bar`, `refresh`*

---
