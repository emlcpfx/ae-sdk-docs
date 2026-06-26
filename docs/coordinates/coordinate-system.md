# Coordinate System

> 3 Q&As · source: AE plugin dev community Discord

### How do you convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL matrix format?

After Effects uses row-major matrix layout while OpenGL uses column-major. You need to transpose the matrix. For AE matrix with layout [0,1,2,3 / 4,5,6,7 / 8,9,10,11 / 12,13,14,15], convert to OpenGL layout [0,4,8,12 / 1,5,9,13 / 2,6,10,14 / 3,7,11,15] by iterating through columns and assigning: glMatrix[x][0] = matrix.mat[0][x], glMatrix[x][1] = matrix.mat[1][x], glMatrix[x][2] = matrix.mat[2][x], glMatrix[x][3] = matrix.mat[3][x].

```cpp
for (x = 0; x < 4; x++)
{
    glMatrix[x][0] = matrix.mat[0][x];
    glMatrix[x][1] = matrix.mat[1][x];
    glMatrix[x][2] = matrix.mat[2][x];
    glMatrix[x][3] = matrix.mat[3][x];
}
```

*Tags: `aegp`, `coordinate-system`, `matrix`, `opengl`*

---

### How do you extract and use camera and light matrix data from After Effects in a Vulkan shader to match AE's render output?

The user is working on extracting camera matrix from AE and converting it to column-major format, then normalizing position in clip space using zoom as a z factor, followed by getting the light matrix and normalizing it. They mention attempting model and projection matrix operations but getting unexpected results. The key challenge is ensuring the matrix transformations match AE's internal coordinate system when not using OpenGL operations like glPerspective or glMatrixMode, particularly when working with Vulkan instead.

*Tags: `camera`, `coordinate-system`, `lighting`, `matrix-transforms`, `shader`, `vulkan`*

---

### How do you convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL matrix format for camera coordinates?

After Effects uses row-major matrix layout while OpenGL uses column-major layout. You need to transpose the matrix by converting AE matrix positions (0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15) to OpenGL positions (0,4,8,12,1,5,9,13,2,6,10,14,3,7,11,15). This is done by iterating through the 4x4 matrix and assigning glMatrix[x][y] = matrix.mat[y][x] for each element. You may also need to perform matrix inversion and other operations, for which the GLM library is recommended.

```cpp
for (x = 0; x < 4; x++)
{
    glMatrix[x][0] = matrix.mat[0][x];
    glMatrix[x][1] = matrix.mat[1][x];
    glMatrix[x][2] = matrix.mat[2][x];
    glMatrix[x][3] = matrix.mat[3][x];
}
```

*Tags: `aegp`, `camera`, `coordinate-system`, `matrix`, `opengl`*

---
