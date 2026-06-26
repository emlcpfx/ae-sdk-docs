# Color Management for AE Effects (HDR Renderers, Path Tracers, Generators)

If your effect produces high-dynamic-range, scene-referred light — a 3D
renderer, a path tracer, a procedural sky, a physically-based generator — the
single most important color decision is: **output scene-linear and let the
*host* apply the view transform. Do not bake a tonemap/display curve into the
pixels you hand back to AE.**

This page is the "why" and the "how it interacts with AE." For the broader
industry context (ACES, OCIO, ACEScg), see the references at the bottom.

## The mental model

A renderer computes **scene-referred linear radiance** — physically linear
light, unbounded (values well above 1.0 in highlights). That is the *correct*
thing to deliver. Two separate questions hang off it:

1. **Primaries / gamut** — which RGB primaries those linear values live in
   (linear-sRGB/Rec.709, or ACEScg/AP1).
2. **View transform** — the curve+gamut-map that makes scene-linear viewable on
   an SDR display (sRGB, the ACES Output Transform, AgX, a show LUT).

The renderer owns #1. The **view transform (#2) belongs at the display/comp
stage**, applied non-destructively — *not* baked into the delivered pixels.

## Why not bake the tonemap in

Baking a filmic/ACES/sRGB curve into your effect output:

- **destroys the linear values compositing needs** — `over`, relight, regrade,
  and merging against a plate are all linear-light operations;
- **crushes the HDR** — everything above the curve's shoulder clips, so the
  highlight detail a path tracer worked to compute is gone;
- **mismatches linear-EXR plates and log footage** — a tonemapped Rec.709 layer
  composited over a scene-linear EXR is simply wrong (different transfer *and*
  different dynamic range).

A game engine bakes a pretty SDR image because it *is* the display. A VFX/mograph
renderer feeds a compositor, so it must stay scene-referred.

## How this interacts with AE specifically

AE effects receive and return pixels **in the project's working space**. What
that means depends on the project:

- **Color-managed project (AE 2023+ with OCIO / a working space):** AE applies
  input transforms to footage and a **display/view transform at the viewer**,
  around your effect. If you *also* bake a transform, you get the
  **double-transform trap** — the look is applied twice and everything is wrong.
  In a managed ACES project the working space is typically **ACEScg-linear**;
  your effect should output ACEScg-linear and nothing else.
- **Legacy / non-managed project (8-bit, no working space):** AE does *not*
  apply a view transform. Scene-linear output here looks "too dark / flat" to
  the user, because linear shown as-is isn't perceptual. These users expect a
  display-ready (tonemapped + sRGB) image.

You serve both audiences, so the robust design is: **output scene-linear by
default for the managed/comp path, and offer a baked "display" view transform as
an explicit option** for the non-managed user. Make the user match the mode to
their project (or, long-term, query the project's color state — the plugin API
for that is thin, so document it for now).

Requirements:

- **Work in 32-bit float.** Scene-linear HDR needs it; 8/16-bit integer can't
  hold the >1.0 values or the linear precision. (See `getting-started/32bpc.md`.)
- **Premultiplied, straight-alpha conventions** still apply — the view transform
  operates on the color channels; keep alpha linear.

## Delegate OCIO to the host — don't embed it

You almost never want to link the OpenColorIO library into an AE effect:

- **AE native color management (2023+)** handles input + display transforms when
  the project is OCIO/ACES-managed. Output working-space linear and AE does the
  rest — the user may not even need a separate effect.
- **The OpenColorIO effect** (the fnord plug-in) is the manual fallback for
  per-layer transforms in a non-managed project.
- **OFX hosts (Resolve, Nuke)** have first-class OCIO. The same "output
  scene-linear, host applies the look" contract works there unchanged.

Embedding OCIO means shipping/maintaining configs, version-skew with the host's
config, and re-solving the double-transform problem yourself. The conversions
you *do* need in-effect (linear↔sRGB, Rec.709↔ACEScg primaries, an ACES Output
Transform fit, AgX) are a handful of constants and a 3×3 matrix — no library
required. Reserve real config-driven OCIO for a pro/v2 feature with demand.

## Primaries conversion is just a 3×3 matrix

Going from linear-Rec.709/sRGB primaries to ACEScg (AP1) is a constant matrix —
no OCIO needed:

```
Rec.709/sRGB linear → ACEScg (AP1):
  [ 0.6131  0.3395  0.0474 ]
  [ 0.0702  0.9164  0.0134 ]
  [ 0.0206  0.1096  0.8698 ]
```

Offer the user a "Scene-Linear (Rec.709)" and a "Scene-Linear (ACEScg)" output so
they can match the host working space; both stay linear (no encode), the only
difference is this matrix.

## Pipeline order

```
render (scene-linear) → denoise (scene-linear; OIDN wants linear) → [output]
   ├─ comp path:    scene-linear, optional Rec.709→AP1 matrix, NO view transform
   └─ display path: view transform (sRGB / ACES OT / AgX) + encode  ← explicit, opt-in
```

Denoise **before** any view transform — OIDN and NRD expect HDR linear; denoising
post-tonemap is wrong.

## Checklist

1. Render and accumulate in **scene-linear float**.
2. **Don't** bake a tonemap into the delivered pixels by default.
3. Expose an explicit **Output Color** choice: scene-linear (for comp / EXR /
   log) vs a display view transform (for non-managed viewing).
4. Offer **ACEScg** output primaries (a 3×3 matrix) for ACES-managed hosts.
5. **Delegate the view transform / OCIO to the host**; don't embed OCIO.
6. Warn about the **double-transform trap**: managed project ⇒ scene-linear out;
   non-managed project ⇒ baked display out. Never both.
7. Work in **32-bit float**; denoise before any view transform.

## Related pages

- [32-bit / float pixels](../getting-started/32bpc.md)
- [2D rendering](2d-rendering.md)
