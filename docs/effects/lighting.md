# Lighting

> 3 Q&As · source: AE plugin dev community Discord

### How do you correctly extract and convert After Effects camera and light matrix data for use in a Vulkan shader?

The user describes a 4-step process: (1) get camera matrix from AE, (2) convert it to column-major format, (3) normalize position in clip space using range (-1,1) with zoom value as z factor, (4) get light matrix and normalize it. However, they report that using model and projection matrix operations hasn't produced expected results. The answer suggests this is a known challenge when working with AE camera data in non-OpenGL contexts like Vulkan, and the correct transformation depends on properly handling the coordinate system conversion between AE's internal representation and the target graphics API.

*Tags: `camera`, `coordinates`, `lighting`, `matrix`, `shader`, `vulkan`*

---

### How do you extract and use camera and light matrix data from After Effects in a Vulkan shader to match AE's rendering?

The process involves: (1) retrieving the camera matrix from AE, (2) converting it to column-major format, (3) normalizing the position in clip space using the range (-1,1) with zoom value as a z factor, and (4) extracting and normalizing the light matrix. The challenge is ensuring proper matrix operations and coordinate space conversions match AE's internal rendering pipeline. Camera data is sent directly from AE to the shader. When working with Vulkan (instead of OpenGL), standard OpenGL operations like glPerspective or glMatrixMode are not available, so manual matrix calculations and transformations must be implemented in the shader code.

*Tags: `camera`, `gpu`, `lighting`, `matrix-math`, `shader`, `vulkan`*

---

### How do you correctly extract and transform camera and light matrix data from After Effects for use in a Vulkan shader?

The process involves several steps: (1) Get the camera matrix from After Effects, (2) Convert it to column-major format, (3) Normalize the position in clip space using the range (-1, 1) and apply the zoom value as a z-factor, (4) Get the light matrix and normalize it similarly. The user was attempting this without using OpenGL operations like glPerspective or glMatrixMode (since they're using Vulkan). They encountered issues where the rendered light position didn't match the AE position even after trying various model and projection matrix operations. The key challenge is ensuring proper coordinate space conversion between After Effects' internal representation and Vulkan's expected format.

*Tags: `camera`, `coordinate-space`, `lighting`, `matrix`, `shader`, `vulkan`*

---
