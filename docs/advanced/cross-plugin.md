# Cross Plugin

> 1 Q&A · source: AE plugin dev community Discord

### How can two different plugin types share arbitrary data through their arb parameters?

AE blocks direct expression connections between arb parameters of different plugin types, as it assumes different effects cannot be guaranteed to have the same structure. However, you can read arb values from other effects using AEGP_GetNewStreamValue on the C/C++ side. To reference another plugin instance, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH and effect index. For consistent identification across sessions, store an ID in sequence data and access it via AEGP_EffectCallGeneric, though this has limitations with duplicated effects. Alternatively, rename an invisible parameter, though this won't survive project reloads.

*Tags: `aegp`, `arb-data`, `cross-plugin`, `params`*

---
