# Effect Identification

> 1 Q&A · source: AE plugin dev community Discord

### How can two different plugin types share arbitrary (ARB) parameter data?

AE blocks expression connections between ARB parameters of different effect types, assuming they can't be guaranteed to have the same structure. On the C side, use AEGP_GetNewStreamValue to read ARB values from other effects. To find the other effect, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH. For consistent identification of a specific effect instance, store an ID in sequence data and access it via AEGP_EffectCallGeneric, or rename an invisible param (though that doesn't survive project reload).

*Tags: `aegp`, `arb-param`, `effect-identification`, `inter-plugin`, `stream-value`*

---
