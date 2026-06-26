# Ecw

> 5 Q&As · source: AE plugin dev community Discord

### How can I continuously update an ECW (Effect Control Window) custom UI drawing while dragging a 2D point parameter?

ECW custom UIs get idle calls on regular intervals only when the cursor is within their perimeter. However, if the point parameter is supervised, you can call PF_RefreshAllWindows during interactions, which will force a redraw. It's somewhat wasteful, but it works.

*Tags: `arb-param`, `custom-ui`, `drag-interaction`, `ecw`, `refresh`*

---

### How can I create a progress bar or continuously animated element in the ECW (Effect Control Window)?

ECW UI elements get idle calls only when the cursor is within their borders. For updates regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows() during that event. It's a brute-force approach but gets the job done. For softer redraws when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `animation`, `custom-ui`, `ecw`, `idle-hook`, `progress-bar`, `refresh`*

---

### What is the correct workflow for storing custom data in an arbitrary (ARB) parameter for custom UI interactions?

The workflow should be: user interacts with UI -> data is stored in a LOCAL array -> local array is serialized and stored in an ARB -> AE sees new value and invalidates cached frames. Before displaying UI, read the value from the ARB, deserialize into your local array, and draw accordingly (this ensures undo/redo reflects correctly). Serializing means writing all data into one continuous block of AE-allocated memory. You don't need to convert to a string - use whichever method works for collecting data in and reconstructing out. This is referred to as 'flattening' and 'unflattening' in the AE SDK.

*Tags: `arb-param`, `custom-ui`, `ecw`, `flattening`, `serialization`, `undo-redo`*

---

### How can I continuously update the ECW (Enhanced Custom UI) drawing while dragging a 2D point parameter?

ECW custom UIs receive idle calls on regular intervals only when the cursor is within their perimeter. To force continuous updates during parameter interactions, call PF_RefreshAllWindows during the interaction. This forces a redraw of the ECW, though it is resource-intensive. If the point parameter is supervised, this approach will keep the ARB viewer updated in real-time as the user drags the point.

*Tags: `ecw`, `params`, `render-loop`, `ui`*

---

### How can I ensure an ECW element receives draw calls and redraws consistently?

ECW UI elements receive idle calls only when the cursor is within their borders. If that's sufficient, you can use the native ECW idle calls. For redrawing regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows(). For softer redraw methods when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `ecw`, `idle-calls`, `redraw`, `ui`*

---
