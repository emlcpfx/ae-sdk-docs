# Scatter / Forward-Render Effects Must Draw Inactive Pieces In Place

> Effects that move pieces forward into the output (shatter, disintegrate,
> cascade, shockwave-driven) must render *every* piece every frame. Skipping a
> piece because it hasn't been activated yet leaves a hole, not a static piece.

## Symptom

A scatter-based effect (shatter, dissolve wavefront, particle cascade) produces
mostly transparent output at the start of the animation. Only the pieces near the
force center -- the ones already "activated" by the wavefront -- appear; every
other piece is simply gone, instead of sitting still until the wavefront reaches
it.

## Root cause

Forward / scatter render kernels iterate over source pieces and write each one
into the destination at its transformed position. A natural-looking early-out
skips pieces that haven't been activated:

```cpp
if (xf.activation < 0.01f) return;   // BUG: the piece vanishes
```

At the trigger frame, only pieces with `activation > 0` get written. All others
are never drawn anywhere -- so the viewer sees a near-empty frame with a few
fragments around the explosion point. Because nothing writes those pixels, the
output stays transparent there.

This is easy to get inconsistent: one kernel in the effect (say, the disintegrate
path) handles it correctly, while a sibling kernel (the shatter path) is missing
the in-place branch.

## The fix

An unactivated piece must still be rendered -- at its **original source
position** with the identity transform -- before the early return:

```cpp
if (xf.activation < 0.01f) {
    // Render at original position (static until the wavefront hits)
    int out_x = src_x + input_offset_x;
    int out_y = src_y + input_offset_y;
    if (in_bounds(out_x, out_y)) {
        write_pixel(out_x, out_y, source_color);
    }
    return;
}
```

Apply the same in-place path to *every* kernel and CPU fallback loop in the
effect, not just one.

## How to avoid it

Any effect with phased activation (shockwave, dissolution wavefront, cascade)
built on a scatter / forward-render architecture has three per-element states,
and all three must write output:

1. **Inactive** -> render at the original position (identity transform).
2. **Activating** -> render with the interpolated/partial transform.
3. **Fully active** -> render with the full transform.

Never skip inactive elements -- skipping creates holes. When the effect has
multiple scatter kernels (one per mode, plus a CPU fallback), verify each one has
the in-place branch; it is common for one to have it and another to be missing
it.

*Tags: `render`, `scatter`, `shatter`, `activation`, `gpu`, `cpu`*
