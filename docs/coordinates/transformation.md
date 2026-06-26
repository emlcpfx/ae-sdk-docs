# Transformation

> 4 Q&As · source: AE plugin dev community Discord

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

### How does the PF_FloatMatrix work and how do you use it with transform_world() to transform a PF_EffectWorld?

PF_FloatMatrix is a standard 3x3 transformation matrix used in graphics programming. It can perform translation, rotation, scaling, and skewing transformations all in one concatenated matrix. The matrix indices control different transformation properties: the first row typically contains x-translation and scaling/rotation components, while the third row contains y-translation and scaling/rotation components. When applying rotations using a matrix, you place sin and cos (and negative values) in specific positions following standard 2D/3D graphics conventions. Important note: After Effects uses row-based matrices, which is the opposite of the OpenGL standard, so you need to swap the axis accordingly. The transform_world() function applies the supplied matrix to transform the input image, and the result is transformed along with the layer. There is no "current transformation" state—the matrix is applied independently.

*Tags: `mfr`, `params`, `reference`, `transformation`*

---

### What is a good resource for understanding transformation matrices for use in After Effects plugins?

Wikipedia's article on Transformation Matrices (https://en.wikipedia.org/wiki/Transformation_matrix) is a good starting point for understanding how 3x3 matrices work in graphics. Be aware that After Effects uses row-based matrices, which differs from the OpenGL standard, so axis values need to be swapped accordingly. Standard transformation matrix tutorials and code samples from general graphics programming resources apply directly to AE plugin development.

*Tags: `open-source`, `reference`, `transformation`*

---

### How do you draw transformed circular handles in macOS After Effects plugins accounting for layer scale, rotation, and position?

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

*Tags: `drawing`, `macos`, `matrix`, `quartz`, `transformation`, `ui`*

---
