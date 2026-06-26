# Affine

> 1 Q&A · source: AE plugin dev community Discord

### Can a 3x3 matrix be used to apply perspective transformations with transform_world()?

No. 3x3 matrices can only perform 2D affine transformations: position, scale, Z rotation, and skewing (anything the Transform effect does). 4x4 matrices contain additional parameters for X and Y rotations and Z translations. You cannot directly translate a 4x4 matrix into a 3x3 matrix; instead, you must decompose the 4x4 matrix and re-feed its components into the 3x3. Corner pin style transforms are not possible using transform_world().

*Tags: `3d`, `affine`, `matrix`, `perspective`, `transform_world`*

---
