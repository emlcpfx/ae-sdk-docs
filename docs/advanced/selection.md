# Selection

> 2 Q&As · source: AE plugin dev community Discord

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

### How can an AEGP detect which parameter stream is selected in the effect control window?

There is currently no documented way in the After Effects SDK to determine which stream was selected in the effect control window (ECW). Unlike the timeline where you can use the selection collection to identify selected streams, the ECW does not provide equivalent selection APIs. As a workaround, developers typically place custom commands in the Keyframe Assist menu rather than registering them in the right-click context menu, which does not support command registration.

*Tags: `aegp`, `params`, `selection`, `ui`*

---
