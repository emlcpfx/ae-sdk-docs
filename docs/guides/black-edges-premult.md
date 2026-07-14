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

## Some formulas already OUTPUT premultiplied -- re-premultiplying them darkens edges

The rule above ("re-premultiply after you touch RGB") has a dangerous exception.
Several standard compositing formulas -- borrowed from Nuke and elsewhere -- are
*defined* to produce a premultiplied result. Wrapping them in the usual
unpremultiply / process / re-premultiply sandwich multiplies by alpha twice.

The canonical case is IBK-style **screen subtraction** (despill by subtracting the
clean plate):

```
out = fg + cleanPlate * (alpha - 1.0)
```

Model an edge pixel of a green-screen plate as a mix of the true foreground and
the screen:

```
fg  = a*FG + (1-a)*screen
out = fg - cp*(1-a)          (cp stands in for screen)
    = a*FG                   <-- ALREADY premultiplied by a
```

That `a*FG` is the premultiplied answer. It is the *point* of the formula: the
subtraction removes exactly the screen contribution that a partially transparent
pixel picked up. So the correct store is:

```cpp
// out_* are premultiplied. Store them as-is.
SetRGBA(pixel, out_r, out_g, out_b, alpha);
```

Do **not** do this:

```cpp
// WRONG: out_* were already premultiplied. This yields FG * a^2.
SetRGBA(pixel, out_r * alpha, out_g * alpha, out_b * alpha, alpha);
```

### Why it is so hard to spot

The extra factor of alpha is `1.0` at `alpha == 1` and `0.0` at `alpha == 0`, so
**opaque interiors and fully transparent regions are both pixel-perfect**. The
error is confined to `0 < alpha < 1` -- and it is worst where alpha is *lowest*.
The result is a dark rim that hugs precisely the soft edge (hair, motion blur,
defocus) while the rest of the frame looks flawless. It reads as a keying-quality
problem, not a maths bug, so it survives every "fix" aimed at the matte.

Implementations often make it even easier to miss by skipping the trivial cases:

```cpp
if (alpha <= 0.001f || alpha >= 0.999f) continue;   // only edges are touched
```

### How to check: the straight-colour invariant

> **Straight colour (`premult / alpha`) is INVARIANT across a correct
> premultiply.** Premultiply scales RGB by alpha and leaves alpha alone, so the
> ratio cannot move. Any stage that changes it is doing something to the colour.

This is the single most useful diagnostic in this whole family of bugs, because it
turns "which stage is wrong?" into a measurement instead of an argument. Print the
straight colour of semi-transparent pixels at every stage boundary:

- straight colour **unchanged** across a premultiply -> correct.
- straight colour **drops by a factor of `alpha`** -> you premultiplied **twice**.
- straight colour **rises by a factor of `1/alpha`** -> you unpremultiplied twice,
  or unpremultiplied something that was already straight.

Two traps worth naming, because they cost days:

**1. Do not compare the "darkest" pixels between stages -- compare the DELTA.**
The darkest pixels in a frame are usually its black letterbox border, which is
legitimately black and identical at every stage. It tells you nothing. The halo is
not the darkest thing in the frame; it is the thing that *got* darker. Report the
biggest *drop* per stage, and exclude the frame border.

**2. Knowing the buffer convention does NOT tell you what the buffer holds.**
"This host's worlds are premultiplied" does not imply "premultiply at the end."
A kernel that applies a matte by scaling only alpha and leaving RGB alone is
*already* emitting premultiplied output if its input was premultiplied -- that is
exactly what AE's `transfer_rect` (`PF_Xfer_COPY` + mask) does. Premultiplying
again is the double. Measure with the invariant above; do not infer.

### The two errors cancel, which is why they survive

A missing premultiply brightens semi-transparent pixels; a double premultiply
darkens them. Put both in one pipeline and it can look *correct*. Fix only one and
the other suddenly appears -- which reads as a regression you just introduced,
when in fact you removed the concealer. If fixing an alpha bug makes a *new* edge
artifact appear, suspect that you have just unmasked a second one rather than
caused it.

### Clamp range

A premultiplied channel is bounded by its alpha, not by 1.0:

```cpp
out_r = std::max(0.0f, std::min(alpha, out_r));   // not min(1.0f, ...)
```

Clamping premultiplied values to `[0, 1]` lets a pixel exceed its own alpha,
which is an invalid premultiplied colour (implied straight value > 1.0) and will
read as a bright fringe once composited.
