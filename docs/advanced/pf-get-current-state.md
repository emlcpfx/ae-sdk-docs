# Pf Get Current State

> 1 Q&A · source: AE plugin dev community Discord

### How do you check if an input frame has changed to avoid unnecessary re-processing?

Use PF_GetCurrentState to detect if an input has changed. In SmartFX, the PreRender/SmartRender pipeline handles this more naturally. You can also use GuidMixInPtr to mix in any data that should trigger re-rendering - if parameters or other state change, mix that into the GUID and AE will know to call SmartRender again. Note: if the layer is continuously rasterized, transforms are applied before your effect receives the input.

*Tags: `caching`, `guid-mixin`, `input-change`, `optimization`, `pf-get-current-state`*

---
