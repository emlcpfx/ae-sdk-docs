# Buffer

> 4 Q&As · source: AE plugin dev community Discord

### Does Premiere support SmartFX for requesting only a portion of the input buffer?

Premiere does not support SmartFX. You cannot officially request only a specific region of the buffer like you can in AE. As a workaround, you can manually iterate through the data using row bytes to delimit the part you need. Be careful with negative row byte values in Premiere.

*Tags: `buffer`, `premiere`, `region-of-interest`, `row-bytes`, `smartfx`, `transition`*

---

### Why does applying a buffer-expanding effect like blur after a SmartFX effect cause an offset in the output?

When using SmartFX to expand the output buffer to the full composition size, subsequent buffer-expanding effects (such as blur or drop shadow) can alter the output origins. To fix this, you need to explicitly set the output->origin_x and output->origin_y parameters in your SmartFX preRender function. This ensures that downstream effects don't inadvertently offset your effect's output.

```cpp
UnionLRect(&req.rect, &extra->output->result_rect);
UnionLRect(&req.rect, &extra->output->max_result_rect);
// Also set:
extra->output->origin_x = /* appropriate value */;
extra->output->origin_y = /* appropriate value */;
```

*Tags: `buffer`, `output-rect`, `sdk`, `smartfx`*

---

### How can you integrate Cairo graphics library with the After Effects SDK for 2D drawing?

To integrate Cairo with the AE SDK, you need to draw onto a native Cairo buffer, and when you're done, convert it pixel by pixel into an AE buffer. This approach is simpler than OpenGL for modest 2D graphics tasks, as you don't need to manage framebuffer objects and contexts. Instead of using schemes like sizeof(GL_RGBA) for buffer calculations, you perform a pixel-by-pixel conversion from the Cairo buffer to the AE output buffer.

*Tags: `2d-rendering`, `buffer`, `cairo`, `graphics`, `sdk`*

---

### Why does buffer height calculation blow up to around 500 when iterating through rows with zero gutter values?

The issue was caused by incorrect casting of the world as the base address for pixels. The correct approach is to use AEGP_GetBaseAddr32(worldH, &baseAddress) to properly obtain the base address for pixel access, rather than casting the world handle directly.

```cpp
ERR(ws3P->AEGP_GetBaseAddr32(worldH, &baseAddress));
```

*Tags: `aegp`, `buffer`, `debugging`, `memory`, `render-loop`*

---
