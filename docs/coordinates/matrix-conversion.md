# Matrix Conversion

> 2 Q&As · source: AE plugin dev community Discord

### How do you convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL matrix format?

After Effects uses row-major matrix layout while OpenGL uses column-major layout. You need to transpose the matrix during conversion. For a 4x4 matrix, map AE matrix elements (row-major: 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15) to OpenGL format (column-major: 0,4,8,12,1,5,9,13,2,6,10,14,3,7,11,15). This can be done by iterating through the matrix and reassigning: glMatrix[x][0] = matrix.mat[0][x], glMatrix[x][1] = matrix.mat[1][x], glMatrix[x][2] = matrix.mat[2][x], glMatrix[x][3] = matrix.mat[3][x].

```cpp
for (x = 0; x < 4; x++)
{
    glMatrix[x][0] = matrix.mat[0][x];
    glMatrix[x][1] = matrix.mat[1][x];
    glMatrix[x][2] = matrix.mat[2][x];
    glMatrix[x][3] = matrix.mat[3][x];
}
```

*Tags: `aegp`, `camera`, `matrix-conversion`, `opengl`*

---

### What tools should be used for matrix operations and inversion when working with camera matrices in After Effects plugins?

The GLM (OpenGL Mathematics) library is recommended for performing matrix operations and inversions. It provides robust mathematical functions for handling matrix transformations needed when converting and manipulating camera coordinate systems.

*Tags: `aegp`, `build`, `camera`, `matrix-conversion`, `opengl`*

---
