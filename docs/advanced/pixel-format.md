# Pixel_Format

> 1 Q&A · source: AE plugin dev community Discord

### How can I detect when a composition's color depth or pixel format is changed by the user?

There is no direct event for color depth changes, but the frame is re-rendered when the comp depth is changed. You can check the current pixel format using PF_GetPixelFormat() during the render call. To update parameter UI in response, use PF_UpdateParamUI during the render call, and you can hide/unhide parameters at any time using AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN. Parameter property changes can be made when receiving events or PARAMS_UI_CHANGED.

*Tags: `aegp`, `bit_depth`, `params`, `pixel_format`, `ui`*

---
