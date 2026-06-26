# Coordinates

> 2 Q&As · source: AE plugin dev community Discord

### How do I get the mouse position in the comp window from a plugin?

The mouse coordinates are part of the data passed to the plugin on a UI event call. Look at the 'CCU' SDK sample project - it shows how to get the mouse click location in comp window coordinates and how to convert those into layer source coordinates. This is only available during custom UI event callbacks.

*Tags: `ccu-sample`, `comp-window`, `coordinates`, `custom-ui`, `mouse-position`*

---

### How do you correctly extract and convert After Effects camera and light matrix data for use in a Vulkan shader?

The user describes a 4-step process: (1) get camera matrix from AE, (2) convert it to column-major format, (3) normalize position in clip space using range (-1,1) with zoom value as z factor, (4) get light matrix and normalize it. However, they report that using model and projection matrix operations hasn't produced expected results. The answer suggests this is a known challenge when working with AE camera data in non-OpenGL contexts like Vulkan, and the correct transformation depends on properly handling the coordinate system conversion between AE's internal representation and the target graphics API.

*Tags: `camera`, `coordinates`, `lighting`, `matrix`, `shader`, `vulkan`*

---
