# Inter Plugin

> 2 Q&As · source: AE plugin dev community Discord

### How can two different plugin types share arbitrary (ARB) parameter data?

AE blocks expression connections between ARB parameters of different effect types, assuming they can't be guaranteed to have the same structure. On the C side, use AEGP_GetNewStreamValue to read ARB values from other effects. To find the other effect, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH. For consistent identification of a specific effect instance, store an ID in sequence data and access it via AEGP_EffectCallGeneric, or rename an invisible param (though that doesn't survive project reload).

*Tags: `aegp`, `arb-param`, `effect-identification`, `inter-plugin`, `stream-value`*

---

### How do you properly implement inter-effect communication using AEGP_EffectCallGeneric in After Effects plugins?

When calling an effect from an AEGP (like Sweetie calling Checkout), you must handle timing carefully. The key issue is that you cannot call an effect back immediately while it's still processing its original call to your AEGP. Instead, use idle_hook: when the effect calls your AEGP, set a flag and wait for the next idle_hook call, then initiate AEGP_EffectCallGeneric when the effect is guaranteed to be idle. Alternatively, respond immediately via the same custom suite that did the calling, passing data for the calling effect to process without needing effectCallGeneric. Do not use AEGP_GetNewEffectForEffect with a passed effect_ref—instead, use AEGP_GetLayerEffectByIndex to look up the effect on the target layer. For permanent effect tracking across the project lifetime, store the comp itemID and layer layerID rather than relying on EffectRefH which can become invalid.

```cpp
// From AEGP: wait for idle hook before calling effect back
if (AEFX_AcquireSuite(in_data, out_data, kDuckSuite1, kDuckSuiteVersion1, "Couldn't load suite.", (void**)&dsP)) {
  PF_STRCPY(out_data->return_msg, "No Duck Suite!");
} else {
  if (dsP) {
    dsP->Quack(2);
  }
  AEFX_ReleaseSuite(in_data, out_data, kDuckSuite1, kDuckSuiteVersion1, "Couldn't release suite.");
}

// From Effect: pass in_data to AEGP via custom suite for callback
static SPAPI A_Err CalledFromEffect(void* ptr) {
  PF_InData* pIndata = static_cast<PF_InData*>(ptr);
  // Use AEGP_GetLayerEffectByIndex instead of AEGP_GetNewEffectForEffect
}
```

*Tags: `aegp`, `effect-ref`, `idle-hook`, `inter-plugin`, `reference`, `timing`*

---
