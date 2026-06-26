# Math

> 1 Q&A · source: AE plugin dev community Discord

### When splitting a matrix result with the math utility, what values appear in the scale component?

The scale values can be extracted from a 4x4 matrix using AEGP_MatrixDecompose4, which decomposes the matrix into position, scale, shear, and rotation components. After multiplying the layer-to-world matrix with a point matrix, decomposing the result will give you the scale vector (scaleVP) among other transformation components.

```cpp
A_FloatPoint3 posVP, scaleVP, shearVP, rotVP;
ERR(suites.MathSuite1()->AEGP_MatrixDecompose4(&resultMat4, &posVP, &scaleVP, &shearVP, &rotVP));
```

*Tags: `aegp`, `math`, `mfr`, `params`*

---
