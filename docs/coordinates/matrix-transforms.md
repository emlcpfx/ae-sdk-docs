# Matrix Transforms

> 1 Q&A · source: AE plugin dev community Discord

### How do you extract and use camera and light matrix data from After Effects in a Vulkan shader to match AE's render output?

The user is working on extracting camera matrix from AE and converting it to column-major format, then normalizing position in clip space using zoom as a z factor, followed by getting the light matrix and normalizing it. They mention attempting model and projection matrix operations but getting unexpected results. The key challenge is ensuring the matrix transformations match AE's internal coordinate system when not using OpenGL operations like glPerspective or glMatrixMode, particularly when working with Vulkan instead.

*Tags: `camera`, `coordinate-system`, `lighting`, `matrix-transforms`, `shader`, `vulkan`*

---
