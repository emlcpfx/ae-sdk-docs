# Composition Traversal: Layers, Cameras, and Lights

This document covers using AEGP suites to traverse compositions, iterate layers, access cameras and lights, detect 3D layers, query parent-child relationships, and read composition-level properties.

> **Primary Suites**: `AEGP_CompSuite12`, `AEGP_LayerSuite9`, `AEGP_StreamSuite6`
> **Header**: `AE_GeneralPlug.h`

---

## Table of Contents

- [Getting a Composition Handle](#getting-a-composition-handle)
- [Comp Properties](#comp-properties)
- [Iterating Layers](#iterating-layers)
- [Layer Properties and Types](#layer-properties-and-types)
- [Layer Flags](#layer-flags)
- [Camera Layers](#camera-layers)
- [Light Layers](#light-layers)
- [3D Layer Detection](#3d-layer-detection)
- [Parent-Child Relationships](#parent-child-relationships)
- [Layer Timing](#layer-timing)
- [Layer Transforms](#layer-transforms)
- [Creating Layers in a Comp](#creating-layers-in-a-comp)
- [Comp Flags](#comp-flags)
- [Complete Example: Listing All Layers](#complete-example-listing-all-layers)
- [Pitfalls and Warnings](#pitfalls-and-warnings)

---

## Getting a Composition Handle

There are several ways to obtain an `AEGP_CompH`:

### From the Active Comp

```cpp
AEGP_CompH compH = NULL;
ERR(comp_suite->AEGP_GetMostRecentlyUsedComp(&compH));
// compH may be NULL if no comp is open
```

### From a Project Item

```cpp
AEGP_ItemH itemH;
AEGP_ItemType item_type;
ERR(item_suite->AEGP_GetItemType(itemH, &item_type));

if (item_type == AEGP_ItemType_COMP) {
    AEGP_CompH compH;
    ERR(comp_suite->AEGP_GetCompFromItem(itemH, &compH));
}
```

### From a Layer's Parent

```cpp
AEGP_CompH parent_compH;
ERR(layer_suite->AEGP_GetLayerParentComp(layerH, &parent_compH));
```

### From an Effect Being Rendered

Within an effect plugin, use `AEGP_PFInterfaceSuite` to bridge from the PF world to the AEGP world:

```cpp
AEGP_LayerH layerH = NULL;
ERR(pf_interface_suite->AEGP_GetEffectLayer(in_data->effect_ref, &layerH));

AEGP_CompH compH = NULL;
ERR(layer_suite->AEGP_GetLayerParentComp(layerH, &compH));
```

---

## Comp Properties

### Dimensions and Frame Rate

```cpp
// Dimensions (via ItemSuite since comp is also an item)
AEGP_ItemH itemH;
ERR(comp_suite->AEGP_GetItemFromComp(compH, &itemH));

A_long width, height;
ERR(item_suite->AEGP_GetItemDimensions(itemH, &width, &height));

// Frame rate
A_FpLong fps;
ERR(comp_suite->AEGP_GetCompFramerate(compH, &fps));

// Frame duration (1/fps as A_Time)
A_Time frame_dur;
ERR(comp_suite->AEGP_GetCompFrameDuration(compH, &frame_dur));
```

### Duration and Work Area

```cpp
// Total duration (via ItemSuite)
A_Time duration;
ERR(item_suite->AEGP_GetItemDuration(itemH, &duration));

// Work area
A_Time work_start, work_dur;
ERR(comp_suite->AEGP_GetCompWorkAreaStart(compH, &work_start));
ERR(comp_suite->AEGP_GetCompWorkAreaDuration(compH, &work_dur));
```

### Pixel Aspect Ratio

```cpp
A_Ratio par;
ERR(item_suite->AEGP_GetItemPixelAspectRatio(itemH, &par));
```

### Downsample Factor

```cpp
AEGP_DownsampleFactor dsf;
ERR(comp_suite->AEGP_GetCompDownsampleFactor(compH, &dsf));
// dsf.xS, dsf.yS -- downsample factors
```

### Display Start Time

```cpp
A_Time display_start;
ERR(comp_suite->AEGP_GetCompDisplayStartTime(compH, &display_start));
```

### Background Color

```cpp
AEGP_ColorVal bg;
ERR(comp_suite->AEGP_GetCompBGColor(compH, &bg));
// bg.redF, bg.greenF, bg.blueF, bg.alphaF (0.0 - 1.0 range)
```

### Motion Blur Settings

```cpp
A_Ratio angle, phase;
ERR(comp_suite->AEGP_GetCompShutterAnglePhase(compH, &angle, &phase));

A_long mb_samples;
ERR(comp_suite->AEGP_GetCompSuggestedMotionBlurSamples(compH, &mb_samples));

A_long adaptive_limit;
ERR(comp_suite->AEGP_GetCompMotionBlurAdaptiveSampleLimit(compH, &adaptive_limit));
```

---

## Iterating Layers

### Basic Layer Iteration

```cpp
A_long num_layers = 0;
ERR(layer_suite->AEGP_GetCompNumLayers(compH, &num_layers));

for (A_long i = 0; i < num_layers && !err; i++) {
    AEGP_LayerH layerH = NULL;
    ERR(layer_suite->AEGP_GetCompLayerByIndex(compH, i, &layerH));

    if (!err && layerH) {
        // Process layer...
    }
}
```

Layer index 0 is the topmost layer in the timeline (rendered last, on top visually).

### Getting Layer Names

```cpp
AEGP_MemHandle layer_nameH = NULL;
AEGP_MemHandle source_nameH = NULL;

ERR(layer_suite->AEGP_GetLayerName(
    plugin_id,
    layerH,
    &layer_nameH,       // User-assigned layer name (NULL if not renamed)
    &source_nameH));     // Source item name

// Lock and read the name
if (layer_nameH) {
    A_UTF16Char *nameP = NULL;
    ERR(mem_suite->AEGP_LockMemHandle(layer_nameH, (void**)&nameP));
    // Use nameP (null-terminated UTF-16)...
    ERR(mem_suite->AEGP_UnlockMemHandle(layer_nameH));
    ERR(mem_suite->AEGP_FreeMemHandle(layer_nameH));
}
if (source_nameH) {
    ERR(mem_suite->AEGP_FreeMemHandle(source_nameH));
}
```

### Getting the Active Layer

```cpp
AEGP_LayerH active_layerH = NULL;
ERR(layer_suite->AEGP_GetActiveLayer(&active_layerH));
// Returns non-null ONLY if exactly one layer is selected
```

---

## Layer Properties and Types

### Object Type

Every layer has an object type that determines what streams are available:

```cpp
AEGP_ObjectType obj_type;
ERR(layer_suite->AEGP_GetLayerObjectType(layerH, &obj_type));
```

| Object Type | Description |
|---|---|
| `AEGP_ObjectType_AV` | Audio/video layer (footage, solid, shape, pre-comp, adjustment layer) |
| `AEGP_ObjectType_LIGHT` | Light layer |
| `AEGP_ObjectType_CAMERA` | Camera layer |
| `AEGP_ObjectType_TEXT` | Text layer |
| `AEGP_ObjectType_VECTOR` | Shape/vector layer |
| `AEGP_ObjectType_3D_MODEL` | 3D model layer (newer AE versions) |
| `AEGP_ObjectType_NONE` | Invalid or unknown |

### Layer Source Item

Every AV layer has a source item (footage, solid, or nested comp):

```cpp
AEGP_ItemH source_itemH;
ERR(layer_suite->AEGP_GetLayerSourceItem(layerH, &source_itemH));

AEGP_ItemType source_type;
ERR(item_suite->AEGP_GetItemType(source_itemH, &source_type));
// AEGP_ItemType_FOOTAGE for footage/solids
// AEGP_ItemType_COMP for pre-comps
```

### Layer ID

Each layer has a unique, persistent ID within its composition:

```cpp
AEGP_LayerIDVal layer_id;
ERR(layer_suite->AEGP_GetLayerID(layerH, &layer_id));

// Later, retrieve a layer by ID:
AEGP_LayerH foundH;
ERR(layer_suite->AEGP_GetLayerFromLayerID(compH, layer_id, &foundH));
```

---

## Layer Flags

Layer flags represent the toggle switches visible in the timeline:

```cpp
AEGP_LayerFlags flags;
ERR(layer_suite->AEGP_GetLayerFlags(layerH, &flags));
```

| Flag | Value | Meaning |
|---|---|---|
| `AEGP_LayerFlag_VIDEO_ACTIVE` | `0x00000001` | Video eyeball is on |
| `AEGP_LayerFlag_AUDIO_ACTIVE` | `0x00000002` | Audio is enabled |
| `AEGP_LayerFlag_EFFECTS_ACTIVE` | `0x00000004` | Effects are enabled (fx switch) |
| `AEGP_LayerFlag_MOTION_BLUR` | `0x00000008` | Motion blur enabled for this layer |
| `AEGP_LayerFlag_FRAME_BLENDING` | `0x00000010` | Frame blending enabled |
| `AEGP_LayerFlag_LOCKED` | `0x00000020` | Layer is locked |
| `AEGP_LayerFlag_SHY` | `0x00000040` | Layer is shy |
| `AEGP_LayerFlag_COLLAPSE` | `0x00000080` | Collapse Transformations / Continuously Rasterize |
| `AEGP_LayerFlag_AUTO_ORIENT_ROTATION` | `0x00000100` | Auto-orient along motion path |
| `AEGP_LayerFlag_ADJUSTMENT_LAYER` | `0x00000200` | Layer is an adjustment layer |
| `AEGP_LayerFlag_TIME_REMAPPING` | `0x00000400` | Time remapping enabled |
| `AEGP_LayerFlag_LAYER_IS_3D` | `0x00000800` | Layer is 3D |
| `AEGP_LayerFlag_LOOK_AT_CAMERA` | `0x00001000` | Auto-orient towards camera |
| `AEGP_LayerFlag_LOOK_AT_POI` | `0x00002000` | Auto-orient towards point of interest |
| `AEGP_LayerFlag_SOLO` | `0x00004000` | Layer is soloed |
| `AEGP_LayerFlag_MARKERS_LOCKED` | `0x00008000` | Markers are locked |
| `AEGP_LayerFlag_NULL_LAYER` | `0x00010000` | Layer is a null object |
| `AEGP_LayerFlag_HIDE_LOCKED_MASKS` | `0x00020000` | Hide locked masks |
| `AEGP_LayerFlag_GUIDE_LAYER` | `0x00040000` | Layer is a guide layer |
| `AEGP_LayerFlag_ADVANCED_FRAME_BLENDING` | `0x00080000` | Pixel motion frame blending |
| `AEGP_LayerFlag_SUBLAYERS_RENDER_SEPARATELY` | `0x00100000` | Sublayers render separately |
| `AEGP_LayerFlag_ENVIRONMENT_LAYER` | `0x00200000` | Environment layer (for 3D) |

### Setting Flags

```cpp
// Enable 3D on a layer
ERR(layer_suite->AEGP_SetLayerFlag(
    layerH,
    AEGP_LayerFlag_LAYER_IS_3D,
    TRUE));
```

### Checking Real Visibility

The flag alone does not account for solo state. Use:

```cpp
A_Boolean is_really_on;
ERR(layer_suite->AEGP_IsLayerVideoReallyOn(layerH, &is_really_on));
// Accounts for solo status of other layers
```

---

## Camera Layers

Camera layers have `AEGP_ObjectType_CAMERA` and expose camera-specific streams.

### Detecting Cameras

```cpp
AEGP_ObjectType type;
ERR(layer_suite->AEGP_GetLayerObjectType(layerH, &type));

if (type == AEGP_ObjectType_CAMERA) {
    // This is a camera layer
}
```

### Camera Streams

| Stream Enum | Property |
|---|---|
| `AEGP_LayerStream_ZOOM` | Camera zoom (focal length proxy) |
| `AEGP_LayerStream_DEPTH_OF_FIELD` | DOF on/off |
| `AEGP_LayerStream_FOCUS_DISTANCE` | Focus distance |
| `AEGP_LayerStream_APERTURE` | Aperture |
| `AEGP_LayerStream_BLUR_LEVEL` | Blur level percentage |
| `AEGP_LayerStream_IRIS_SHAPE` | Iris shape sides |
| `AEGP_LayerStream_IRIS_ROTATION` | Iris rotation angle |
| `AEGP_LayerStream_IRIS_ROUNDNESS` | Iris roundness |
| `AEGP_LayerStream_IRIS_ASPECT_RATIO` | Iris aspect ratio |
| `AEGP_LayerStream_IRIS_DIFFRACTION_FRINGE` | Diffraction fringe |
| `AEGP_LayerStream_IRIS_HIGHLIGHT_GAIN` | Highlight gain |
| `AEGP_LayerStream_IRIS_HIGHLIGHT_THRESHOLD` | Highlight threshold |
| `AEGP_LayerStream_IRIS_HIGHLIGHT_SATURATION` | Highlight saturation |

### Reading Camera Properties

```cpp
// Read zoom at a specific time
AEGP_StreamVal2 zoom_val;
AEGP_StreamType stream_type;
A_Time t = {0, 1};

ERR(stream_suite->AEGP_GetLayerStreamValue(
    layerH,
    AEGP_LayerStream_ZOOM,
    AEGP_LTimeMode_CompTime,
    &t,
    FALSE,              // post-expression
    &zoom_val,
    &stream_type));

A_FpLong zoom = zoom_val.one_d;
```

### Camera Type

Camera type is not available as a stream. It is accessed through `AEGP_CameraType`:

```cpp
enum {
    AEGP_CameraType_NONE = -1,
    AEGP_CameraType_PERSPECTIVE,
    AEGP_CameraType_ORTHOGRAPHIC,
};
```

### Calculating Field of View from Zoom

The zoom value in AE relates to the field of view by:

```
FOV = 2 * atan(comp_height / (2 * zoom))
```

Where `comp_height` is in pixels and `zoom` is the value from `AEGP_LayerStream_ZOOM`.

---

## Light Layers

### Light Types

```cpp
enum {
    AEGP_LightType_NONE = -1,
    AEGP_LightType_PARALLEL,    // Directional light
    AEGP_LightType_SPOT,        // Spot light
    AEGP_LightType_POINT,       // Point (omni) light
    AEGP_LightType_AMBIENT,     // Ambient light
    AEGP_LightType_ENVIRONMENT, // Environment light
};
```

### Light Streams

| Stream Enum | Property | Applicable Types |
|---|---|---|
| `AEGP_LayerStream_INTENSITY` | Light intensity | All |
| `AEGP_LayerStream_COLOR` | Light color | All |
| `AEGP_LayerStream_CONE_ANGLE` | Cone angle | Spot |
| `AEGP_LayerStream_CONE_FEATHER` | Cone feather | Spot |
| `AEGP_LayerStream_SHADOW_DARKNESS` | Shadow darkness | Parallel, Spot, Point |
| `AEGP_LayerStream_SHADOW_DIFFUSION` | Shadow diffusion | Parallel, Spot, Point |
| `AEGP_LayerStream_SHADOW_COLOR` | Shadow color | Parallel, Spot, Point |
| `AEGP_LayerStream_LIGHT_FALLOFF_TYPE` | Falloff type | Spot, Point |
| `AEGP_LayerStream_LIGHT_FALLOFF_START` | Falloff start distance | Spot, Point |
| `AEGP_LayerStream_LIGHT_FALLOFF_DISTANCE` | Falloff distance | Spot, Point |
| `AEGP_LayerStream_LIGHT_BACKGROUND_VISIBLE` | BG visible | Spot |
| `AEGP_LayerStream_LIGHT_BACKGROUND_OPACITY` | BG opacity | Spot |
| `AEGP_LayerStream_LIGHT_BACKGROUND_BLUR` | BG blur | Spot |

### Light Falloff Types

```cpp
enum {
    AEGP_LightFalloff_NONE = 0,
    AEGP_LightFalloff_SMOOTH,
    AEGP_LightFalloff_INVERSE_SQUARE_CLAMPED
};
```

### Checking Stream Legality

Not all streams are valid for all layer types. Always check before accessing:

```cpp
A_Boolean is_legal = FALSE;
ERR(stream_suite->AEGP_IsStreamLegal(layerH, AEGP_LayerStream_CONE_ANGLE, &is_legal));

if (is_legal) {
    // Safe to access this stream
}
```

---

## 3D Layer Detection

### Via Layer Flags

```cpp
AEGP_LayerFlags flags;
ERR(layer_suite->AEGP_GetLayerFlags(layerH, &flags));
A_Boolean is_3d = (flags & AEGP_LayerFlag_LAYER_IS_3D) != 0;
```

### Via Dedicated Functions

```cpp
A_Boolean is_3d, is_2d;
ERR(layer_suite->AEGP_IsLayer3D(layerH, &is_3d));
ERR(layer_suite->AEGP_IsLayer2D(layerH, &is_2d));
```

> **Note**: Cameras and lights are always 3D. `AEGP_IsLayer3D` returns TRUE for them regardless of the 3D layer flag. `AEGP_IsLayer2D` returns FALSE for cameras and lights.

### 3D Material Properties

3D AV layers expose material property streams:

| Stream | Property |
|---|---|
| `AEGP_LayerStream_ACCEPTS_SHADOWS` | Does this layer receive shadows |
| `AEGP_LayerStream_ACCEPTS_LIGHTS` | Does this layer accept lighting |
| `AEGP_LayerStream_AMBIENT_COEFF` | Ambient material coefficient |
| `AEGP_LayerStream_DIFFUSE_COEFF` | Diffuse material coefficient |
| `AEGP_LayerStream_SPECULAR_INTENSITY` | Specular intensity |
| `AEGP_LayerStream_SPECULAR_SHININESS` | Specular shininess |
| `AEGP_LayerStream_METAL` | Metal material property |
| `AEGP_LayerStream_LIGHT_TRANSMISSION` | Light transmission amount |
| `AEGP_LayerStream_REFLECTION_INTENSITY` | Reflection intensity |
| `AEGP_LayerStream_REFLECTION_SHARPNESS` | Reflection sharpness |
| `AEGP_LayerStream_REFLECTION_ROLLOFF` | Reflection rolloff |
| `AEGP_LayerStream_TRANSPARENCY_COEFF` | Transparency coefficient |
| `AEGP_LayerStream_TRANSPARENCY_ROLLOFF` | Transparency rolloff |
| `AEGP_LayerStream_INDEX_OF_REFRACTION` | Index of refraction |

These streams are only legal when the layer is 3D (`AEGP_LayerFlag_LAYER_IS_3D`).

---

## Parent-Child Relationships

### Get Parent Layer

```cpp
AEGP_LayerH parent_layerH = NULL;
ERR(layer_suite->AEGP_GetLayerParent(layerH, &parent_layerH));
// parent_layerH is NULL if the layer has no parent
```

### Set Parent Layer

```cpp
// Parent layerH to parent_layerH
ERR(layer_suite->AEGP_SetLayerParent(layerH, parent_layerH));

// Remove parent (unparent)
ERR(layer_suite->AEGP_SetLayerParent(layerH, NULL));
```

### Finding All Children of a Layer

There is no direct API to get children. You must iterate all layers and check:

```cpp
void FindChildren(
    AEGP_CompH compH, AEGP_LayerH parentH,
    AEGP_LayerSuite9 *ls, AEGP_LayerIDVal parent_id,
    /* output */ std::vector<AEGP_LayerH> &children)
{
    A_Err err = A_Err_NONE;
    A_long num_layers;
    ERR(ls->AEGP_GetCompNumLayers(compH, &num_layers));

    for (A_long i = 0; i < num_layers && !err; i++) {
        AEGP_LayerH layerH;
        ERR(ls->AEGP_GetCompLayerByIndex(compH, i, &layerH));

        AEGP_LayerH layer_parentH = NULL;
        ERR(ls->AEGP_GetLayerParent(layerH, &layer_parentH));

        if (layer_parentH) {
            AEGP_LayerIDVal pid;
            ERR(ls->AEGP_GetLayerID(layer_parentH, &pid));
            if (pid == parent_id) {
                children.push_back(layerH);
            }
        }
    }
}
```

---

## Layer Timing

### In Point and Duration

```cpp
A_Time in_point, duration;

ERR(layer_suite->AEGP_GetLayerInPoint(layerH,
    AEGP_LTimeMode_CompTime, &in_point));

ERR(layer_suite->AEGP_GetLayerDuration(layerH,
    AEGP_LTimeMode_CompTime, &duration));
```

### Layer Offset and Stretch

```cpp
A_Time offset;
ERR(layer_suite->AEGP_GetLayerOffset(layerH, &offset));
// Offset is always in comp time

A_Ratio stretch;
ERR(layer_suite->AEGP_GetLayerStretch(layerH, &stretch));
```

### Time Conversion

Convert between comp time and layer time:

```cpp
A_Time comp_time = {30, 30};  // 1 second
A_Time layer_time;

ERR(layer_suite->AEGP_ConvertCompToLayerTime(layerH, &comp_time, &layer_time));
ERR(layer_suite->AEGP_ConvertLayerToCompTime(layerH, &layer_time, &comp_time));
```

### Check if Layer is Active at a Time

```cpp
A_Boolean is_active;
A_Time check_time = {60, 30};  // 2 seconds

ERR(layer_suite->AEGP_IsVideoActive(layerH,
    AEGP_LTimeMode_CompTime, &check_time, &is_active));
```

---

## Layer Transforms

### Layer-to-World Transform

Get the full transformation matrix from layer space to world (comp) space at a given time:

```cpp
A_Matrix4 xform;
A_Time t = {0, 1};

ERR(layer_suite->AEGP_GetLayerToWorldXform(layerH, &t, &xform));
// xform is a 4x4 matrix that transforms from layer coords to comp coords
```

For transforms that account for the current view (useful during rendering):

```cpp
ERR(layer_suite->AEGP_GetLayerToWorldXformFromView(
    layerH,
    &view_time,     // view time
    &comp_time,     // comp time
    &xform));
```

### Reading Transform Streams

Individual transform properties are available as streams:

```cpp
// Position
AEGP_StreamRefH posH;
ERR(stream_suite->AEGP_GetNewLayerStream(plugin_id, layerH,
    AEGP_LayerStream_POSITION, &posH));

// Scale
AEGP_StreamRefH scaleH;
ERR(stream_suite->AEGP_GetNewLayerStream(plugin_id, layerH,
    AEGP_LayerStream_SCALE, &scaleH));

// Rotation (Z, or single rotation for 2D)
AEGP_StreamRefH rotH;
ERR(stream_suite->AEGP_GetNewLayerStream(plugin_id, layerH,
    AEGP_LayerStream_ROTATION, &rotH));

// For 3D layers, also available:
// AEGP_LayerStream_ROTATE_X
// AEGP_LayerStream_ROTATE_Y
// AEGP_LayerStream_ORIENTATION (quaternion-like 3D rotation)

// Anchor point
AEGP_StreamRefH anchorH;
ERR(stream_suite->AEGP_GetNewLayerStream(plugin_id, layerH,
    AEGP_LayerStream_ANCHORPOINT, &anchorH));

// Don't forget to dispose all streams when done
```

---

## Creating Layers in a Comp

### Solids

```cpp
AEGP_ColorVal color = {1.0, 1.0, 0.0, 0.0};  // Red
A_Time dur = {300, 30};  // 10 seconds
AEGP_LayerH new_solidH;

ERR(comp_suite->AEGP_CreateSolidInComp(
    u"My Solid",
    1920, 1080,
    &color,
    compH,
    &dur,
    &new_solidH));
```

### Cameras

```cpp
A_FloatPoint center = {960.0f, 540.0f};
AEGP_LayerH new_cameraH;

ERR(comp_suite->AEGP_CreateCameraInComp(
    u"Main Camera",
    center,
    compH,
    &new_cameraH));
```

### Lights

```cpp
A_FloatPoint center = {960.0f, 540.0f};
AEGP_LayerH new_lightH;

ERR(comp_suite->AEGP_CreateLightInComp(
    u"Key Light",
    center,
    compH,
    &new_lightH));
```

### Null Objects

```cpp
A_Time dur = {300, 30};
AEGP_LayerH new_nullH;

ERR(comp_suite->AEGP_CreateNullInComp(
    u"Controller",
    compH,
    &dur,
    &new_nullH));
```

### Text Layers

```cpp
AEGP_LayerH new_textH;

ERR(comp_suite->AEGP_CreateTextLayerInComp(
    compH,
    TRUE,       // select the new layer
    TRUE,       // horizontal text
    &new_textH));
```

### Adding Existing Items as Layers

```cpp
// Check if the item can be added first
A_Boolean valid;
ERR(layer_suite->AEGP_IsAddLayerValid(footageItemH, compH, &valid));

if (valid) {
    AEGP_LayerH new_layerH;
    ERR(layer_suite->AEGP_AddLayer(footageItemH, compH, &new_layerH));
}
```

---

## Comp Flags

```cpp
AEGP_CompFlags comp_flags;
ERR(comp_suite->AEGP_GetCompFlags(compH, &comp_flags));
```

| Flag | Value | Meaning |
|---|---|---|
| `AEGP_CompFlag_SHOW_ALL_SHY` | `0x1` | Show shy layers |
| `AEGP_CompFlag_ENABLE_MOTION_BLUR` | `0x8` | Motion blur enabled for comp |
| `AEGP_CompFlag_ENABLE_TIME_FILTER` | `0x10` | Time filtering enabled |
| `AEGP_CompFlag_GRID_TO_FRAMES` | `0x20` | Snap to frames |
| `AEGP_CompFlag_GRID_TO_FIELDS` | `0x40` | Snap to fields |
| `AEGP_CompFlag_USE_LOCAL_DSF` | `0x80` | Use comp's own downsample factor |
| `AEGP_CompFlag_DRAFT_3D` | `0x100` | Draft 3D mode |
| `AEGP_CompFlag_SHOW_GRAPH` | `0x200` | Show graph editor |

---

## Complete Example: Listing All Layers

This example iterates all layers in the active comp and prints their type, name, and key properties:

```cpp
A_Err ListCompLayers(
    AEGP_PluginID       plugin_id,
    AEGP_CompSuite12    *cs,
    AEGP_LayerSuite9    *ls,
    AEGP_StreamSuite6   *ss,
    AEGP_ItemSuite9     *is,
    AEGP_MemorySuite1   *ms)
{
    A_Err err = A_Err_NONE;

    AEGP_CompH compH = NULL;
    ERR(cs->AEGP_GetMostRecentlyUsedComp(&compH));
    if (!compH) return err;

    // Comp info
    AEGP_ItemH compItemH;
    ERR(cs->AEGP_GetItemFromComp(compH, &compItemH));

    A_long comp_w, comp_h;
    ERR(is->AEGP_GetItemDimensions(compItemH, &comp_w, &comp_h));

    A_FpLong fps;
    ERR(cs->AEGP_GetCompFramerate(compH, &fps));

    // Iterate layers
    A_long num_layers;
    ERR(ls->AEGP_GetCompNumLayers(compH, &num_layers));

    for (A_long i = 0; i < num_layers && !err; i++) {
        AEGP_LayerH layerH;
        ERR(ls->AEGP_GetCompLayerByIndex(compH, i, &layerH));

        // Object type
        AEGP_ObjectType obj_type;
        ERR(ls->AEGP_GetLayerObjectType(layerH, &obj_type));

        // Flags
        AEGP_LayerFlags flags;
        ERR(ls->AEGP_GetLayerFlags(layerH, &flags));

        A_Boolean is_3d;
        ERR(ls->AEGP_IsLayer3D(layerH, &is_3d));

        // Parent
        AEGP_LayerH parentH = NULL;
        ERR(ls->AEGP_GetLayerParent(layerH, &parentH));

        // Timing
        A_Time in_pt, dur;
        ERR(ls->AEGP_GetLayerInPoint(layerH, AEGP_LTimeMode_CompTime, &in_pt));
        ERR(ls->AEGP_GetLayerDuration(layerH, AEGP_LTimeMode_CompTime, &dur));

        // Source item (for AV layers)
        if (obj_type == AEGP_ObjectType_AV) {
            AEGP_ItemH srcH;
            ERR(ls->AEGP_GetLayerSourceItem(layerH, &srcH));

            AEGP_ItemType src_type;
            ERR(is->AEGP_GetItemType(srcH, &src_type));

            if (src_type == AEGP_ItemType_COMP) {
                // This is a pre-comp layer -- see nested-comps.md
            }
        }
    }

    return err;
}
```

---

## Pitfalls and Warnings

1. **Layer handles are not persistent.** `AEGP_LayerH` values are valid only during the current call. Use `AEGP_GetLayerID` / `AEGP_GetLayerFromLayerID` for persistent references.

2. **Index 0 is the topmost layer.** This is the opposite of what some developers expect. Layer 0 is rendered last (on top), and the highest index is the bottom layer.

3. **Camera and light streams are only legal on their respective object types.** Attempting to access `AEGP_LayerStream_ZOOM` on a non-camera layer will fail. Always check `AEGP_IsStreamLegal` or verify the object type first.

4. **`AEGP_GetActiveLayer` returns NULL if multiple layers are selected.** It only returns a layer handle when exactly one layer is selected.

5. **Comp handles obtained from layers do not need disposal.** `AEGP_CompH` values returned by `AEGP_GetLayerParentComp` are owned by AE, not by your plugin.

6. **Stream references MUST be disposed.** Every `AEGP_GetNewLayerStream`, `AEGP_GetNewEffectStreamByIndex`, etc. allocates a stream that must be freed with `AEGP_DisposeStream`.

7. **Time values use rational arithmetic.** Always use `A_Time` with proper numerator/denominator pairs. Do not convert to floating point for intermediate calculations if you need frame-accurate timing.

8. **`AEGP_LayerFlag_COLLAPSE` serves dual purpose.** For pre-comp layers it means "Collapse Transformations." For vector/shape layers it means "Continuously Rasterize." The flag is the same (`0x00000080`).

9. **Layer creation functions are undoable.** They modify the project. Wrap them in undo groups for a clean undo experience.

10. **Transfer mode is separate from flags.** Blend mode is accessed via `AEGP_GetLayerTransferMode` / `AEGP_SetLayerTransferMode`, not through the flags system.
