# HDRI / Lat-Long Environment Flipped Vertically (AE Y-Down)

> Source: PluginPort / Vox path tracer (2026-06). A recurring "the HDRI is
> upside down" bug when lighting a 3D scene inside an AE effect.

### Why does my equirectangular HDRI render vertically flipped (sky on the floor) inside an AE plugin?

Because After Effects world space is **Y-down** (+Y points toward the bottom of
the frame), but a standard equirectangular HDRI is authored **Y-up** with the
sky at the **top row** of the image (texture `v = 0`) and the ground at the
bottom row (`v = 1`).

When you sample a lat-long panorama you map a world direction to UV with:

```
u = atan2(d.x, -d.z) / (2*pi) + 0.5
v = acos(d.y) / pi
```

That `v = acos(d.y)` is correct in a **Y-up** renderer. In AE's **Y-down**
world it samples the panorama upside down: "looking up" in AE is `d.y = -1`,
which `acos(-1) = pi` maps to `v = 1` (the ground row) — so the sky ends up on
the floor.

**Fix:** negate `d.y` in the latitude term so the AE-down direction lands on
the right row of a Y-up HDRI:

```
v = acos(-d.y) / pi      // -d.y = +1 (AE up) -> v = 0 -> HDRI top (sky)
```

This is the single most common HDRI mistake when bolting a path tracer / IBL
onto an AE effect. The longitude (`u`) term is usually fine; it is almost
always the latitude (`v`) that is flipped.

*Tags: `coordinates`, `3d`, `rendering`, `gpu`, `hdri`, `ibl`, `y-down`*

---

### My HDRI looks right on the CPU but flipped on the GPU (or vice versa). Why?

You have two samplers that disagree on the Y convention. This is easy to do
when the CPU reference sampler and the GPU shader were written at different
times: one uses `acos(-d.y)`, the other `acos(d.y)`. They must use the **same**
convention.

A second trap: if you ALSO generate a **procedural sky** (sun/sky dome) into the
same environment texture, that generator was written to match whichever
convention the sampler had. If you fix the sampler's `v`, the procedural sky
will then flip. Flip the generator's row-to-direction mapping in the same
change so it stays consistent:

```
// generator, per output row py (v = (py+0.5)/H):
d.y = -cos(v * pi)       // matches v = acos(-d.y)/pi; top row -> d.y=-1 (AE up) -> sky
```

Net effect: the procedural sky is visually unchanged, an uploaded HDRI is no
longer flipped, and CPU + GPU agree.

*Tags: `coordinates`, `3d`, `rendering`, `gpu`, `hdri`, `parity`, `y-down`*

---

### Should I flip the HDRI pixels on upload instead of fixing the sampler?

Prefer fixing the sampler's `v = acos(-d.y)`. Flipping rows on upload only works
if **every** consumer of that texture (background miss, diffuse/specular GI,
reflections, any per-Gaussian/per-vertex bake) reads it the same way — and it
silently breaks a shared procedural-sky generator that writes into the same
texture. The math fix is one line, lives next to the sampling, and keeps a
single source of truth for the convention.

*Tags: `coordinates`, `rendering`, `hdri`, `gpu`*
