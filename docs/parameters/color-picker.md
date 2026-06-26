# Color Picker

> 2 Q&As · source: AE plugin dev community Discord

### Is there a way to enable the alpha channel in a COLOR parameter's color picker?

The standard AE color parameter does not support alpha. PF_AppColorPickerDialog might show the system dialog which could have an alpha component, but probably doesn't. The practical solution is to implement your own color picking dialog launched via param supervision or a custom UI, or simply add a separate opacity parameter alongside the color parameter.

*Tags: `alpha`, `color-param`, `color-picker`, `custom-dialog`, `opacity`*

---

### How can you enable an alpha channel in a COLOR parameter for user editing?

The native COLOR parameter in After Effects does not support alpha channel editing in the color picker. According to community experts, you cannot enable this directly. However, you have two workarounds: (1) Use PF_AppColorPickerDialog to access the system color dialog, though it typically won't have an alpha component either, or (2) Implement your own custom color picking dialog and launch it via param supervision or custom UI. Alternatively, consider adding a separate opacity parameter instead of trying to include alpha in the color picker.

*Tags: `color-picker`, `params`, `sdk`, `ui`*

---
