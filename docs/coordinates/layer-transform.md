# Layer Transform

> 1 Q&A · source: AE plugin dev community Discord

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
