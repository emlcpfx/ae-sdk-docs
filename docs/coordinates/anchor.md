# Anchor

> 1 Q&A · source: AE plugin dev community Discord

### How do I implement matrix transformation with a custom pivot/anchor point in the AE SDK?

The anchor point goes in the place of position with negative values, but you can't do it all in one matrix. Create three separate matrices: (1) anchor point translation matrix (negative of anchor position), (2) rotation/scale matrix, (3) position translation matrix. Multiply them together: anchor * rotation * position. AE doesn't offer built-in matrix handling functions, but the 'Artie' sample contains code for 4x4 matrix multiplication. For 3x3, use standard nested loop multiplication.

```cpp
for (int i = 0; i < a; i++)
    for (int j = 0; j < d; j++) {
        Mat3[i][j] = 0;
        for (int k = 0; k < c; k++)
            Mat3[i][j] += Mat1[i][k] * Mat2[k][j];
    }
```

*Tags: `anchor`, `matrix`, `pivot-point`, `rotation`, `scale`, `transform`*

---
