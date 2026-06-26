# Path

> 2 Q&As · source: AE plugin dev community Discord

### How does the AddArc function work when drawing multiple circles in After Effects UI?

The AddArc function treats all drawing operations as a continuous path. When drawing multiple circles, you must call close() after each AddArc to properly close that shape before starting a new one. Without closing, the path continues from the end point of one circle back to the beginning of the next, creating unexpected intersections. Alternatively, you can draw each shape in separate paths with separate drawing calls. The center argument for AddArc should use absolute position coordinates.

```cpp
DRAWBOT_PointF32 center1 = { drawRectF.left + point.x , drawRectF.top + drawRectF.height - point.y };
suites.PathSuiteCurrent()->AddArc(plotPath, &center1, 4.0f, 0.0f, 360.0f);
suites.PathSuiteCurrent()->Close(plotPath);
DRAWBOT_PointF32 center2 = { drawRectF.left + point.x + 100 , drawRectF.top + drawRectF.height - point.y };
suites.PathSuiteCurrent()->AddArc(plotPath, &center2, 4.0f, 0.0f, 360.0f);
suites.PathSuiteCurrent()->Close(plotPath);
```

*Tags: `debugging`, `drawbot`, `path`, `ui`*

---

### How can I duplicate layers along a 3D path with auto-orientation in After Effects?

The After Effects API does not offer built-in tools for rendering 3D transformations. While transform_world() is available for 2D transformations, you need to implement your own 3D transform algorithm to render layer instances along a path. One approach is to calculate the transformation yourself and use transform_world() to fake a 3D look with scale and rotation. For a proof of concept, examine the "Artie" sample project which contains macros for getting XY coordinates of a texture using a 4x4 matrix. If antialiasing is not a concern, this code can provide a quick setup for a 3D transform function.

*Tags: `3d`, `aegp`, `path`, `reference`, `transform`*

---
