# AE Camera Matrix Conventions: Row-Vector, Column-Vector, and the WebGPU/Vulkan Trap

When you take an After Effects camera matrix and feed it into a graphics API
(WebGPU, Vulkan, Metal, OpenGL, CUDA), you almost always hit the same wall:
the math compiles, the shader runs, the output is dead — black, off-screen,
or pointed away from the scene. The wall is a **convention mismatch** between
how AE stores matrices and how modern graphics APIs interpret the same bytes.

This page covers the conventions, the load-bearing identity that lets you
skip an explicit transpose, and the inversion rule that catches almost
everyone the first time.

## The three conventions in play

| Side | Vector style | Composition | Translation slot | Storage |
|------|--------------|-------------|------------------|---------|
| AE `A_Matrix4` | **row vector** `p · M` | `M_a · M_b` first applies `M_a` | row 3 | row-major |
| C++ POD wrapping AE | row vector | row 3 = translation | row-major (`m[row][col]`) | row-major |
| WGSL `mat4x4<f32>` uniform | **column vector** `M · p` | `M_a · M_b` first applies `M_b` | column 3 | column-major |
| HLSL / GLSL `mat4` (default) | column vector | column 3 = translation | column-major |
| Vulkan / Metal / DX12 push constants | column vector | column 3 | column-major |

AE is row-vector, almost every modern shader API is column-vector. Both
conventions can describe the same transform, but they read the same 16
floats differently.

## The load-bearing identity

> **Reinterpreting a row-major buffer as column-major equals transposing it.**

This is the one fact that makes the whole pipeline work without an explicit
C++ transpose call. The same 16 floats stored as row-major-row-vector
appear to a column-major-column-vector consumer as the **transpose** of
the original matrix.

So for a "world → camera" view matrix stored row-major-row-vector:

```
WGSL's  (M_wgsl · p_col)   ==   AE's  (p_row · M_ae)ᵀ
                           ==   AE's  M_ae · p_col   (because Mᵀ on row-vector = M on column-vector)
                           ==   correct world-to-camera transform
```

**Sending the raw bytes to your shader composes to the right `M · p`
transform with no C++ transpose.** Memcpy AE's matrix bytes into your
uniform buffer, declare it `mat4x4<f32>` in WGSL (or `mat4` in GLSL/HLSL
with default column-major), and `view * world_pos` works.

## The inversion rule — invert the bytes, do NOT transpose first

This is the trap. When you need an **inverse** view/projection (e.g. to
reconstruct a world-space ray from NDC for raymarching), the rule is:

> **Invert the matrix in place, treating the buffer as row-vector.
> Do not transpose before or after.**

Why: column-major reinterpretation is *already* a transpose. If you
transpose the bytes before inverting, the shader's reinterpretation
applies a second transpose, and the two cancel out — but they don't
cancel into the inverse you wanted; they cancel into something that
looks like a valid matrix but reconstructs rays in the wrong direction.

The shader compiles, the AABB intersection runs, the raymarch executes —
and produces black pixels because the rays are aimed nowhere useful.

Concretely, if `view[16]` and `proj[16]` are AE's row-vector matrices:

```cpp
float viewInv[16], projInv[16];
invertMat4(view, viewInv);   // standard 4x4 inverse, treating input as row-vector
invertMat4(proj, projInv);
// Memcpy viewInv / projInv straight into the WGSL uniform buffer.
// In WGSL: mat4x4<f32> viewInv;  // column-major reinterpretation does the rest.
```

Then in WGSL:

```wgsl
let ndc_pos     = vec4<f32>(ndc_xy, 0.0, 1.0);
let view_pos    = projInv * ndc_pos;       // unproject
let world_pos   = viewInv * vec4<f32>(view_pos.xyz / view_pos.w, 1.0);
let world_ray   = normalize(world_pos.xyz - cam_world_pos);
```

## Camera world position falls out for free

Because translation lives in row 3 of the row-vector matrix:

- **C++ side**: `cam_world_pos = (viewInv[12], viewInv[13], viewInv[14])`
  — read from row 3 of the inverted view matrix.
- **WGSL side**: after column-major load, the same bytes appear as
  column 3, so `viewInv[3].xyz` (or `(viewInv * vec4(0,0,0,1)).xyz`)
  reads the camera world position directly.

Same bytes, same information, indexed differently on each side.

## The Y-down → Y-up projection flip

AE's pixel-world is **Y-down**: comp origin is the top-left, a point
"above the comp centre" has `y < compH / 2`. NDC in WebGPU/Vulkan/Metal
is **Y-up** (`+1` at the top of the output texture). DirectX is also
Y-up post-projection.

The flip lives inside `proj.y`. The conventional perspective matrix has
`proj[5] = cot(fov/2)`; for AE input you want:

```cpp
const float cotH = 1.f / tanf(fov * 0.5f);
proj[ 0] =  cotH / aspect;
proj[ 5] = -cotH;                       // negate to flip Y-down → Y-up
proj[10] = zf / (zf - zn);
proj[11] = 1.f;                         // row-vector: w pickup from z is in row 2
proj[14] = -zn * zf / (zf - zn);
```

Ortho follows the same rule: the `1 / halfH` term in `proj[5]` is
negated to `-1 / halfH`.

If you're reconstructing rays from NDC inside a fragment / compute
shader, mirror the flip on the consumer side: `ndc.y = 1.0 - 2.0 * uv.y`
(top-of-texture is +Y in NDC, top-of-comp is `uv.y = 0`).

## flipZ — AE's "+Z toward viewer" vs. "camera looks +forward"

AE's `+Z` axis points **toward the viewer**, out of the comp plane. A
standard "camera looks along +forward" convention wants the camera's
local Z to point **into** the scene. There are two equivalent ways to
reconcile this; pick one and apply it once.

**Option A — flipZ on the camera matrix (recommended).** Negate row 2 of
the camera's local-to-world matrix. The camera's local Z axis now points
into the comp plane, and the rest of the pipeline can use a standard
"forward = +Z" convention:

```cpp
// flipZ — apply once, after AEGP_GetLayerToWorldXform on the camera layer
for (int j = 0; j < 4; j++) m[2][j] *= -1.f;
```

**Option B — `-focal` compensation in the shader.** Build rays with
`dir_local.z = -focal` instead of `+focal`, and rotate them into world
using the un-flipped camera basis. The two negations cancel.

Mixing the two is the bug. If you flipZ in C++ *and* use `-focal` in
the shader, rays advance away from the scene and the raymarch returns
all-black. Pick one.

If you later need the un-flipped "world forward" for a downstream
consumer (like a renderer that expects the original AE convention),
read row 2 with a leading minus rather than re-applying flipZ:

```cpp
float fx = -view.m[2][0], fy = -view.m[2][1], fz = -view.m[2][2];
```

## Ortho cameras are not orthonormal

AE's orthographic camera matrix has a quirk that catches you when you
try to extract an orthonormal basis from it: **the rows are not unit
length**. Row 2 (or column 2, depending on orientation) encodes the
ortho zoom factor — its magnitude can be 1.6, 0.5, anything, depending
on how the user set up the Top / Front / Left / Custom Preview view.

If you re-use the basis as-is, your shader's "forward" axis is scaled by
the zoom, and rays drift.

The fix: **re-normalize each row of the basis** before consuming it as
a view direction, and put the zoom into the projection's `1 / halfW`,
`1 / halfH` scale where it belongs:

```cpp
for (int row = 0; row < 3; row++) {
    float len = sqrtf(m[row][0]*m[row][0] + m[row][1]*m[row][1] + m[row][2]*m[row][2]);
    if (len > 1e-6f) {
        m[row][0] /= len; m[row][1] /= len; m[row][2] /= len;
    }
}
```

You can detect ortho via `AEGP_CameraSuite2::AEGP_GetCameraType` returning
`AEGP_CameraType_ORTHOGRAPHIC` and only normalize in that branch.

## Checklist when adding a new matrix-consuming pass

1. **No transpose on the way in.** AE's `A_Matrix4` is 16 row-major floats
   with translation in row 3. Memcpy directly into your uniform buffer.
2. **No transpose on the way out.** If the shader needs the inverse,
   invert the buffer in place on the C++ side. Do not transpose before
   or after — column-major reinterpretation already is a transpose.
3. **Camera world position**: C++ reads `viewInv[12..14]`; shader reads
   `viewInv[3].xyz` after column-major load. Same bytes.
4. **NDC Y flip**: AE is Y-down. Negate `proj[5]` (perspective) or
   `proj[5] = -1/halfH` (ortho). Mirror in the shader if reconstructing
   rays from `uv`.
5. **flipZ exactly once**, on the C++ side. Don't re-apply in downstream
   consumers.
6. **Ortho re-normalize**: if `AEGP_GetCameraType` says ortho, normalize
   the basis rows; let the proj matrix carry the zoom.

## Related pages

- [AE Camera Pipeline](ae-camera-pipeline.md) — the AEGP fetch sequence,
  Zoom → FOV conversion, default-camera fallback.
- [Transform Pipeline](transform-pipeline.md) — the broader 2D coordinate
  story (layer / comp / view / screen).
- [3D](3d.md) — Q&A on 3D plugin flags and `AEGP_GetEffectCameraMatrix`.
- [Lighting](../effects/lighting.md) and
  [AE Lights](../effects/ae-lights.md) — the same conventions apply to
  light matrices.
