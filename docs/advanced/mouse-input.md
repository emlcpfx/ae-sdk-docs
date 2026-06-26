# Mouse Input

> 1 Q&A · source: AE plugin dev community Discord

### How can I get the mouse position relative to the composition when the user clicks in the comp preview window?

Mouse coordinates are passed to the plugin through UI event callbacks. You can get the mouse click location in composition window coordinates and convert those into layer source coordinates. The CCU SDK sample project demonstrates how to implement this functionality with working code examples.

*Tags: `coords`, `mouse-input`, `reference`, `sdk`, `ui`*

---
