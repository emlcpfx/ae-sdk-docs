# Alpha

> 2 Q&As · source: AE plugin dev community Discord

### Is there a way to enable the alpha channel in a COLOR parameter's color picker?

The standard AE color parameter does not support alpha. PF_AppColorPickerDialog might show the system dialog which could have an alpha component, but probably doesn't. The practical solution is to implement your own color picking dialog launched via param supervision or a custom UI, or simply add a separate opacity parameter alongside the color parameter.

*Tags: `alpha`, `color-param`, `color-picker`, `custom-dialog`, `opacity`*

---

### What flag should be set in global setup to handle premultiplied alpha correctly?

Set the flag PF_outflags2_REVEALS_ZERO_ALPHA in global setup to properly handle cases where the plugin reveals zero alpha, which is related to the premultiplied alpha of the input layer.

*Tags: `alpha`, `mfr`, `params`, `pipl`*

---
