# Coordinate Space

> 3 Q&As · source: AE plugin dev community Discord

### What does AEGP_GetLayerToWorldXform actually represent?

AEGP_GetLayerToWorldXform translates XY coordinates from the layer origin to XYZ in composition space. 'World' means composition space. You can use the returned matrix for simple layer-to-comp coordinate conversion, or use it to construct a full 'render view' matrix from the camera's point of view.

*Tags: `aegp`, `coordinate-space`, `layer-to-comp`, `matrix`, `transform`*

---

### How do you correctly extract and transform camera and light matrix data from After Effects for use in a Vulkan shader?

The process involves several steps: (1) Get the camera matrix from After Effects, (2) Convert it to column-major format, (3) Normalize the position in clip space using the range (-1, 1) and apply the zoom value as a z-factor, (4) Get the light matrix and normalize it similarly. The user was attempting this without using OpenGL operations like glPerspective or glMatrixMode (since they're using Vulkan). They encountered issues where the rendered light position didn't match the AE position even after trying various model and projection matrix operations. The key challenge is ensuring proper coordinate space conversion between After Effects' internal representation and Vulkan's expected format.

*Tags: `camera`, `coordinate-space`, `lighting`, `matrix`, `shader`, `vulkan`*

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
