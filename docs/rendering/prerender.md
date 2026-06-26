# Prerender

> 3 Q&As · source: AE plugin dev community Discord

### Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Tags: `code-organization`, `cpp`, `dll`, `prerender`, `smartfx`, `smartrender`*

---

### How do you generate unique checkout IDs for checkout_layer/checkout_layer_pixels in SmartFX with multiple layers and instances?

This is a known challenge when dealing with multiple layers, multiple instances, and nested loops in SmartFX PreRender/SmartRender. The checkout ID must be unique across all checkouts. No definitive solution was provided in the discussion, but the issue typically arises with many cloned plugin instances resulting in 'checkout id is not unique' errors.

*Tags: `checkout-layer`, `multiple-instances`, `prerender`, `smartfx`, `smartrender`, `unique-id`*

---

### How do you use GuidMixInPtr to force re-rendering in After Effects plugins?

Use the GuidMixInPtr callback to mix in a GUID dependency that forces re-renders. First, set the PF_OutFlag2_I_MIX_GUID_DEPENDENCIES flag. Then in PreRender, check that in_data->version.major >= PF_AE130_PLUG_IN_VERSION and verify extraP, extraP->cb, and extraP->cb->GuidMixInPtr are non-null and not the sentinel value 0xabababababababab. Call GuidMixInPtr with a value that changes (e.g., based on time or other conditions) to trigger re-renders. For timestamps, use gettimeofday on non-Windows platforms or GetTickCount64 on Windows, then multiply by 100 and add randomness to ensure uniqueness.

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data, PF_PreRenderExtra* extraP) {
    if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION && extraP && extraP->cb && extraP->cb->GuidMixInPtr && (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab) {
        int r = time(0) + in_data->current_time;
        extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(r), reinterpret_cast<void *>(&r));
    }
}

#ifndef WIN32
struct timeval tp;
gettimeofday(&tp, NULL);
long long dummy = (long long) tp.tv_sec * 1000L + tp.tv_usec / 1000;
#else
long long dummy = GetTickCount64();
#endif
dummy = dummy * 100 + rand() % 100;
extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(dummy), reinterpret_cast<void *>(&dummy));
```

*Tags: `caching`, `cross-platform`, `guid-dependencies`, `prerender`, `render-loop`*

---
