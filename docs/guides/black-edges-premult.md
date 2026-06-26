 How TEST_BLUR_PREMULT Works and Why It Eliminates Black Edges

  The Problem with the Original Approach

  The original implementation was following the classic "blur and unpremult" technique:
  1. Unpremultiply the input (RGB / Alpha)
  2. Fill transparent areas with nearby colors
  3. Blur the unpremultiplied data
  4. Premultiply back for compositing

  The issue: When you unpremultiply, transparent pixels (alpha = 0) create undefined RGB values. Even though we tried to fill these with nearby colors, the blur kernel would
  still sample some black/undefined pixels at edges, creating the dark contamination.

  The Photoshop-Style Solution (TEST_BLUR_PREMULT)

  Instead of unpremultiplying first, we:
  1. Keep data premultiplied (skip the unpremult step)
  2. Blur directly in premultiplied space
  3. Data stays premultiplied for compositing

  Why this works:
  - In premultiplied space, transparent pixels are naturally (0,0,0,0) - true transparent black
  - When the blur samples these pixels, it gets zeros which don't contribute darkness
  - The blur naturally "spreads" the premultiplied colors outward
  - No undefined values, no black contamination

  The key insight: Photoshop likely does its blurs in premultiplied space by default, which is why manual edge extension in Photoshop doesn't create black edges.

## Preserving the premultiplied RGB/alpha invariant

The black-edge problem above is one instance of a broader rule. When you work in
premultiplied float RGBA, the invariant is:

```
RGB_premult = RGB_straight * alpha
```

Any time you change a pixel's alpha *without* scaling its RGB by the same ratio,
you break that relationship and the implied straight color shifts:

- **Expanding alpha** (e.g. 0.3 -> 0.8) without scaling RGB makes the straight
  color too dark: `RGB_straight = RGB_premult / new_alpha` shrinks.
- **Constraining alpha** (e.g. 0.9 -> 0.5) without scaling RGB makes the straight
  color too bright.

This is why alpha-only "expand alpha" / "constrain to alpha" routines that were
written for a host whose SDK manages premultiplication internally produce
dark/bright halos when run on raw premultiplied pixels: in that other model the
iterate/transfer machinery rescales RGB for you; in raw premultiplied space
nothing does.

**The rule:** whenever you change a pixel's alpha in premultiplied space, scale
RGB by `new_alpha / old_alpha` to keep the invariant.

Expand alpha -- scale RGB up as alpha increases:

```cpp
float old_a = px.a;
// ... compute new alpha 'a' ...
if (a > old_a && old_a > 0.001f) {
    float scale = a / old_a;
    px.r *= scale;
    px.g *= scale;
    px.b *= scale;
}
px.a = a;
```

Constrain alpha -- scale RGB down as alpha decreases:

```cpp
if (in.a > 0.001f) {
    float scale = adj_a / in.a;
    return { in.r * scale, in.g * scale, in.b * scale, adj_a };
}
return { 0, 0, 0, adj_a };   // fully transparent: RGB is meaningless, zero it
```

This applies to alpha expansion, alpha constraint, alpha remapping, and any other
alpha modification -- not just blur.

> **Premultiplication context differs by host.** After Effects always delivers
> and expects premultiplied pixels, and its iterate suites and `transfer_rect`
> (with `PF_MF_Alpha_PREMUL`) handle the RGB scaling for you -- so alpha-only
> changes are safe *inside that machinery*. When you operate on the buffer
> directly, or on a host (such as OFX) where the input may be straight and you
> convert to premultiplied yourself, the SDK does **not** compensate -- your code
> must maintain the invariant. Before reusing alpha-manipulation code from one
> host in another, check whether the SDK was silently rescaling RGB.
