# Shape

> 2 Q&As · source: AE plugin dev community Discord

### How can I detect if a layer is a shape layer using the AEGP?

Use the function AEGP_GetLayerObjectType() and check if the result equals AEGP_ObjectType_VECTOR to determine if a layer is a shape layer.

```cpp
AEGP_GetLayerObjectType()
// Returns AEGP_ObjectType_VECTOR for shape layers
```

*Tags: `aegp`, `layer-checkout`, `sdk`, `shape`*

---

### What do the origin_x and origin_y fields in PF_EffectWorld represent when checking out a shape layer?

The origin_x and origin_y fields in the PF_EffectWorld structure represent the offset position of the checked-out shape within its buffer. When checking out a shape layer, the resulting buffer is the size of the shape itself (not the layer), and the origin fields indicate where that shape is positioned relative to the layer's coordinate space.

*Tags: `layer-checkout`, `output-rect`, `sdk`, `shape`*

---
