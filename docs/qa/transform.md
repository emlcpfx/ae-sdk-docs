# Q&A: transform

**7 entries** tagged with `transform`.

---

## What does AEGP_GetLayerToWorldXform actually represent?

AEGP_GetLayerToWorldXform translates XY coordinates from the layer origin to XYZ in composition space. 'World' means composition space. You can use the returned matrix for simple layer-to-comp coordinate conversion, or use it to construct a full 'render view' matrix from the camera's point of view.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-11-01 · Tags: `aegp`, `transform`, `matrix`, `coordinate-space`, `layer-to-comp`*

---

## How do I implement matrix transformation with a custom pivot/anchor point in the AE SDK?

The anchor point goes in the place of position with negative values, but you can't do it all in one matrix. Create three separate matrices: (1) anchor point translation matrix (negative of anchor position), (2) rotation/scale matrix, (3) position translation matrix. Multiply them together: anchor * rotation * position. AE doesn't offer built-in matrix handling functions, but the 'Artie' sample contains code for 4x4 matrix multiplication. For 3x3, use standard nested loop multiplication.

```cpp
for (int i = 0; i < a; i++)
    for (int j = 0; j < d; j++) {
        Mat3[i][j] = 0;
        for (int k = 0; k < c; k++)
            Mat3[i][j] += Mat1[i][k] * Mat2[k][j];
    }
```

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `matrix`, `transform`, `pivot-point`, `anchor`, `rotation`, `scale`*

---

## Where can I find discussions about transform_world and motion blur issues in After Effects?

Rich shared a question posted on the Adobe After Effects community forum discussing transform_world and motion blur. The forum thread can be found at https://community.adobe.com/t5/after-effects-discussions/transform-world-amp-motion-blur/td-p/12918545

*Tags: `reference`, `transform`, `motion-blur`, `debugging`*

---

## How can I read layer transform data like position, anchor, and scale in an After Effects plugin?

To get a layer's numeric transform values (position, anchor, scale), use AEGP_GetNewLayerStream to get the parameter's stream and AEGP_GetNewStreamValue to retrieve its numeric value. However, if you need all of the layer's transformations in the composition including parenting or camera movement effects that affect the layer's position on screen, use AEGP_GetLayerToWorldXform mixed with the camera matrix. Note that you'll need to get AEGP_PluginID during GlobalSetup(), which is required for many AEGP methods.

*Tags: `aegp`, `params`, `layer-checkout`, `transform`*

---

## How can I duplicate layers along a 3D path with auto-orientation in After Effects?

The After Effects API does not offer built-in tools for rendering 3D transformations. While transform_world() is available for 2D transformations, you need to implement your own 3D transform algorithm to render layer instances along a path. One approach is to calculate the transformation yourself and use transform_world() to fake a 3D look with scale and rotation. For a proof of concept, examine the "Artie" sample project which contains macros for getting XY coordinates of a texture using a 4x4 matrix. If antialiasing is not a concern, this code can provide a quick setup for a 3D transform function.

*Tags: `3d`, `transform`, `path`, `aegp`, `reference`*

---

## Why is motion blur incorrect when using PF_TransformWorld with fast rotation?

When using PF_TransformWorld with only two matrices (previous and current), motion blur artifacts can occur during fast rotation, causing particles to appear size-limited. The solution is to use multiple intermediate matrices for in-between values rather than just the start and end positions. Additionally, ensure that the PF_Rect *dest_rect is expanded to encompass the full area of all rotated samples to capture the complete motion blur region.

```cpp
PF_Err transform_world (
  PF_InData *in_data,
  PF_Quality quality,
  PF_ModeFlags m_flags,
  PF_Field field,
  const PF_EffectWorld *src_world,
  const PF_CompositeMode *comp_mode,
  const PF_MaskWorld *mask_world0,
  const PF_FloatMatrix *matrices,
  A_long num_matrices,
  Boolean src2dst_matrix,
  const PF_Rect *dest_rect,
  PF_EffectWorld *dst_world);
```

*Tags: `mfr`, `transform`, `motion-blur`, `rendering`, `output-rect`*

---

## How do you properly use the transform_world() function to scale and rotate a PF_LayerDef?

To use transform_world() effectively: (1) Pass NULL for mask_world0 to use the whole frame. (2) You can pass one or two matrices—two matrices enable motion blur between transformations. Use syntax (&matrix1, &matrix2) for two matrices or &matrix1 for one, and specify the matrix count. (3) Note that AE's 3x3 matrices have swapped x and y compared to standard math notation—what should be matrix[1][2] goes in matrix[2][1]. See the CCU sample for identity and transformation matrix functions. (4) Always pass a full-frame rectangle initially; transform_world() does not accept NULL for the destination rectangle. The offset should be embedded in the matrix itself, not assumed from the rectangle's top-left corner.

*Tags: `mfr`, `params`, `matrix`, `transform`, `reference`*

---
