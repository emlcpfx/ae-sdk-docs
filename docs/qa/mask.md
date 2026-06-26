# Q&A: mask

**7 entries** tagged with `mask`.

---

## Can AEGP_MaskOutlineSuite be used to procedurally modify mask path vertices each frame as a regular effect?

AEGP_MaskOutlineSuite can modify mask outlines, but the distinction is between destructive/one-time modifications (like running a script that permanently changes path vertices) versus procedural, non-destructive modifications applied each frame before rasterization. A regular effect that procedurally modifies path vertices each frame would be the latter approach, allowing the modifications to be recalculated and applied dynamically during rendering rather than permanently altering the mask data.

*Tags: `aegp`, `mask`, `render-loop`, `params`*

---

## Why does AEGP_GetLayerNumMasks return 0 when a mask is clearly set on the layer in After Effects?

The issue may stem from confusion between track mattes and masks. Track mattes are different from masks—a track matte uses the alpha or luma of one layer to affect another layer, while a mask is a vector shape drawn on a layer with the pen tool. If you're trying to use the Mask Suite on a layer that only has a track matte applied, AEGP_GetLayerNumMasks will correctly return 0 because there are no actual masks on that layer. Additionally, verify that the layer handle is valid and check the error value returned by AEGP_GetLayerNumMasks to diagnose the root cause.

```cpp
A_long numMasks = 0;
RECORD_ERROR(suites.MaskSuite6()->AEGP_GetLayerNumMasks(layerHandle, &numMasks));
for (int i = 0; i < numMasks; i++) {
  AEGP_MaskRefH maskHandle;
  RECORD_ERROR(suites.MaskSuite6()->AEGP_GetLayerMaskByIndex(layerHandle, i, &maskHandle));
  auto mask = ExportMask(context, maskHandle);
  masks.push_back(mask);
  RECORD_ERROR(suites.MaskSuite6()->AEGP_DisposeMask(maskHandle));
}
```

*Tags: `aegp`, `mask`, `debugging`, `macos`*

---

## How can I access mask position and tangent information at a particular parameter for a stroke effect plugin?

Use the PF_PathEvalSegLengthDeriv1 function from the After Effects SDK. This function allows you to evaluate path segments and obtain derivative information, which provides the position and tangent data needed for stamping patterns along mask paths without having to write your own Bezier curve rasterizer.

*Tags: `aegp`, `mask`, `params`, `reference`*

---

## How can you update a layer's mask using the After Effects SDK?

Use the AEGP_MaskSuite and AEGP_MaskOutlineSuite to update layer masks. Note that these suites require copying mask vertices individually. For rare operations or simpler mask copying between layers, consider using AEGP_ExecuteScript() with JavaScript, which may offer a more straightforward solution.

*Tags: `aegp`, `mask`, `sdk`, `layer-checkout`*

---

## What is the simplest way to copy a full mask from one layer to another in After Effects SDK?

While AEGP_MaskSuite and AEGP_MaskOutlineSuite allow mask manipulation, they require vertex-by-vertex copying. For a simpler approach to full mask copying, use AEGP_ExecuteScript() to execute JavaScript code, which typically offers more straightforward mask operations than the C SDK.

*Tags: `aegp`, `mask`, `scripting`, `sdk`*

---

## How do I define a mask parameter in an effect plugin?

Use the PathMaster sample project as a reference, which demonstrates how to add a path parameter to your effect. Note that mask selector parameters are limited to masks on the same layer as the effect only. There is no built-in API for a mask selector on a different layer. To access masks on other layers, use the AEGP suites, but be aware that the API function to render a mask will only work on the local effect layer with mask selector parameters. For cross-layer mask selection, you must either implement your own path rasterizing function or use workarounds such as invisible mask selectors pointing to temporary masks.

*Tags: `mask`, `params`, `aegp`, `ui`, `reference`*

---

## How do I rasterize a mask and use it to restrict pixel operations in my effect plugin?

You can rasterize a mask by filling a dummy buffer with color and applying the mask using the MaskWorldWithPath function from the mask suite. However, this requires the mask mode to be set to something other than NONE. A workaround is to use a temporary buffer for rasterization without affecting the project state. Note that calling AEGP_SetMaskMode is undoable and will prompt users to save changes. For querying whether pixels are contained in a mask, rasterize the mask extent into a temporary buffer and query it directly.

*Tags: `mask`, `aegp`, `memory`, `render-loop`*

---
