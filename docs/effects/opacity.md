# Opacity

> 1 Q&A · source: AE plugin dev community Discord

### Is there a way to enable the alpha channel in a COLOR parameter's color picker?

The standard AE color parameter does not support alpha. PF_AppColorPickerDialog might show the system dialog which could have an alpha component, but probably doesn't. The practical solution is to implement your own color picking dialog launched via param supervision or a custom UI, or simply add a separate opacity parameter alongside the color parameter.

*Tags: `alpha`, `color-param`, `color-picker`, `custom-dialog`, `opacity`*

---
