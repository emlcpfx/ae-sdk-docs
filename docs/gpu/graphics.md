# Graphics

> 1 Q&A · source: AE plugin dev community Discord

### How can you integrate Cairo graphics library with the After Effects SDK for 2D drawing?

To integrate Cairo with the AE SDK, you need to draw onto a native Cairo buffer, and when you're done, convert it pixel by pixel into an AE buffer. This approach is simpler than OpenGL for modest 2D graphics tasks, as you don't need to manage framebuffer objects and contexts. Instead of using schemes like sizeof(GL_RGBA) for buffer calculations, you perform a pixel-by-pixel conversion from the Cairo buffer to the AE output buffer.

*Tags: `2d-rendering`, `buffer`, `cairo`, `graphics`, `sdk`*

---
