# Frame Invalidation

> 1 Q&A · source: AE plugin dev community Discord

### How do you invalidate rendered frames when using smart render with data that changes independently?

Set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag, and in pre-render call extra->cb->GuidMixInPtr with the data to scan. You can call GuidMixInPtr multiple times for different data. The data should be stored in sequence data or a static variable to force render from specific variables.

```cpp
if (extra->cb->GuidMixInPtr && !err) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(data_to_scan_to_force_render), reinterpret_cast<void*>(*data_to_scan_to_force_render));
}
```

*Tags: `frame-invalidation`, `sequence-data`, `smart-render`*

---
