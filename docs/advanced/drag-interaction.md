# Drag Interaction

> 1 Q&A · source: AE plugin dev community Discord

### How can I continuously update an ECW (Effect Control Window) custom UI drawing while dragging a 2D point parameter?

ECW custom UIs get idle calls on regular intervals only when the cursor is within their perimeter. However, if the point parameter is supervised, you can call PF_RefreshAllWindows during interactions, which will force a redraw. It's somewhat wasteful, but it works.

*Tags: `arb-param`, `custom-ui`, `drag-interaction`, `ecw`, `refresh`*

---
