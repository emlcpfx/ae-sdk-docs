# Comp Window

> 2 Q&As · source: AE plugin dev community Discord

### How do I get the mouse position in the comp window from a plugin?

The mouse coordinates are part of the data passed to the plugin on a UI event call. Look at the 'CCU' SDK sample project - it shows how to get the mouse click location in comp window coordinates and how to convert those into layer source coordinates. This is only available during custom UI event callbacks.

*Tags: `ccu-sample`, `comp-window`, `coordinates`, `custom-ui`, `mouse-position`*

---

### Can you disable the crosshair/visual representation of point parameters in the AE comp window?

No, the crosshairs are the visual representation of point parameters and cannot be disabled. As a workaround, you could use a different parameter type like an int or float param pair for x and y values instead of a point parameter.

*Tags: `comp-window`, `crosshair`, `point-parameters`, `ui`, `workaround`*

---
