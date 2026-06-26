# Q&A: matrix

**14 entries** tagged with `matrix`.

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

## How do you correctly extract and convert After Effects camera and light matrix data for use in a Vulkan shader?

The user describes a 4-step process: (1) get camera matrix from AE, (2) convert it to column-major format, (3) normalize position in clip space using range (-1,1) with zoom value as z factor, (4) get light matrix and normalize it. However, they report that using model and projection matrix operations hasn't produced expected results. The answer suggests this is a known challenge when working with AE camera data in non-OpenGL contexts like Vulkan, and the correct transformation depends on properly handling the coordinate system conversion between AE's internal representation and the target graphics API.

*Tags: `camera`, `matrix`, `vulkan`, `shader`, `coordinates`, `lighting`*

---

## How do you correctly extract and transform camera and light matrices from After Effects to use in a Vulkan shader?

The user is working with After Effects camera/light matrix data for a Vulkan-based plugin. Their approach involves: (1) getting the camera matrix from AE, (2) converting it to column-major format, (3) normalizing position in clip space (-1,1) using zoom as a z factor, and (4) getting and normalizing the light matrix. They note they are not using OpenGL operations like glPerspective or glMatrixMode since they are targeting Vulkan. The user indicates they've tried model and projection matrix operations but results don't match AE's render. This suggests the camera transformation pipeline in AE may require specific handling of coordinate systems, matrix conventions (row vs column major), and potentially AE-specific camera parameters that differ from standard graphics API conventions.

*Tags: `camera`, `vulkan`, `matrix`, `shader`, `render-loop`, `gpu`*

---

## How do you correctly extract and transform camera and light matrix data from After Effects for use in a Vulkan shader?

The process involves several steps: (1) Get the camera matrix from After Effects, (2) Convert it to column-major format, (3) Normalize the position in clip space using the range (-1, 1) and apply the zoom value as a z-factor, (4) Get the light matrix and normalize it similarly. The user was attempting this without using OpenGL operations like glPerspective or glMatrixMode (since they're using Vulkan). They encountered issues where the rendered light position didn't match the AE position even after trying various model and projection matrix operations. The key challenge is ensuring proper coordinate space conversion between After Effects' internal representation and Vulkan's expected format.

*Tags: `vulkan`, `camera`, `matrix`, `shader`, `lighting`, `coordinate-space`*

---

## How do I convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL coordinate system?

After Effects uses a row-major matrix layout (0 1 2 3 / 4 5 6 7 / 8 9 10 11 / 12 13 14 15) while OpenGL uses column-major (0 4 8 12 / 1 5 9 13 / 2 6 10 14 / 3 7 11 15). To convert, transpose the matrix by iterating through rows and columns: for each x from 0 to 3, set glMatrix[x][0] = matrix.mat[0][x], glMatrix[x][1] = matrix.mat[1][x], glMatrix[x][2] = matrix.mat[2][x], glMatrix[x][3] = matrix.mat[3][x]. You may also need matrix operations like inversion, for which the GLM library is recommended.

```cpp
for (x = 0; x < 4; x++)
{
    glMatrix[x][0] = matrix.mat[0][x];
    glMatrix[x][1] = matrix.mat[1][x];
    glMatrix[x][2] = matrix.mat[2][x];
    glMatrix[x][3] = matrix.mat[3][x];
}
```

*Tags: `aegp`, `opengl`, `matrix`, `coordinate-conversion`, `camera`*

---

## What resources explain how to convert After Effects camera matrices to OpenGL format?

Two archived Adobe Community threads provide detailed guidance on matrix coordinate system conversion for AE camera data. The first thread at https://community.adobe.com/t5/after-effects-discussions/glator-for-dummies-and-from-dummy/m-p/6930311 discusses camera matrix conversion generally. The second thread (archived at https://web.archive.org/web/20111223234233/http://forums.adobe.com/thread/570135) provides specific code examples for transposing AE matrices to OpenGL format, explaining the row-major vs column-major difference and matrix operations needed.

*Tags: `aegp`, `opengl`, `camera`, `reference`, `matrix`*

---

## What does AEGP_GetLayerToWorldXform matrix represent and how should it be used?

AEGP_GetLayerToWorldXform translates XY coordinates from the layer origin to XYZ in composition space. It can be used for simple coordinate conversion between layer-space and composition-space, or to construct a full render view matrix from the camera's point of view.

*Tags: `aegp`, `coordinate-transform`, `layer-space`, `composition-space`, `matrix`, `camera`*

---

## How can I get camera position in world coordinates when the camera is parented to another layer?

Use AEGP_GetEffectCameraMatrix() to get the camera matrix post all parenting transformations. You can then extract the camera position from that matrix.

```cpp
ERR(suites.StreamSuite4()->AEGP_GetLayerStreamValue(camera_layerH, AEGP_LayerStream_ANCHORPOINT, AEGP_LTimeMode_CompTime, &comp_timeT, FALSE, &stream_anchor, NULL));
```

*Tags: `aegp`, `camera`, `3d`, `matrix`, `sdk`*

---

## What is a good reference sample project for implementing 3D matrix transformations in After Effects plugins?

The "Artie" sample project included in the After Effects SDK contains macros for getting XY coordinates of a texture using a 4x4 matrix. This sample is useful as a proof of concept for implementing 3D transform functions when antialiasing is not a critical concern.

*Tags: `reference`, `3d`, `matrix`, `sample`, `sdk`, `open-source`*

---

## Can a 3x3 matrix be used to apply perspective transformations with transform_world()?

No. 3x3 matrices can only perform 2D affine transformations: position, scale, Z rotation, and skewing (anything the Transform effect does). 4x4 matrices contain additional parameters for X and Y rotations and Z translations. You cannot directly translate a 4x4 matrix into a 3x3 matrix; instead, you must decompose the 4x4 matrix and re-feed its components into the 3x3. Corner pin style transforms are not possible using transform_world().

*Tags: `transform_world`, `matrix`, `3d`, `affine`, `perspective`*

---

## How do you draw transformed circular handles in macOS After Effects plugins accounting for layer scale, rotation, and position?

Use Quartz context transformation matrices to apply layer-to-frame transformations. First get the layer2frame transformation matrix using get_layer2comp_xform and source_to_frame callbacks. Then create a CGAffineTransform that includes the Y-axis inversion correction, and apply it to the context using CGContextConcatCTM before drawing. The Y-inversion adjustment should negate the b and d components and adjust the ty translation by the flipped origin offset.

```cpp
float a = xform.mat[0][0];
float b = xform.mat[0][1];
float c = xform.mat[1][0];
float d = xform.mat[1][1];
float tx = xform.mat[2][0];
float ty = xform.mat[2][1];
CGPoint origin = CGPointMake(0,0);
CGPoint originFlipped = CGContextConvertPointToUserSpace(context, origin);
CGAffineTransform trans = CGAffineTransformMake (a, -b, c, -d, tx, originFlipped.y - (ty - event_extraP->u.draw.update_rect.top));
CGContextConcatCTM (context, trans);
CGRect rect = CGRectMake ( center.x - cr, center.y - cr, 2*cr, 2*cr);
CGContextAddEllipseInRect(context, rect);
CGContextStrokePath(context);
```

*Tags: `macos`, `ui`, `quartz`, `drawing`, `transformation`, `matrix`*

---

## How do you properly use the transform_world() function to scale and rotate a PF_LayerDef?

To use transform_world() effectively: (1) Pass NULL for mask_world0 to use the whole frame. (2) You can pass one or two matrices—two matrices enable motion blur between transformations. Use syntax (&matrix1, &matrix2) for two matrices or &matrix1 for one, and specify the matrix count. (3) Note that AE's 3x3 matrices have swapped x and y compared to standard math notation—what should be matrix[1][2] goes in matrix[2][1]. See the CCU sample for identity and transformation matrix functions. (4) Always pass a full-frame rectangle initially; transform_world() does not accept NULL for the destination rectangle. The offset should be embedded in the matrix itself, not assumed from the rectangle's top-left corner.

*Tags: `mfr`, `params`, `matrix`, `transform`, `reference`*

---

## Where can I find sample code for creating and manipulating transformation matrices in After Effects plugins?

The CCU sample included in the After Effects SDK contains helper functions for creating identity matrices and concatenating them with scale and rotation transformations. This sample demonstrates proper matrix manipulation for use with transform_world() and similar functions.

*Tags: `mfr`, `reference`, `sdk`, `sample`, `matrix`*

---
