# Cinema4D Alembic Camera Workaround

> **Note:** The coordinate system advice here is context-specific. C4D uses left-handed Y-up; verify coordinate conversion needs for your specific pipeline.

## Problem

Cinema4D exports cameras via the Alembic Generator tag as `AbcGeom_Xform_v3` (transform nodes) instead of proper `AbcGeom_Camera_v1` objects. This means the standard Alembic camera detection (`ICamera::matches()`) fails to find the camera.

### Symptoms
- Camera not detected in Alembic file
- Debug output shows: `[CAMERA DEBUG] Falling back to manual view matrix creation`
- Wireframe renders from wrong viewpoint (hardcoded default camera position)

### Root Cause
When using Cinema4D's Alembic Generator tag on a camera, it exports the camera's transform data as an Xform node with a child "Shape" node, rather than as a proper ICamera object with lens parameters.

Example hierarchy from C4D export:
```
ABC/
  BDY_R3_060_060_v001_CAM (schema: AbcGeom_Xform_v3)  <-- Camera exported as Xform!
    BDY_R3_060_060_v001_CAMShape (schema: unknown)
  TV_Screen (schema: AbcGeom_Xform_v3)
    TV_ScreenShape (schema: AbcGeom_PolyMesh_v1)
```

## Solution

### 1. Naming Convention
Add `_CAM` suffix to camera objects in Cinema4D before exporting:
- Rename camera from `MyCameraName` to `MyCameraName_CAM`

### 2. Code Detection (AlembicLoader.cpp)

The code detects Cinema4D exports via metadata and looks for the `_CAM` suffix:

```cpp
// Detect Cinema4D export from metadata
bool isCinema4DExport = false;
for (auto it = topMeta.begin(); it != topMeta.end(); ++it) {
    if (it->first == "_ai_Application" &&
        it->second.find("Cinema4D") != std::string::npos) {
        isCinema4DExport = true;
    }
}

// In the camera search function, check for _CAM suffix on Xforms
if (isCinema4DExport && IXform::matches(obj.getMetaData())) {
    std::string objName = obj.getName();
    if (objName.length() >= 4) {
        std::string suffix = objName.substr(objName.length() - 4);
        for (char& c : suffix) c = toupper(c);
        if (suffix == "_CAM") {
            // Extract camera data from Xform...
        }
    }
}
```

### 3. Extracting Camera Transform

The camera's world transform is extracted directly from the Xform's matrix:

```cpp
IXform xformObj(obj);
IXformSchema& xformSchema = xformObj.getSchema();
ISampleSelector sampleSel(time);
XformSample xformSample;
xformSchema.get(xformSample, sampleSel);

M44d matrix = xformSample.getMatrix();

// Extract position
camera->position.x = static_cast<float>(matrix[3][0]);
camera->position.y = static_cast<float>(matrix[3][1]);
camera->position.z = static_cast<float>(matrix[3][2]);

// Build view matrix (inverse of world transform)
M44d invMatrix = matrix.inverse();

// Copy to camera's view matrix in row-major order
for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 4; j++) {
        camera->viewMatrix[i * 4 + j] = static_cast<float>(invMatrix[i][j]);
    }
}
```

### 4. Key Insight: No Coordinate System Conversion Needed

**Critical**: Do NOT apply any Z-axis flip or coordinate system conversion to the camera. Both the geometry and camera are exported in C4D's native coordinate system. Applying a conversion to only one of them will break alignment.

The geometry loader doesn't convert coordinates, so the camera loader shouldn't either.

### 5. Hardcoded Lens Parameters

Since C4D doesn't export lens data in the Xform, you must set FOV/aperture to match your C4D camera settings:

```cpp
// These values must match your Cinema4D camera!
camera->fov = 54.625f;           // Horizontal FOV from C4D
camera->horizontalAperture = 22.756f;  // Sensor width from C4D
camera->verticalAperture = 22.756f * 9.0f / 16.0f;  // Assume 16:9
camera->horizontalOffset = 0.0f;
camera->verticalOffset = 0.0f;
```

To find these values in Cinema4D:
1. Select the camera
2. Go to Object tab
3. Note: Focal Length, Sensor Size (Film Gate), Field of View (Horizontal)

## Debugging Tips

### Check if camera is detected
Look for these debug messages:
```
[ALEMBIC] Detected Cinema4D export - will check for _CAM suffix on Xforms
[C4D CAMERA] Found Cinema4D camera via _CAM suffix: 'YourCameraName_CAM'
[C4D CAMERA] Position: (x, y, z)
[C4D CAMERA] Successfully created camera from Xform with _CAM suffix
[CAMERA DEBUG] Using Alembic view matrix directly
```

### Check coordinate alignment
If wireframe doesn't align:
1. Verify geometry and camera Z values have same sign (both positive or both negative)
2. Check FOV matches C4D camera settings
3. Ensure no coordinate conversion is applied to camera but not geometry (or vice versa)

### View matrix format
The view matrix is stored in **row-major** order to match the rest of the codebase:
- `viewMatrix[i * 4 + j]` = row i, column j
- Translation is in elements [12], [13], [14]

## Alternative: Bake Camera in C4D

If you want C4D to export a proper ICamera object:

1. Select the camera in Object Manager
2. Go to **Functions > Bake Objects...**
3. Check Position, Rotation, Scale
4. Set frame range
5. Click OK
6. Delete the Alembic Generator tag
7. Re-export as fresh Alembic with camera included

This will export the camera as `AbcGeom_Camera_v1` with full lens parameters.

## Files Modified

- `src/main/AlembicLoader.cpp` - Added C4D camera detection and Xform-to-camera conversion in `AlembicLoader_LoadCamera()`

## Date
November 2025
