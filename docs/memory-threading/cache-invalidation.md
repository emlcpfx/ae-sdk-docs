# Cache Invalidation

> 1 Q&A · source: AE plugin dev community Discord

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
