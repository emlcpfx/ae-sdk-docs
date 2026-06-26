# Matrix Math

> 3 Q&As · source: AE plugin dev community Discord

### How do you extract and use camera and light matrix data from After Effects in a Vulkan shader to match AE's rendering?

The process involves: (1) retrieving the camera matrix from AE, (2) converting it to column-major format, (3) normalizing the position in clip space using the range (-1,1) with zoom value as a z factor, and (4) extracting and normalizing the light matrix. The challenge is ensuring proper matrix operations and coordinate space conversions match AE's internal rendering pipeline. Camera data is sent directly from AE to the shader. When working with Vulkan (instead of OpenGL), standard OpenGL operations like glPerspective or glMatrixMode are not available, so manual matrix calculations and transformations must be implemented in the shader code.

*Tags: `camera`, `gpu`, `lighting`, `matrix-math`, `shader`, `vulkan`*

---

### How do you transform layer-space masked bounds to world space using matrix operations in After Effects plugins?

Use AEGP_GetLayerMaskedBounds to get the mask rectangle in layer space, then transform the corner points using the layer-to-world matrix. Create a 4x4 matrix from the 2D point coordinates, multiply it by the layer2WorldMat using AEGP_MultiplyMatrix4, then decompose the result with AEGP_MatrixDecompose4 to extract the final position. The matrix multiplication order matters—multiply the point matrix first, then the layer-to-world transformation.

```cpp
suites.LayerSuite8()->AEGP_GetLayerMaskedBounds(layerH, AEGP_LTimeMode_LayerTime, &curTime, &maskRect);
A_FloatPoint3 topLeft = { maskRect.left, maskRect.top, posVP.z };
A_FloatPoint3 botRight = { maskRect.right, maskRect.bottom, posVP.z };
topLeftP = Multiply2DPointBy4x4Matrix(in_data, out_data, topLeft, myDependencies.layer2WorldMat);
botRightP = Multiply2DPointBy4x4Matrix(in_data, out_data, botRight, myDependencies.layer2WorldMat);
```

*Tags: `aegp`, `coordinate-space`, `layer-transform`, `matrix-math`*

---

### How do you apply transformation matrices with a custom pivot point instead of the upper left corner?

To change the pivot/origin point from the upper left corner to a custom location (like center), you need to use matrix multiplication of three separate matrices: first create a matrix for the anchor point (using negative values of the offset), multiply it by the rotation matrix, then multiply the result by the position matrix. You cannot place all transformations in a single matrix placement.

```cpp
matrix.mat[0][0] = cosf(angle) * scale;
matrix.mat[0][1] = sinf(angle) * scale;
matrix.mat[1][0] = -sinf(angle) * scale;
matrix.mat[1][1] = cosf(angle) * scale;
matrix.mat[2][0] = position_x;
matrix.mat[2][1] = position_y;
matrix.mat[2][2] = 1;
```

*Tags: `matrix-math`, `params`, `transformation`*

---
