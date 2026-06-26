# Cti

> 1 Q&A · source: AE plugin dev community Discord

### How can I change the current time (CTI) in After Effects using an AEGP plugin?

To change the current time (CTI) with a plugin, first get the comp's item from the effect layer, then use AEGP_SetItemCurrentTime. The process involves: (1) using AEGP_GetEffectLayer to get the layer handle, (2) using AEGP_GetLayerParentComp to get the composition handle, (3) using AEGP_GetItemFromComp to get the item handle from the composition, and (4) finally calling AEGP_SetItemCurrentTime with the item handle and desired time value.

```cpp
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &m_layerH);
suites.LayerSuite8()->AEGP_GetLayerParentComp(m_layerH, &m_compH);
suites.CompSuite11()->AEGP_GetItemFromComp(m_compH, &compItemH);
suites.ItemSuite9()->AEGP_SetItemCurrentTime(compItemH, &compTime);
```

*Tags: `aegp`, `cti`, `reference`, `scripting`, `ui`*

---
