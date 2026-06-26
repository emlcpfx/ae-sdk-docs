# Parameter Sync

> 1 Q&A · source: AE plugin dev community Discord

### How can I synchronize two parameters so that changing one updates the other (e.g., radius and area)?

Full sync is not entirely possible due to AE's architecture. Problems include: (1) keyframable params may have contradictory interpolation settings, (2) 'current time' is ambiguous with nested comps and multi-frame rendering, (3) since AE 2015, you cannot change param values from the render thread. Workarounds: (1) Use an expression on one param, set programmatically via AEGP_SetExpression during UPDATE_PARAMS_UI. (2) Use PF_PUI_STD_CONTROL_ONLY to make a param non-keyframable. (3) Use PF_PUI_DISABLED to prevent manual changes while still allowing expressions. To get param values without downsample scaling, use AEGP_GetNewStreamValue instead of regular checkout.

*Tags: `downsample`, `expressions`, `keyframes`, `parameter-sync`, `update-params-ui`*

---
