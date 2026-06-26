# Rendering

The render pipeline — how your plugin processes pixels.

## Smart Render (Modern Path)

The two-phase render is how all modern plugins work:

1. **`PF_Cmd_SMART_PRE_RENDER`** — Tell AE what input rectangles you need, request layers
2. **`PF_Cmd_SMART_RENDER`** — Actually process pixels using checked-out layers

```cpp
// Pre-render: request input
ERR(extra->cb->checkout_layer(in_data->effect_ref, 0, 0, &req,
    in_data->current_time, in_data->time_step, in_data->time_scale, &checkout));

// Smart render: process
PF_EffectWorld *input, *output;
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input));
output = params[0]->u.ld;  // Output world
```

## Key Concepts

- **Effect World** (`PF_EffectWorld`) — A pixel buffer with width, height, rowbytes, and pixel data
- **Rowbytes** — The stride of each row in bytes. **Always use rowbytes**, never assume `width * pixel_size`
- **Output rect expansion** — If your effect needs neighboring pixels (like blur), expand the output rect in pre-render
- **Iterate suites** — Use `PF_ITERATE` or `iterate_generic` for per-pixel processing instead of raw loops

---

*23 documents in this section. Use **Search** (top of page) to find what you need.*
