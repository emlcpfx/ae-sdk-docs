# Pixel_Bender

> 1 Q&A · source: AE plugin dev community Discord

### How can I convert Pixel Bender code that samples and transforms pixel coordinates to a native After Effects plugin?

You can accomplish coordinate transformation and pixel sampling using the C++ API. The recommended approach is to pull values from input layers rather than push to arbitrary pixels. Iterate through output samples and pull input values from transformed locations. For direct buffer pixel access in RAM, study the CCU sample project. For iterating through output and pulling from other locations, examine the Shifter sample. Note that subpixel interpolation to 4 adjacent pixels is not supported by the standard suite.

*Tags: `c++`, `effect_api`, `pixel_bender`, `reference`, `sampling`*

---
