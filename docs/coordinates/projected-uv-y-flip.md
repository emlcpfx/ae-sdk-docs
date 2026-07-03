# Projected Texture UVs Flip on 3D Geometry (AE Y-Down)

> Source: PluginPort / Lux 3D (2026-07). The projection sibling of
> [HDRI / Lat-Long Flipped Vertically](hdri-latlong-y-flip.md): when you
> *generate* UVs by projecting a texture onto a model's surface inside an AE
> effect, the result comes in upside-down (and often mirrored), because the
> projection is computed from AE's **Y-down** world positions while the image is
> authored top-down.

## Symptom

A plugin that path-traces / textures 3D geometry inside AE lets the user project
a texture onto a mesh whose file UVs are unusable -- planar / box / cylindrical /
spherical projection. The texture maps onto the surface, but it renders:

- **upside-down** (a phone-screen graphic has its clock at the bottom), and
- often **mirrored** left-to-right as well.

The user has to flip both axes by hand (negative UV scale on U *and* V) to make
it read correctly.

## Root cause

Procedural projections derive UV from the vertex **position** in world space,
e.g. a planar projection onto the XY plane:

```
u = (p.x - bbox.min.x) / bbox.size.x
v = (p.y - bbox.min.y) / bbox.size.y
```

After Effects world space is **Y-down** (+Y points toward the bottom of the
frame), and its handedness differs from a typical Y-up modelling package. So:

- **V is inverted.** Increasing AE Y goes *down* the screen, but a texture's
  `v = 0` is its *top* row. The projected V runs opposite the image, and the
  texture lands upside-down.
- **U is often mirrored.** The X / Z handedness after import flips the projected
  U for surfaces facing the viewer, so the image reads reversed as well.

Net effect: a generated projection is rotated 180 degrees from what the image
expects. Contrast **From File** UVs, which are already authored in the file's own
convention and need no correction -- only *generated* projections are affected.

## The fix

Bake a both-axis flip into the projected UVs so the default is upright. Because
projection transforms usually apply tiling / offset / rotation about the UV
center `(0.5, 0.5)`, a **negated tile is a clean flip about center**:

```cpp
// After computing the raw projection, for GENERATED projections only
// (leave From-File UVs untouched):
if (projection != Projection::FromFile) {
    tile_u = -tile_u;   // u' = 1 - u   (mirror)
    tile_v = -tile_v;   // v' = 1 - v   (vertical flip)
}
```

Now Tile = `1, 1` is upright, and a *negative* tile becomes the opt-out for the
rare part that still needs a mirror.

If you expose UV controls to the user, clamp **nothing** to positive -- allow
negative tiling so a flip stays reachable, and note in the tooltip that
projections are auto-oriented for AE.

## How to avoid it

Treat **any UV you generate from AE-space geometry** as living in AE's Y-down,
mixed-handed frame, and convert to the image's top-down frame exactly once -- the
same discipline as the HDRI lat-long case (`v = acos(-d.y)/pi`), just for a
surface projection instead of an environment lookup. File-authored UVs are
already in the right frame; only correct the UVs you compute yourself.

## See also

- [HDRI / Lat-Long Flipped Vertically (AE Y-Down)](hdri-latlong-y-flip.md) -- the
  same Y-down flip for equirectangular environment sampling.
- [AE Camera Matrix Conventions](ae-camera-matrix-conventions.md) -- the broader
  "AE is Y-down" treatment for cameras and NDC.

*Tags: `coordinates`, `y-down`, `uv`, `texture`, `projection`, `3d`, `path-tracer`*
