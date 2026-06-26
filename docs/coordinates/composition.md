# Composition

> 1 Q&A · source: AE plugin dev community Discord

### How can an AEGP plugin check if there is an active camera in the current composition?

Use the AEGP_GetCameraType() function with a layer handle to determine the camera type. This function returns one of three values: AEGP_CameraType_NONE (value -1) if no camera, AEGP_CameraType_PERSPECTIVE, or AEGP_CameraType_ORTHOGRAPHIC. Alternatively, iterate through the composition's layers and identify the first camera layer that is not out of trim at the current time—this will be the active camera. If no camera layers are found, the composition has no active camera.

```cpp
AEGP_GetCameraType(layerH) returns:
AEGP_CameraType_NONE = -1
AEGP_CameraType_PERSPECTIVE
AEGP_CameraType_ORTHOGRAPHIC
```

*Tags: `aegp`, `camera`, `composition`, `layer-checkout`*

---
