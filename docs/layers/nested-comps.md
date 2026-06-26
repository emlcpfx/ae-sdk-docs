# Working with Nested Compositions and Pre-comps

This document covers detecting and traversing nested compositions (pre-comps) from both effect plugins and AEGP plugins, including render order implications, collapse transformations, and continuous rasterization.

> For short Q&A-style information, see also [nested-comp.md](nested-comp.md) and [precomp.md](precomp.md).

---

## Table of Contents

- [What is a Pre-comp Layer?](#what-is-a-pre-comp-layer)
- [Detecting Pre-comp Layers](#detecting-pre-comp-layers)
- [Accessing Nested Comp Contents](#accessing-nested-comp-contents)
- [Render Order Implications](#render-order-implications)
- [Collapse Transformations](#collapse-transformations)
- [Continuous Rasterization](#continuous-rasterization)
- [Effects on Pre-comp Layers](#effects-on-pre-comp-layers)
- [Traversing the Full Comp Hierarchy](#traversing-the-full-comp-hierarchy)
- [Nesting Relationships and Lookups](#nesting-relationships-and-lookups)
- [Precomposing via AEGP](#precomposing-via-aegp)
- [Coordinate Space Considerations](#coordinate-space-considerations)
- [Complete Example: Recursive Comp Traversal](#complete-example-recursive-comp-traversal)
- [Pitfalls and Warnings](#pitfalls-and-warnings)

---

## What is a Pre-comp Layer?

In After Effects, any composition can be used as a layer in another composition. When Comp A is placed as a layer inside Comp B:

- The layer in Comp B is called a **pre-comp layer**
- Comp A is the **source composition** (or "nested comp")
- Comp B is the **parent composition**

From the SDK perspective, a pre-comp layer is an `AEGP_ObjectType_AV` layer whose source item is of type `AEGP_ItemType_COMP`.

```
Comp B (parent)
  |-- Layer 1: Solid
  |-- Layer 2: Pre-comp --> Comp A (nested/source)
  |     |-- Layer A1: Footage
  |     |-- Layer A2: Text
  |-- Layer 3: Camera
```

---

## Detecting Pre-comp Layers

### From AEGP

```cpp
A_Boolean IsPrecompLayer(
    AEGP_LayerSuite9 *ls,
    AEGP_ItemSuite9  *is,
    AEGP_LayerH      layerH)
{
    A_Err err = A_Err_NONE;

    // First check that it's an AV layer
    AEGP_ObjectType obj_type;
    ERR(ls->AEGP_GetLayerObjectType(layerH, &obj_type));
    if (err || obj_type != AEGP_ObjectType_AV) return FALSE;

    // Get the source item
    AEGP_ItemH sourceH;
    ERR(ls->AEGP_GetLayerSourceItem(layerH, &sourceH));
    if (err || !sourceH) return FALSE;

    // Check if source is a composition
    AEGP_ItemType item_type;
    ERR(is->AEGP_GetItemType(sourceH, &item_type));

    return (!err && item_type == AEGP_ItemType_COMP);
}
```

### From an Effect Plugin

Within an effect's render call, you can detect if the layer your effect is applied to is inside a nested comp, but you cannot directly detect if a checked-out layer parameter is a pre-comp. You would need AEGP suites for that:

```cpp
// Bridge from PF to AEGP
AEGP_LayerH my_layerH = NULL;
ERR(pf_interface_suite->AEGP_GetEffectLayer(in_data->effect_ref, &my_layerH));

AEGP_CompH parent_compH;
ERR(layer_suite->AEGP_GetLayerParentComp(my_layerH, &parent_compH));

// Now check if parent_compH is itself nested somewhere...
```

---

## Accessing Nested Comp Contents

Once you have identified a pre-comp layer, you can access the contents of the nested composition:

```cpp
// Get the source item from the pre-comp layer
AEGP_ItemH sourceH;
ERR(layer_suite->AEGP_GetLayerSourceItem(layerH, &sourceH));

// Convert to a comp handle
AEGP_CompH nested_compH;
ERR(comp_suite->AEGP_GetCompFromItem(sourceH, &nested_compH));

// Now you can iterate layers in the nested comp
A_long num_layers;
ERR(layer_suite->AEGP_GetCompNumLayers(nested_compH, &num_layers));

for (A_long i = 0; i < num_layers && !err; i++) {
    AEGP_LayerH nested_layerH;
    ERR(layer_suite->AEGP_GetCompLayerByIndex(nested_compH, i, &nested_layerH));

    // Process nested layer...
    // This could itself be another pre-comp (recursive nesting)
}
```

### Getting Nested Comp Properties

```cpp
// Dimensions of the nested comp
AEGP_ItemH nested_itemH;
ERR(comp_suite->AEGP_GetItemFromComp(nested_compH, &nested_itemH));

A_long nested_w, nested_h;
ERR(item_suite->AEGP_GetItemDimensions(nested_itemH, &nested_w, &nested_h));

// Frame rate of the nested comp (may differ from parent)
A_FpLong nested_fps;
ERR(comp_suite->AEGP_GetCompFramerate(nested_compH, &nested_fps));

// Duration
A_Time nested_dur;
ERR(item_suite->AEGP_GetItemDuration(nested_itemH, &nested_dur));
```

---

## Render Order Implications

### Normal Pre-comp Rendering

Without Collapse Transformations, a pre-comp layer renders in a two-stage process:

1. **Stage 1**: The nested composition is rendered at its own resolution to produce a flat 2D image
2. **Stage 2**: That flat image is placed into the parent composition as a layer, with the layer's transforms (position, scale, rotation, opacity) applied

This means:

- Effects on layers inside the pre-comp are rendered first
- The result is flattened (composited) at the pre-comp's resolution
- Then the flat result is treated as footage in the parent comp

### Implications for Effect Plugins

If your effect is applied to a pre-comp layer (in the parent comp):

- Your effect receives the **flattened, pre-rendered** output of the nested comp
- You see a single flat image, not the individual layers inside
- The resolution of the input matches the nested comp's dimensions (not the parent comp)
- Any 3D layers inside the pre-comp have already been rendered to 2D

If your effect is applied to a layer INSIDE a pre-comp:

- Your effect runs during Stage 1
- You operate on that individual layer's pixels
- Other layers in the same pre-comp may or may not be visible depending on your effect type
- Your effect's output is composited with other layers in the pre-comp before the result is used in the parent comp

---

## Collapse Transformations

The **Collapse Transformations** switch (`AEGP_LayerFlag_COLLAPSE`, value `0x00000080`) fundamentally changes how a pre-comp layer is rendered.

### Without Collapse (Normal)

```
Pre-comp renders at its own resolution --> Flat image --> Transformed in parent comp
```

### With Collapse Enabled

```
Pre-comp layers are rendered directly in the parent comp context
```

When Collapse Transformations is enabled:

1. The nested comp is NOT rendered to a flat intermediate image
2. Instead, the individual layers from the nested comp are composed directly in the parent comp
3. 3D layers inside the pre-comp participate in the parent comp's 3D space
4. Transforms are concatenated rather than applied sequentially
5. Resolution is determined by the parent comp, not the pre-comp

### Impact on Effect Plugins

**Effects on a collapsed pre-comp layer:**

- Your effect still receives a rendered frame, but the rendering context changes
- The layer dimensions may differ from the non-collapsed case
- Buffer origin and coordinate space may be affected
- The rendered content reflects the parent comp's resolution and camera perspective

**Effects inside a collapsed pre-comp:**

- Your effect runs in the context of the parent comp's render pipeline
- `in_data->width` and `in_data->height` may reflect the parent comp dimensions
- Downsample factors come from the parent comp

### Detecting Collapse Transformations

```cpp
AEGP_LayerFlags flags;
ERR(layer_suite->AEGP_GetLayerFlags(layerH, &flags));

if (flags & AEGP_LayerFlag_COLLAPSE) {
    // Collapse Transformations is enabled
    // (or Continuously Rasterize for vector layers)
}
```

### Programmatically Setting Collapse

```cpp
ERR(layer_suite->AEGP_SetLayerFlag(
    layerH,
    AEGP_LayerFlag_COLLAPSE,
    TRUE));     // Enable collapse
```

---

## Continuous Rasterization

The same flag (`AEGP_LayerFlag_COLLAPSE`) serves as **Continuously Rasterize** for vector-based layers (shape layers, text layers, Illustrator files). The behavior differs from pre-comp collapse:

### What Continuously Rasterize Does

- Vector/shape layers are rasterized at the final output resolution rather than at their native size
- This prevents pixelation when scaling up vector content
- The layer is re-rasterized every frame if it or its parent transforms change

### Impact on Effect Plugins

When Continuously Rasterize is enabled on a shape/text layer that your effect is applied to:

- The layer buffer may be larger than expected (rendered at comp resolution, not layer source resolution)
- `in_data->width` and `in_data->height` still reflect the source dimensions
- The actual buffer (`PF_EffectWorld`) dimensions may differ from `in_data` dimensions
- Your effect needs to work with whatever buffer size it receives

### Detecting Which Meaning Applies

```cpp
AEGP_ObjectType type;
ERR(layer_suite->AEGP_GetLayerObjectType(layerH, &type));

AEGP_LayerFlags flags;
ERR(layer_suite->AEGP_GetLayerFlags(layerH, &flags));

if (flags & AEGP_LayerFlag_COLLAPSE) {
    AEGP_ItemH srcH;
    ERR(layer_suite->AEGP_GetLayerSourceItem(layerH, &srcH));

    AEGP_ItemType src_type;
    ERR(item_suite->AEGP_GetItemType(srcH, &src_type));

    if (src_type == AEGP_ItemType_COMP) {
        // "Collapse Transformations" on a pre-comp layer
    } else if (type == AEGP_ObjectType_VECTOR ||
               type == AEGP_ObjectType_TEXT) {
        // "Continuously Rasterize" on a vector/text layer
    } else {
        // "Continuously Rasterize" on an Illustrator or PDF layer
    }
}
```

---

## Effects on Pre-comp Layers

### Effect Processing Order

When an effect is on a pre-comp layer:

1. All layers inside the pre-comp are rendered (with their own effects)
2. The pre-comp output is composited to a flat image
3. Effects on the pre-comp layer (in the parent comp) are applied in order, top to bottom
4. The final result is composited into the parent comp

### Resolution Considerations

The pre-comp is rendered at its own resolution. If the pre-comp is 1920x1080 but the parent comp is 3840x2160, your effect on the pre-comp layer will receive a 1920x1080 input (unless Collapse Transformations is on).

### Time Considerations

Time mapping between parent and nested comp:

- The pre-comp layer can have time remapping enabled (`AEGP_LayerFlag_TIME_REMAPPING`)
- It can be stretched (`AEGP_GetLayerStretch`)
- It can have a different frame rate from the parent comp

When your effect checks out the layer at a given time, AE handles all time translation automatically. The frame you receive corresponds to the correct time in the nested comp.

---

## Traversing the Full Comp Hierarchy

### Finding All Comps That Use a Given Comp

There is no direct API to find where a comp is used. You must scan the project:

```cpp
void FindParentComps(
    AEGP_PluginID       plugin_id,
    AEGP_CompH          target_compH,
    AEGP_ProjSuite6     *ps,
    AEGP_ItemSuite9     *is,
    AEGP_CompSuite12    *cs,
    AEGP_LayerSuite9    *ls,
    std::vector<AEGP_CompH> &parent_comps)
{
    A_Err err = A_Err_NONE;

    // Get target comp's item handle for comparison
    AEGP_ItemH target_itemH;
    ERR(cs->AEGP_GetItemFromComp(target_compH, &target_itemH));

    A_long target_id;
    ERR(is->AEGP_GetItemID(target_itemH, &target_id));

    // Iterate all project items
    AEGP_ProjectH projH;
    ERR(ps->AEGP_GetProjectByIndex(0, &projH));

    AEGP_ItemH itemH;
    ERR(is->AEGP_GetFirstProjItem(projH, &itemH));

    while (itemH && !err) {
        AEGP_ItemType type;
        ERR(is->AEGP_GetItemType(itemH, &type));

        if (type == AEGP_ItemType_COMP) {
            AEGP_CompH compH;
            ERR(cs->AEGP_GetCompFromItem(itemH, &compH));

            A_long num_layers;
            ERR(ls->AEGP_GetCompNumLayers(compH, &num_layers));

            for (A_long i = 0; i < num_layers && !err; i++) {
                AEGP_LayerH layerH;
                ERR(ls->AEGP_GetCompLayerByIndex(compH, i, &layerH));

                AEGP_ItemH srcH;
                ERR(ls->AEGP_GetLayerSourceItem(layerH, &srcH));

                A_long src_id;
                ERR(is->AEGP_GetItemID(srcH, &src_id));

                if (src_id == target_id) {
                    parent_comps.push_back(compH);
                    break;  // Found one reference in this comp, move to next
                }
            }
        }

        ERR(is->AEGP_GetNextProjItem(projH, itemH, &itemH));
    }
}
```

### Building a Full Comp Tree

```cpp
struct CompNode {
    AEGP_CompH compH;
    std::vector<CompNode*> children;  // Nested comps within this comp
};

void BuildTree(CompNode *node, /* suites */) {
    A_Err err = A_Err_NONE;
    A_long num_layers;
    ERR(ls->AEGP_GetCompNumLayers(node->compH, &num_layers));

    for (A_long i = 0; i < num_layers && !err; i++) {
        AEGP_LayerH layerH;
        ERR(ls->AEGP_GetCompLayerByIndex(node->compH, i, &layerH));

        AEGP_ObjectType obj_type;
        ERR(ls->AEGP_GetLayerObjectType(layerH, &obj_type));
        if (obj_type != AEGP_ObjectType_AV) continue;

        AEGP_ItemH srcH;
        ERR(ls->AEGP_GetLayerSourceItem(layerH, &srcH));

        AEGP_ItemType src_type;
        ERR(is->AEGP_GetItemType(srcH, &src_type));

        if (src_type == AEGP_ItemType_COMP) {
            AEGP_CompH childCompH;
            ERR(cs->AEGP_GetCompFromItem(srcH, &childCompH));

            CompNode *child = new CompNode();
            child->compH = childCompH;
            node->children.push_back(child);

            // Recurse (be careful of circular references -- AE prevents them)
            BuildTree(child, /* suites */);
        }
    }
}
```

> **Note**: After Effects prevents circular nesting. You cannot nest Comp A inside Comp B if Comp B is already inside Comp A. Therefore, recursive traversal will always terminate.

---

## Nesting Relationships and Lookups

### Animations Live on the Layer, Not the Source

A critical concept: when Comp A is nested as a layer in Comp B, any animations (position, scale, opacity, effects) applied to that layer belong to the **layer in Comp B**, not to Comp A.

To access:
- **Animations inside the pre-comp**: Query layers within the nested comp directly
- **Animations on the pre-comp layer**: Query the layer in the parent comp

```cpp
// Get position animation of the pre-comp LAYER in the parent comp
AEGP_StreamRefH pos_in_parent;
ERR(stream_suite->AEGP_GetNewLayerStream(
    plugin_id,
    precomp_layerH,             // the layer in compB
    AEGP_LayerStream_POSITION,
    &pos_in_parent));
// This gives you the position of the pre-comp layer in compB

// Get position of a layer INSIDE the pre-comp
AEGP_LayerH inner_layerH;
ERR(layer_suite->AEGP_GetCompLayerByIndex(nested_compH, 0, &inner_layerH));

AEGP_StreamRefH pos_inside;
ERR(stream_suite->AEGP_GetNewLayerStream(
    plugin_id,
    inner_layerH,               // a layer inside compA
    AEGP_LayerStream_POSITION,
    &pos_inside));
// This gives you the position of the inner layer in compA
```

---

## Precomposing via AEGP

You can trigger the precompose command programmatically, but with important restrictions:

```cpp
// Trigger precompose via the command system
// WARNING: Do NOT call this during PF_Cmd_USER_CHANGED_PARAM or any
// effect callback -- it will crash because precomposing invalidates
// the calling effect.

// Instead, defer to an idle hook:
static A_Err IdleHook(
    AEGP_GlobalRefcon   plugin_refconP,
    AEGP_IdleRefcon     refconP,
    A_long              *max_sleepPL)
{
    if (should_precompose) {
        // Execute the precompose command
        AEGP_Command precomp_cmd;
        ERR(command_suite->AEGP_GetUniqueCommand(&precomp_cmd));
        // ... or use the built-in command ID for precompose
        should_precompose = false;
    }
    return A_Err_NONE;
}
```

> **Critical Warning**: Calling precompose from within an effect callback (USER_CHANGED_PARAM, RENDER, etc.) will crash After Effects. The precompose operation invalidates the effect from which you are calling, creating a use-after-free condition. Always defer such operations to an AEGP idle hook.

---

## Coordinate Space Considerations

### Pre-comp vs Parent Comp Coordinates

Without Collapse Transformations:
- Layers inside the pre-comp use the pre-comp's coordinate system
- The pre-comp's output is positioned in the parent comp using the pre-comp layer's transform
- Your effect on a pre-comp layer works in the pre-comp's coordinate space (the buffer you receive is the pre-comp's rendered output)

With Collapse Transformations:
- Coordinates are mapped through to the parent comp
- The effective coordinate space depends on the concatenated transforms
- 3D layers pass through the parent comp's camera perspective

### Getting the Effective Transform

To map coordinates from a nested layer all the way to the root comp:

```cpp
// Get the transform from the inner layer to world (its own comp)
A_Matrix4 inner_xform;
ERR(layer_suite->AEGP_GetLayerToWorldXform(inner_layerH, &t, &inner_xform));

// Get the transform from the pre-comp layer to its parent comp
A_Matrix4 precomp_xform;
ERR(layer_suite->AEGP_GetLayerToWorldXform(precomp_layerH, &t, &precomp_xform));

// The full transform is the product:
// point_in_parent = precomp_xform * inner_xform * point_in_inner_layer
```

> **Note**: This manual matrix multiplication is only an approximation. It does not account for collapse transformations, time remapping, or other special rendering modes. For pixel-accurate results, AE's internal rendering pipeline handles these transformations.

---

## Complete Example: Recursive Comp Traversal

```cpp
A_Err TraverseComp(
    AEGP_PluginID       plugin_id,
    AEGP_CompH          compH,
    A_long              depth,
    AEGP_CompSuite12    *cs,
    AEGP_LayerSuite9    *ls,
    AEGP_ItemSuite9     *is,
    AEGP_StreamSuite6   *ss,
    AEGP_MemorySuite1   *ms)
{
    A_Err err = A_Err_NONE;

    A_long num_layers;
    ERR(ls->AEGP_GetCompNumLayers(compH, &num_layers));

    for (A_long i = 0; i < num_layers && !err; i++) {
        AEGP_LayerH layerH;
        ERR(ls->AEGP_GetCompLayerByIndex(compH, i, &layerH));

        // Get layer type
        AEGP_ObjectType obj_type;
        ERR(ls->AEGP_GetLayerObjectType(layerH, &obj_type));

        // Get layer flags
        AEGP_LayerFlags flags;
        ERR(ls->AEGP_GetLayerFlags(layerH, &flags));

        switch (obj_type) {
            case AEGP_ObjectType_AV: {
                // Check if it's a pre-comp
                AEGP_ItemH srcH;
                ERR(ls->AEGP_GetLayerSourceItem(layerH, &srcH));

                AEGP_ItemType src_type;
                ERR(is->AEGP_GetItemType(srcH, &src_type));

                if (src_type == AEGP_ItemType_COMP) {
                    AEGP_CompH nestedH;
                    ERR(cs->AEGP_GetCompFromItem(srcH, &nestedH));

                    A_Boolean collapsed =
                        (flags & AEGP_LayerFlag_COLLAPSE) != 0;

                    // Recurse into nested comp
                    ERR(TraverseComp(plugin_id, nestedH, depth + 1,
                        cs, ls, is, ss, ms));
                }
                break;
            }

            case AEGP_ObjectType_CAMERA: {
                // Read camera zoom
                AEGP_StreamVal2 zoom;
                A_Time t = {0, 1};
                ERR(ss->AEGP_GetLayerStreamValue(layerH,
                    AEGP_LayerStream_ZOOM,
                    AEGP_LTimeMode_CompTime, &t,
                    FALSE, &zoom, NULL));
                break;
            }

            case AEGP_ObjectType_LIGHT: {
                // Read light intensity
                AEGP_StreamVal2 intensity;
                A_Time t = {0, 1};
                ERR(ss->AEGP_GetLayerStreamValue(layerH,
                    AEGP_LayerStream_INTENSITY,
                    AEGP_LTimeMode_CompTime, &t,
                    FALSE, &intensity, NULL));
                break;
            }

            case AEGP_ObjectType_TEXT:
            case AEGP_ObjectType_VECTOR:
                // Handle text and shape layers
                break;

            default:
                break;
        }
    }

    return err;
}
```

---

## Pitfalls and Warnings

1. **A comp can be nested in multiple parent comps.** The same `AEGP_CompH` (same source comp) can appear as layers in many different compositions. There is no single "parent" -- you must scan the project to find all usages.

2. **Precomposing during effect callbacks crashes AE.** Always defer precompose operations to an idle hook. The precompose operation invalidates the effect instance from which you are calling.

3. **Animations on pre-comp layers vs inside pre-comps are separate.** The position/scale/rotation of the pre-comp layer in the parent comp is distinct from any animations on layers inside the nested comp. Query the appropriate layer for the data you need.

4. **Collapse Transformations changes buffer dimensions.** When enabled, the buffer your effect receives may be at the parent comp's resolution rather than the pre-comp's resolution. Do not assume buffer dimensions will match `in_data->width/height`.

5. **Continuously Rasterize uses the same flag as Collapse Transformations.** `AEGP_LayerFlag_COLLAPSE` (`0x80`) means "Collapse" for pre-comp layers and "Continuously Rasterize" for vector layers. Check the source type to disambiguate.

6. **Circular nesting is impossible.** AE prevents it at the UI level and via `AEGP_IsAddLayerValid`. You do not need to guard against infinite recursion when traversing comp hierarchies.

7. **Frame rates can differ between parent and nested comps.** Time remapping and different frame rates mean that frame N in the parent does not necessarily correspond to frame N in the nested comp. Always use proper time conversion.

8. **`AEGP_GetLayerSourceItem` on cameras and lights does not return a comp.** Only AV layers can be pre-comp layers. Cameras, lights, and null objects do not have source items that are compositions.

9. **No direct API to find nesting relationships.** You must iterate all project items and layers to discover which comps use which other comps. Build your own lookup structure if you need fast queries.

10. **Effect render order inside collapsed pre-comps is different.** When a pre-comp is collapsed, effects inside it may see different buffer sizes and coordinate spaces than when the same comp is rendered standalone. Always use the buffer dimensions from the actual `PF_EffectWorld` rather than assuming they match `in_data->width/height`.
