# Coordinates & Transforms

Coordinate spaces, transforms, and matrix math in AE plugins.

## In-Depth Guides

- [The AE Transform Pipeline](transform-pipeline.md) — layer / comp / view / screen spaces and how to convert between them.
- [AE Camera Matrix Conventions](ae-camera-matrix-conventions.md) — row-vector vs column-vector, the byte-reinterpretation = transpose identity, the inversion trap, AE Y-down → NDC Y-up, `flipZ`, and ortho non-orthonormality. **Read this before integrating an AE camera into Vulkan / WebGPU / Metal / CUDA.**
- [AE Camera Pipeline](ae-camera-pipeline.md) — the AEGP fetch sequence (effect_ref → comp → camera layer), Zoom → FOV conversion, default-camera fallback, and a CPU `projectWorldToComp` mirror for Custom UI gizmos.
- [HDRI / Lat-Long Flipped Vertically (AE Y-Down)](hdri-latlong-y-flip.md) — why an equirectangular HDRI renders upside down in an AE path tracer (use `v = acos(-d.y)/pi`, not `acos(d.y)`), CPU/GPU sampler parity, and keeping a procedural sky generator consistent.
- [Projected Texture UVs Flip on 3D Geometry (AE Y-Down)](projected-uv-y-flip.md) — the projection sibling of the HDRI flip: planar / box / cylindrical / spherical UVs generated from AE-space geometry come in upside-down and mirrored; bake a both-axis flip (negate tile) for generated projections, leave From-File UVs alone.

## AE Coordinate Spaces

- **Layer space** — pixel coordinates relative to the layer (0,0 is top-left of the layer)
- **Composition space** — coordinates relative to the comp
- **World space** — for 3D layers, includes camera transforms

## Common Operations

- **Layer → Comp**: Use `AEGP_LayerSuite::GetLayerToWorldXform` or the layer's transform matrix
- **Comp → Layer**: Invert the layer-to-comp matrix
- **Downsample factor**: AE may render at reduced resolution — check `in_data->downsample_x/y`

## Matrix Math

AE uses **3x3 matrices** for 2D transforms and **4x4 matrices** for 3D:

```cpp
// 3x3 for 2D: [a b c; d e f; 0 0 1]
// Translation: mat[0][2] = tx, mat[1][2] = ty
// Scale: mat[0][0] = sx, mat[1][1] = sy
```

**Watch out for**: Row-major vs column-major conventions — AE SDK uses a specific convention that may differ from your math library. See [AE Camera Matrix Conventions](ae-camera-matrix-conventions.md) for a full treatment.

---

*30 documents in this section. Use **Search** (top of page) to find what you need.*
