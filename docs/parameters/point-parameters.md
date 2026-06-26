# Point Parameters

> 2 Q&As · source: AE plugin dev community Discord

### Can you disable the crosshair/visual representation of point parameters in the AE comp window?

No, the crosshairs are the visual representation of point parameters and cannot be disabled. As a workaround, you could use a different parameter type like an int or float param pair for x and y values instead of a point parameter.

*Tags: `comp-window`, `crosshair`, `point-parameters`, `ui`, `workaround`*

---

### My Point2D parameter values are off by a factor of ~65536 — coordinates land way outside the frame. What's wrong?

AE stores `PF_ADD_POINT` values as `PF_Fixed` — a 16.16 fixed-point integer where the upper 16 bits are the integer part and the lower 16 bits are the fraction. Reading `val->u.td.x_value` directly as a float gives you the raw integer (e.g., `6553600` instead of `100.0`).

Always apply `FIX_2_FLOAT()` and then scale by the downsample factor:

```cpp
pt.x = FIX_2_FLOAT(val->u.td.x_value) * downsample_x;
pt.y = FIX_2_FLOAT(val->u.td.y_value) * downsample_y;
```

The downsample factor is necessary because AE coordinates are in full-resolution layer space; at reduced preview resolutions the render buffer is smaller, so without scaling the point lands outside the buffer.

The same applies to angle parameters:
```cpp
double angle = FIX_2_FLOAT(val->u.ad.value);  // PF_ADD_ANGLE is also PF_Fixed
```

**Exception**: 3D point parameters (`PF_ADD_POINT_3D`) use `PF_FpLong` (double), not `PF_Fixed`, so no `FIX_2_FLOAT` is needed — only downsample scaling.

*Tags: `point-parameters`, `fixed-point`, `pf-fixed`, `fix2float`, `downsample`, `params`*

---
