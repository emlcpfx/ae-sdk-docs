# Collection

> 1 Q&A · source: AE plugin dev community Discord

### How do I get the name of the currently selected layer using the AE SDK?

Use AEGP_GetNewCollectionFromCompSelection() to get the selection, then traverse it with AEGP_GetCollectionNumItems() and AEGP_GetCollectionItemByIndex(). Each item returns an AEGP_CollectionItemV2 struct with a type field. If type is layer, the 'layer' union member contains a valid AEGP_LayerH. Then use AEGP_GetLayerName() with that handle. Note there may be multiple selected layers.

```cpp
typedef struct {
    AEGP_CollectionItemType type;
    union {
        AEGP_LayerCollectionItem layer;
        AEGP_MaskCollectionItem mask;
        AEGP_EffectCollectionItem effect;
        AEGP_StreamCollectionItem stream;
        AEGP_MaskVertexCollectionItem mask_vertex;
        AEGP_KeyframeCollectionItem keyframe;
    } u;
    AEGP_StreamRefH stream_refH;
} AEGP_CollectionItemV2;
```

*Tags: `aegp`, `collection`, `layer-handle`, `layer-name`, `selection`*

---
