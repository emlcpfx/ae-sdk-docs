# Guid Mixin

> 3 Q&As · source: AE plugin dev community Discord

### How do you use GuidMixInPtr to force re-rendering of cached frames?

Set the flag PF_OutFlag2_I_MIX_GUID_DEPENDENCIES, then in PreRender call extraP->cb->GuidMixInPtr with changed data. When the mixed-in data changes, AE invalidates the cache for that frame. Check for null pointers on extraP, cb, and GuidMixInPtr. Also check against 0xabababababababab (uninitialized memory pattern). You can mix in timestamps, parameter values, or any data that should trigger re-rendering.

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data, PF_PreRenderExtra* extraP) {
    if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION &&
        extraP && extraP->cb && extraP->cb->GuidMixInPtr &&
        (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab) {
        int r = time(0) + in_data->current_time;
        extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(r), reinterpret_cast<void*>(&r));
    }
    // ...
}
```

*Tags: `cache-invalidation`, `force-rerender`, `guid-mixin`, `pre-render`*

---

### Why might SmartRender never be called even though PreRender returns PF_Err_NONE?

Possible causes: (1) GuidMixInPtr in PreRender is telling AE 'no new render needed' if the mixed-in data hasn't changed. (2) Mismatched SDK versions - building with 2025.2 SDK source but 2023 SDK headers causes undefined behavior, especially in Release builds where optimizations may expose the mismatch. Always ensure your SDK headers and source files are from the same version.

*Tags: `guid-mixin`, `pre-render`, `release-build`, `sdk-version`, `smart-render`*

---

### How do you check if an input frame has changed to avoid unnecessary re-processing?

Use PF_GetCurrentState to detect if an input has changed. In SmartFX, the PreRender/SmartRender pipeline handles this more naturally. You can also use GuidMixInPtr to mix in any data that should trigger re-rendering - if parameters or other state change, mix that into the GUID and AE will know to call SmartRender again. Note: if the layer is continuously rasterized, transforms are applied before your effect receives the input.

*Tags: `caching`, `guid-mixin`, `input-change`, `optimization`, `pf-get-current-state`*

---
