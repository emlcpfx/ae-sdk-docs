# Camera

> 15 Q&As · source: AE plugin dev community Discord

### How do you correctly extract and convert After Effects camera and light matrix data for use in a Vulkan shader?

The user describes a 4-step process: (1) get camera matrix from AE, (2) convert it to column-major format, (3) normalize position in clip space using range (-1,1) with zoom value as z factor, (4) get light matrix and normalize it. However, they report that using model and projection matrix operations hasn't produced expected results. The answer suggests this is a known challenge when working with AE camera data in non-OpenGL contexts like Vulkan, and the correct transformation depends on properly handling the coordinate system conversion between AE's internal representation and the target graphics API.

*Tags: `camera`, `coordinates`, `lighting`, `matrix`, `shader`, `vulkan`*

---

### How do you extract and use camera and light matrix data from After Effects in a Vulkan shader to match AE's rendering?

The process involves: (1) retrieving the camera matrix from AE, (2) converting it to column-major format, (3) normalizing the position in clip space using the range (-1,1) with zoom value as a z factor, and (4) extracting and normalizing the light matrix. The challenge is ensuring proper matrix operations and coordinate space conversions match AE's internal rendering pipeline. Camera data is sent directly from AE to the shader. When working with Vulkan (instead of OpenGL), standard OpenGL operations like glPerspective or glMatrixMode are not available, so manual matrix calculations and transformations must be implemented in the shader code.

*Tags: `camera`, `gpu`, `lighting`, `matrix-math`, `shader`, `vulkan`*

---

### Where can I find code to get all active camera properties?

The Resizer example in the SDK contains code for accessing camera properties and interfacing with camera layers.

*Tags: `aegp`, `camera`, `layer-checkout`, `sdk`*

---

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

### How do you correctly extract and transform camera and light matrices from After Effects to use in a Vulkan shader?

The user is working with After Effects camera/light matrix data for a Vulkan-based plugin. Their approach involves: (1) getting the camera matrix from AE, (2) converting it to column-major format, (3) normalizing position in clip space (-1,1) using zoom as a z factor, and (4) getting and normalizing the light matrix. They note they are not using OpenGL operations like glPerspective or glMatrixMode since they are targeting Vulkan. The user indicates they've tried model and projection matrix operations but results don't match AE's render. This suggests the camera transformation pipeline in AE may require specific handling of coordinate systems, matrix conventions (row vs column major), and potentially AE-specific camera parameters that differ from standard graphics API conventions.

*Tags: `camera`, `gpu`, `matrix`, `render-loop`, `shader`, `vulkan`*

---

### How do you correctly extract and transform camera and light matrix data from After Effects for use in a Vulkan shader?

The process involves several steps: (1) Get the camera matrix from After Effects, (2) Convert it to column-major format, (3) Normalize the position in clip space using the range (-1, 1) and apply the zoom value as a z-factor, (4) Get the light matrix and normalize it similarly. The user was attempting this without using OpenGL operations like glPerspective or glMatrixMode (since they're using Vulkan). They encountered issues where the rendered light position didn't match the AE position even after trying various model and projection matrix operations. The key challenge is ensuring proper coordinate space conversion between After Effects' internal representation and Vulkan's expected format.

*Tags: `camera`, `coordinate-space`, `lighting`, `matrix`, `shader`, `vulkan`*

---

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

### What resources explain how to convert After Effects camera matrices to OpenGL format?

Two archived Adobe Community threads provide detailed guidance on matrix coordinate system conversion for AE camera data. The first thread at https://community.adobe.com/t5/after-effects-discussions/glator-for-dummies-and-from-dummy/m-p/6930311 discusses camera matrix conversion generally. The second thread (archived at https://web.archive.org/web/20111223234233/http://forums.adobe.com/thread/570135) provides specific code examples for transposing AE matrices to OpenGL format, explaining the row-major vs column-major difference and matrix operations needed.

*Tags: `aegp`, `camera`, `matrix`, `opengl`, `reference`*

---

### What does AEGP_GetLayerToWorldXform matrix represent and how should it be used?

AEGP_GetLayerToWorldXform translates XY coordinates from the layer origin to XYZ in composition space. It can be used for simple coordinate conversion between layer-space and composition-space, or to construct a full render view matrix from the camera's point of view.

*Tags: `aegp`, `camera`, `composition-space`, `coordinate-transform`, `layer-space`, `matrix`*

---

### How do I make a 3D plugin display the 3D icon and trigger rendering automatically when the camera moves in After Effects?

To enable 3D camera support in your After Effects plugin, you need to set the flags PF_OutFlag2_I_USE_3D_CAMERA and PF_OutFlag2_I_USE_3D_LIGHTS, and use AEGP_GetEffectCameraMatrix to access the camera matrix. However, you must also change the corresponding flags in the PiPL file. Additionally, on Windows, you need to clean and rebuild your project, as the PiPL only regenerates after a clean build.

*Tags: `3d`, `aegp`, `build`, `camera`, `pipl`*

---

### How can I get camera position in world coordinates when the camera is parented to another layer?

Use AEGP_GetEffectCameraMatrix() to get the camera matrix post all parenting transformations. You can then extract the camera position from that matrix.

```cpp
ERR(suites.StreamSuite4()->AEGP_GetLayerStreamValue(camera_layerH, AEGP_LayerStream_ANCHORPOINT, AEGP_LTimeMode_CompTime, &comp_timeT, FALSE, &stream_anchor, NULL));
```

*Tags: `3d`, `aegp`, `camera`, `matrix`, `sdk`*

---

### Why does an After Effects effect plugin crash when dragging camera geometry values instead of typing them?

The crash occurs when keyframing masks during frameSetup after getting camera data, especially when the camera layer is positioned before the current layer in the composition hierarchy. The issue was specific to After Effects CS5.5 and disappeared in CS6 and later versions. A workaround is to check the version and deselect the camera or layer if needed when working with older versions.

```cpp
ERR(suites.PFInterfaceSuite1()->AEGP_ConvertEffectToCompTime(	in_data->effect_ref,
	in_data->current_time,
	in_data->time_scale,
	&comp_timeT));
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectCamera(	in_data->effect_ref,
	&comp_timeT,
	&camera_layerH));
if (camera_layerH){
	ERR(suites.LayerSuite6()->AEGP_GetLayerToWorldXformFromView(	camera_layerH,
		&comp_timeT,
		&comp_timeT,
		&matrix));
	AEGP_StreamVal2		camVal;
	ERR(suites.StreamSuite4()->AEGP_GetLayerStreamValue( camera_layerH, AEGP_LayerStream_ZOOM, AEGP_LTimeMode_CompTime, &comp_timeT, FALSE, &camVal, NULL));
	zoom = camVal.one_d;
}
```

*Tags: `aegp`, `camera`, `debugging`, `macos`, `windows`*

---

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

### How can I set a camera to One-Node type when creating it via AEGP plugin?

Use AEGP_SetLayerFlag() with the flags AEGP_LayerFlag_LOOK_AT_POI or AEGP_LayerFlag_AUTO_ORIENT_ROTATION to control camera node type. Disabling these flags will give you one-node camera behavior. Alternatively, you can execute a script using AEGP_ExecuteScript() to set autoOrient properties directly.

```cpp
AEGP_ExecuteScript("app.project.item(compItem).layer(camLayerIndex).autoOrient = AutoOrientType.NO_AUTO_ORIENT");
```

*Tags: `aegp`, `camera`, `layer-checkout`*

---
