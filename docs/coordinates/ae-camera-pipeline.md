# AE Camera Pipeline: From Comp to Shader-Ready Matrices

Companion to [AE Camera Matrix Conventions](ae-camera-matrix-conventions.md),
which covers row-vector / column-vector sign rules. This page is the
**fetch sequence**: which AEGP calls produce a usable view + projection,
how to convert AE's "Zoom" stream into a proper FOV, how to handle the
case where the comp has no camera layer, and how to mirror the same
projection on the CPU so a Custom UI gizmo lands on the same pixel as
the GPU render.

## Pipeline overview

```
AE comp / camera layer
      │
      │  AEGP_PFInterfaceSuite::AEGP_GetEffectCamera     (active scene cam, or null)
      │  AEGP_LayerSuite::AEGP_GetLayerToWorldXform      (camera local-to-world)
      │  AEGP_StreamSuite::AEGP_GetLayerStreamValue(ZOOM)
      │  AEGP_CameraSuite::AEGP_GetCameraType            (persp / ortho)
      ▼
   CamData                                  (your POD)
      │ view   : Mat4   (layer-to-world, flipZ-applied — see camera-matrix-conventions.md §6)
      │ fov    : f32    (full vertical, radians)
      │ aspect : f32    (compW / compH)
      │ isOrtho: i32
      ▼
   buildViewProj(cam, compW, compH)
      │
      │ view[16] : row-major, row 3 = camera world pos
      │ proj[16] : row-major, Y-flipped for AE Y-down pixels
      ▼
   GPU uniform buffer (memcpy, no transpose)
```

## AEGP calls

All listed with the call you make and what it gives you.

| Purpose | Call | Returns |
|---------|------|---------|
| Find the active scene camera for this effect | `AEGP_PFInterfaceSuite1::AEGP_GetEffectCamera(effect_ref, &timeT, &camLayerH)` | `AEGP_LayerH` or null |
| Persp vs ortho | `AEGP_CameraSuite2::AEGP_GetCameraType(camLayerH, &camType)` | `AEGP_CameraType_PERSPECTIVE` / `_ORTHOGRAPHIC` |
| FOV via Zoom (pixels) | `AEGP_StreamSuite1::AEGP_GetLayerStreamValue(camLayerH, AEGP_LayerStream_ZOOM, ...)` | `one_d` value, in pixels |
| Camera layer transform (post-parenting) | `AEGP_LayerSuite5::AEGP_GetLayerToWorldXform(camLayerH, &timeT, &mat)` | `A_Matrix4` |
| Effect's host layer | `AEGP_PFInterfaceSuite1::AEGP_GetEffectLayer(effect_ref, &effectLayerH)` | layer the effect is applied to |
| Parent comp | `AEGP_LayerSuite5::AEGP_GetLayerParentComp(effectLayerH, &compH)` | the comp containing the effect |
| Default camera distance | `AEGP_CameraSuite2::AEGP_GetDefaultCameraDistanceToImagePlane(compH, &distD)` | virtual default cam distance, pixels |

You need the comp handle for the default-camera fallback (next section)
even if you've successfully fetched a real camera layer, because some
downstream queries need it.

## Required PiPL flags

The system only calls you with valid camera information if you've
declared interest in 3D camera state:

```
PF_OutFlag2_I_USE_3D_CAMERA      — required for camera fetch
PF_OutFlag2_I_USE_3D_LIGHTS      — required if you also fetch lights
```

Set these in your PiPL **and** mirror them in your `OutFlags2` at
`PF_Cmd_GLOBAL_SETUP`. If only the C++ flag is set and the PiPL is
stale, AE will not invalidate your render when the camera moves and
you'll see frozen camera state across keyframes.

On Windows: a stale PiPL is the most common cause of "the camera moves
but my render doesn't update". A clean rebuild forces the PiPL .rsrc
to regenerate.

## Zoom → FOV conversion

AE doesn't expose camera FOV directly. It exposes **Zoom**, which is the
pixel distance from the camera to the image plane. Convert with:

```cpp
// Full vertical FOV, radians.
cam.fov = (zoom > 0.f) ? std::atan((compH * 0.5f) / zoom) * 2.f : 1.f;
```

The 1.0 fallback is just a sane default for the degenerate `zoom = 0`
case (which AE shouldn't produce, but checking it costs nothing).

Then build the perspective projection with the standard cotangent form,
remembering the Y-flip for AE's Y-down pixels:

```cpp
const float fov  = (cam.fov > 1e-4f) ? cam.fov : 0.8f;
const float cotH = 1.f / tanf(fov * 0.5f);
const float aspect = float(compW) / float(compH);

proj[ 0] =  cotH / aspect;
proj[ 5] = -cotH;                       // AE Y-down → NDC Y-up
proj[10] = zf / (zf - zn);
proj[11] = 1.f;
proj[14] = -zn * zf / (zf - zn);
```

Reasonable `zn` / `zf` defaults for AE pixel-world: `1` / `20000` pixels.
That covers the default 1920×1080 framing where the virtual camera sits
~2666 px behind the comp plane.

## Orthographic branch

When `AEGP_GetCameraType` returns `AEGP_CameraType_ORTHOGRAPHIC`, you're
in Top / Front / Left / Custom Preview / similar. The view matrix needs
to be re-normalized (see [Camera Matrix Conventions §
"Ortho cameras are not orthonormal"](ae-camera-matrix-conventions.md#ortho-cameras-are-not-orthonormal)),
and the projection becomes a plain orthographic with `compW` / `compH`
as the frustum dimensions:

```cpp
const float halfW = 0.5f * float(compW);
const float halfH = 0.5f * float(compH);
proj[ 0] =  1.f / halfW;
proj[ 5] = -1.f / halfH;                // Y-down → NDC Y-up
proj[10] =  1.f / (zf - zn);
proj[14] = -zn / (zf - zn);
proj[15] =  1.f;
```

Zoom-aware ortho (the camera's Zoom stream actually driving the frustum
size) is a refinement that's worth doing eventually but isn't required
for baseline correctness — the comp-frame-size frustum matches what AE's
own Top/Front/Left views render when the camera is far enough that the
comp frame fills the image.

## Default synthesised camera (no camera layer in the comp)

If the user hasn't added a camera layer, `AEGP_GetEffectCamera` returns
null. AE still has a virtual default camera that the Active Camera view
uses; reconstruct it from the comp handle:

```cpp
A_FpLong distD = 0.0;
suites.CameraSuite2()->AEGP_GetDefaultCameraDistanceToImagePlane(compH, &distD);
const float dist = float(distD);

// View matrix: identity orientation, position at comp centre, distance behind plane
Mat4 m = identity4();
m.m[3][0] = compW * 0.5f;               // row 3 = translation
m.m[3][1] = compH * 0.5f;
m.m[3][2] = -dist;                       // pixels behind the comp plane
applyFlipZ(m);                          // see camera-matrix-conventions.md §6

cam.view    = m;
cam.fov     = atanf((compH * 0.5f) / dist) * 2.f;
cam.isOrtho = 0;
```

For a 1920×1080 comp, `dist` ≈ 2666 px → ~54° vertical FOV — exactly the
framing a fresh camera layer reproduces.

This fallback path is important: a lot of plugins ship with no camera
in the test comp, and rendering "no camera = no work" silently can look
like the plugin is broken when it's actually waiting for a camera that
the user never added.

## projectWorldToComp — CPU mirror for Custom UI

If you draw anything in `PF_Cmd_EVENT` (gizmos, overlays, manipulators)
that needs to land on the same pixel the GPU renders to, you need a
**CPU implementation of the same view × proj × NDC-to-pixel pipeline**:

```cpp
// world → NDC → top-origin comp pixel
A_FloatPoint projectWorldToComp(
    const float* view, const float* proj,
    int compW, int compH,
    float wx, float wy, float wz)
{
    // view * world (row-vector: vp.x = w · view[col 0])
    float vx = wx*view[ 0] + wy*view[ 4] + wz*view[ 8] + view[12];
    float vy = wx*view[ 1] + wy*view[ 5] + wz*view[ 9] + view[13];
    float vz = wx*view[ 2] + wy*view[ 6] + wz*view[10] + view[14];
    float vw = wx*view[ 3] + wy*view[ 7] + wz*view[11] + view[15];

    // proj * view_pos (same convention)
    float cx = vx*proj[ 0] + vy*proj[ 4] + vz*proj[ 8] + vw*proj[12];
    float cy = vx*proj[ 1] + vy*proj[ 5] + vz*proj[ 9] + vw*proj[13];
    float cw = vx*proj[ 3] + vy*proj[ 7] + vz*proj[11] + vw*proj[15];

    if (fabsf(cw) < 1e-6f) return { -1.f, -1.f };
    const float ndcX = cx / cw;
    const float ndcY = cy / cw;

    // NDC → comp pixel. Y is already +up post-projection (we flipped in proj[5]),
    // so to land on top-origin AE pixels: y_pixel = (0.5 - 0.5*ndcY) * compH.
    A_FloatPoint pt;
    pt.x = (0.5f + 0.5f * ndcX) * float(compW);
    pt.y = (0.5f - 0.5f * ndcY) * float(compH);
    return pt;
}
```

The exact 0.5±0.5 mapping is the second Y-flip that takes NDC `+Y is up`
back to AE's `top-of-comp is row 0`. Use this from your Custom UI draw
code to convert your gizmo's world-space anchor into the screen pixel
where it needs to render. The gizmo will then track the camera through
view switches, parented camera moves, and ortho ↔ perspective changes
without any per-view-mode special casing.

## Cache invalidation

Every byte of camera state needs to participate in your effect's cache
key:

- `view[16]` — 16 floats
- `proj[16]` — 16 floats (or just `fov`, `aspect`, `isOrtho` if you
  rebuild proj deterministically from those)
- `compW`, `compH`
- `downsample_x`, `downsample_y` (from `PF_InData`)

If you mix any of these into your `GuidMixInPtr` / sequence-data hash,
camera moves invalidate properly. Miss any of them and you'll see
stale frames during scrub.

## Related pages

- [AE Camera Matrix Conventions](ae-camera-matrix-conventions.md) — the
  row/column convention rules and inversion trap.
- [Lighting](../effects/lighting.md) and
  [AE Lights](../effects/ae-lights.md) — same fetch pattern (per-comp
  layer enumeration) for light layers.
- [3D](3d.md) — Q&A on `PF_OutFlag2_I_USE_3D_CAMERA` and
  `AEGP_GetEffectCameraMatrix`.
- [Transform Pipeline](transform-pipeline.md) — the broader 2D coordinate
  spaces.
