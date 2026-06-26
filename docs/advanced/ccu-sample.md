# Ccu Sample

> 1 Q&A · source: AE plugin dev community Discord

### How do I get the mouse position in the comp window from a plugin?

The mouse coordinates are part of the data passed to the plugin on a UI event call. Look at the 'CCU' SDK sample project - it shows how to get the mouse click location in comp window coordinates and how to convert those into layer source coordinates. This is only available during custom UI event callbacks.

*Tags: `ccu-sample`, `comp-window`, `coordinates`, `custom-ui`, `mouse-position`*

---
