# Effect_Api

> 4 Q&As · source: AE plugin dev community Discord

### Is it possible to add or remove effect parameters after the PF_Cmd_PARAM_SETUP event?

No, you cannot add or remove parameters at any time other than the initial setup. The number and count of parameters are fixed. However, you can hide and show parameters dynamically based on user input. An alternative workaround used by plugins like Plexus is to add additional effects to the layer that contain sets of controls, and have the main effect read their parameters during rendering.

*Tags: `effect_api`, `params`, `pipl`*

---

### How can I display a custom image or logo in an effect parameter?

Every parameter can have a custom UI in which you can draw an image. Look at the "Color Grid" sample to see how a custom UI is implemented in the ECW (Effect Custom UI). You can use any parameter type with a custom UI—in the case of Keylight, it appears to be an arbitrary parameter with a custom UI. You may need to simplify the sample if you only want to draw the image without handling clicks and drags.

*Tags: `custom_ecw_ui`, `effect_api`, `params`, `ui`*

---

### How can I set a slider parameter to match the height or width of the layer the plugin is applied to?

PF_Cmd_PARAMS_SETUP occurs before the plugin is applied to the layer, so it cannot read layer parameters or even identify which layer it's on. Instead, use the PF_Cmd_UPDATE_PARAMS_UI callback that occurs afterwards to set parameter values. While the documentation recommends only cosmetic changes during this phase, you can use the Stream Suite to update parameter values programmatically rather than direct parameter array access.

*Tags: `aegp`, `effect_api`, `params`, `sdk`*

---

### How can I convert Pixel Bender code that samples and transforms pixel coordinates to a native After Effects plugin?

You can accomplish coordinate transformation and pixel sampling using the C++ API. The recommended approach is to pull values from input layers rather than push to arbitrary pixels. Iterate through output samples and pull input values from transformed locations. For direct buffer pixel access in RAM, study the CCU sample project. For iterating through output and pulling from other locations, examine the Shifter sample. Note that subpixel interpolation to 4 adjacent pixels is not supported by the standard suite.

*Tags: `c++`, `effect_api`, `pixel_bender`, `reference`, `sampling`*

---
