---
title: Buffer Expansion with Masks and Track Mattes
tags: [buffer-expansion, masks, track-matte, origin-offset, comp-edge, stride, premultiplication]
---

# Buffer Expansion with Masks and Track Mattes

Effects that expand their output buffer (blurs, glows, edge extension) encounter several undocumented behaviors when the source layer has AE masks or track mattes applied. These issues don't appear with unmasked layers and are difficult to diagnose.

## Q: My effect expands the buffer in PreRender but content is shifted when the layer has a mask. How do I calculate the correct offset?

When using buffer expansion, don't assume the input is centered in the output by dividing the size difference. With masks, AE's ROI cropping changes the input size unpredictably.

**Wrong approach:**
```cpp
A_long offset_x = (output_worldP->width - input_worldP->width) / 2;
```

**Correct approach — use AE's origin fields:**
```cpp
A_long offset_x, offset_y;
if (in_data->output_origin_x != 0 || in_data->output_origin_y != 0) {
    offset_x = in_data->output_origin_x;
    offset_y = in_data->output_origin_y;
} else {
    offset_x = (output_worldP->width - input_worldP->width) / 2;
    offset_y = (output_worldP->height - input_worldP->height) / 2;
}
```

The `output_origin` values reflect AE's actual buffer geometry, including mask-induced ROI changes.

## Q: Can I request different expansion amounts on different sides (asymmetric)?

No. AE silently normalizes buffer expansion to symmetric values. If you request 110px on top/left/right and 0px on bottom, AE will provide approximately 105px on all four sides. No error is returned — the output buffer is simply larger than expected on some sides and smaller on others.

**Always request equal expansion on all four sides:**
```cpp
A_long expansion = max_radius_needed + safety_margin;
extra->output->result_rect.left   -= expansion;
extra->output->result_rect.right  += expansion;
extra->output->result_rect.top    -= expansion;
extra->output->result_rect.bottom += expansion;
```

## Q: My blur produces green/magenta corruption only when a track matte is applied. What's wrong?

Track matte buffers often have `rowbytes > width * sizeof(PixelType)` due to memory alignment padding. If your pixel loops use `width` as the row stride, you read into the next row's padding on each line, producing channel-shifted corruption.

**Always compute stride from rowbytes:**
```cpp
int stride = world->rowbytes / sizeof(PixelType);
```

This corruption is most visible at 16-bit (8 bytes/pixel where padding is common) and may be subtle at 32-bit. At 8-bit (4 bytes/pixel), AE often aligns rows to 4-byte boundaries so the bug may not manifest.

## Q: How do I detect which edges of a layer touch the composition boundary?

Compare the layer's bounds against the composition dimensions. AE provides composition dimensions via `in_data->width` and `in_data->height`:

```cpp
struct CompEdgeFlags {
    bool left, right, top, bottom;
    bool any() const { return left || right || top || bottom; }
};

CompEdgeFlags DetectCompEdges(PF_InData* in_data, PF_PreRenderExtra* extra) {
    CompEdgeFlags edges = {};
    PF_CheckoutResult in_result;
    // ... checkout layer ...
    edges.left   = (in_result.result_rect.left   <= 0);
    edges.right  = (in_result.result_rect.right  >= in_data->width);
    edges.top    = (in_result.result_rect.top    <= 0);
    edges.bottom = (in_result.result_rect.bottom >= in_data->height);
    return edges;
}
```

This is useful for effects that detect edges (blur effects, edge extension) — comp boundary edges are crop artifacts, not real content edges, and should be excluded from processing.

## Q: How do I exclude comp boundary edges from edge detection?

Two approaches:

**Approach A — Zero mask at boundary:** After computing an edge mask, zero the mask values at buffer positions corresponding to comp-touching edges. For expanded buffers, zero at `input_offset` position (where content starts), NOT at buffer position 0 (which is transparent padding for left/top edges).

**Approach B — BFS flood fill from boundaries:** Start a breadth-first search from all four buffer boundaries and clear any edge-mask pixels connected to the boundary. This automatically handles all edge shapes without needing to know which specific edges touch the comp.

```cpp
void ExcludeBoundaryEdges(float* mask, int w, int h, int stride) {
    std::vector<bool> visited(w * h, false);
    std::queue<std::pair<int,int>> q;
    for (int x = 0; x < w; x++) { q.push({x, 0}); q.push({x, h-1}); }
    for (int y = 0; y < h; y++) { q.push({0, y}); q.push({w-1, y}); }
    while (!q.empty()) {
        auto [cx, cy] = q.front(); q.pop();
        if (cx < 0 || cx >= w || cy < 0 || cy >= h) continue;
        int idx = cy * w + cx;
        if (visited[idx]) continue;
        visited[idx] = true;
        if (mask[cy * stride + cx] < 0.05f) continue;
        mask[cy * stride + cx] = 0.0f;
        q.push({cx+1,cy}); q.push({cx-1,cy});
        q.push({cx,cy+1}); q.push({cx,cy-1});
    }
}
```

## Q: My blur spreads random colors from transparent areas. The input is premultiplied — shouldn't transparent pixels be black?

Not necessarily. After masking, track matte application, or alpha-only erosion, transparent pixels can retain non-zero RGB values (`r=0.8, g=0.2, b=0.0, a=0.0`). These are invisible but contaminate blur output because the blur kernel averages their RGB into neighboring visible pixels.

**Always clean transparent pixels before blurring:**
```cpp
for (int y = 0; y < height; y++) {
    PF_PixelFloat* row = data + y * stride;
    for (int x = 0; x < width; x++) {
        if (row[x].alpha < 0.001f) {
            row[x].red   = 0.0f;
            row[x].green = 0.0f;
            row[x].blue  = 0.0f;
        }
    }
}
```

## Q: What's a fast way to detect alpha edges without Sobel/gradient operators?

For alpha-channel edge detection, this formula peaks at transitions and is zero at fully opaque or transparent:

```cpp
float edge = (1.0f - alpha) * alpha * 4.0f;
// alpha=0.0 → 0.0 (transparent, no edge)
// alpha=0.5 → 1.0 (transition, peak edge)
// alpha=1.0 → 0.0 (opaque, no edge)
```

The `* 4` normalizes the peak to 1.0. This is cheaper than morphological dilate-minus-erode and sufficient for blur masking and edge-selective effects.

## Q: My CPU render uses ctx.output() dimensions for the working buffer, but border expansion effects can't grow beyond the input boundary. What's wrong?

When a framework or abstraction layer returns a pre-offset output pointer (adjusted to where the input sits within the expanded buffer), the **reported width/height on that pointer reflect the input dimensions, not the expanded output dimensions**.

This is the core issue: SmartFX expands the output buffer for `border_expansion` effects. If your abstraction returns a pointer pre-adjusted to `(offset_x, offset_y)` with `width=input_w, height=input_h, stride=output_w`, then using that width/height as your working dimensions means you never process the expanded border area.

**The pattern:**
```
Full expanded output buffer (output_w × output_h):
┌─────────────────────────────┐
│  transparent padding        │
│   ┌─────────────────┐       │
│   │  input content  │ ← pre-offset pointer lands here
│   │  (in_w × in_h)  │       │
│   └─────────────────┘       │
│                             │
└─────────────────────────────┘
```

If you only write to `in_w × in_h`, the border expansion area stays empty.

**Fix — use the true expanded dimensions:**
1. Query the real output dimensions separately from the output pointer (e.g., `output_width()` / `output_height()` vs `output().width`)
2. Work at `max(input, output)` dimensions (the unified working-buffer model)
3. Back out the offset from the pre-adjusted pointer to get the true buffer base:
   ```cpp
   Pixel* output_base = dst.data - (offset_y * dst.stride + offset_x);
   ```
4. Write final results to `output_base` at full expanded dimensions

**Key insight:** The pre-offset pointer is convenient for simple effects that don't need border expansion — they just read/write at `(0,0)` coordinates. But effects that grow beyond the input boundary (blurs, edge extension, glows) must recover the true buffer base and work at the full expanded size.
