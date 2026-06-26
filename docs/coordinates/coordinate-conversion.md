# Coordinate Conversion

> 1 Q&A · source: AE plugin dev community Discord

### How do I convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL coordinate system?

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

*Tags: `aegp`, `camera`, `coordinate-conversion`, `matrix`, `opengl`*

---
