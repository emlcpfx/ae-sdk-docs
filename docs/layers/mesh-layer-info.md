# A Layer's Matrix Does Not Carry Its Source Dimensions

> A layer-to-world matrix encodes scale x rotation x translation -- it does *not*
> encode the layer's pixel size. To compute a layer's world-space extent you need
> its source dimensions (`AEGP_GetItemDimensions`) *and* the matrix scale,
> multiplied together. The matrix alone cannot distinguish a big layer from a
> tiny one at the same scale.

## Symptom

Code that derives a layer's world-space size from its transform matrix gets it
wrong for anything but a coincidentally unit-sized layer. A concrete case: an
area light sized from a 3D solid came out as roughly a 1x1 emitter regardless of
the solid's actual size, so its contribution was effectively zero and the scene
rendered black even at high intensity.

## Root cause

It is tempting to read a layer's size from the length of its local X/Y axis
columns in the layer-to-world matrix. But a transform matrix carries only
**scale x rotation x translation** -- never the source dimensions. For an
unscaled (100%) solid, the matrix' X-axis column has length 1.0, not the solid's
pixel width.

The correct relationship is:

```
world half-extent = (source pixels x matrix scale) / 2
```

Without the source dimensions you cannot tell a 100x100 solid at 100% scale
(world half-extent 50) from a 1x1 layer at 100% scale (half-extent 0.5) -- the
matrices are identical.

## The fix

Fetch the source dimensions separately and combine them with the matrix scale.
Use `AEGP_GetItemDimensions` on the layer's **source item**:

```cpp
A_long src_w = 0, src_h = 0;
ItemSuite->AEGP_GetItemDimensions(source_itemH, &src_w, &src_h);

// scale_x / scale_y come from the matrix axis-column lengths (or a decompose)
float half_w = 0.5f * float(src_w > 0 ? src_w : 100) * scale_x;  // 100 = AE default solid
float half_h = 0.5f * float(src_h > 0 ? src_h : 100) * scale_y;
```

If you already fetch the source item for another reason (a source-path lookup,
say), this is a one-call addition with no new suite cost. Defaulting to 100 (AE's
default solid size) keeps things safe for synthetic layers whose source dims
aren't available.

## How to avoid it

Whenever new code interprets a layer's matrix as containing world-space extents,
remember it does not -- the matrix is purely the transform. World extents combine
the matrix scale with the source-item geometry. This applies to area lights,
bounding boxes, emissive surfaces, and anything that needs to know "how big is
this layer in world units."

*Tags: `aegp`, `layer`, `matrix`, `source-dimensions`, `geometry`, `world-space`*
