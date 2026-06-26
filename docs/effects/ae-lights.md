# AE Lights: Streams, Falloff Math, and How to Match AE's Renderer

When a plugin renders 3D content (volumetrics, custom 3D geometry,
participating media, projected textures), users expect AE's scene lights
to drive the plugin's lighting too — same direction, same intensity,
same colour, same falloff. Otherwise the plugin's output looks
disconnected from the rest of the comp.

This page documents the **fetch sequence**, **stream units**, and
**falloff math** for AE's four practical light types
(PARALLEL / POINT / SPOT / AMBIENT) — the math has been
**reverse-engineered empirically** because Adobe publishes the parameter
names but not the formulas. ENVIRONMENT lights are deferred (HDRI
sampling is its own problem).

## Light types

`AEGP_LightType` (`AE_GeneralPlug.h`):

| Type | Geometry | Use case |
|------|----------|----------|
| `PARALLEL` | infinitely distant, all rays parallel to one axis | sun |
| `SPOT` | point + cone, angle + feather | stage light |
| `POINT` | omnidirectional point, radial falloff | bulb |
| `AMBIENT` | no direction, uniform scene illumination | fill |
| `ENVIRONMENT` | image-based (HDRI), 360° skymap | studio HDRI |
| `NONE = -1` | returned for a non-light layer | — |

## Suites and PiPL flags

- `AEGP_LightSuite3` — `AEGP_GetLightType(layerH)` (AE 24.4 frozen; v2 also works on older AE).
- `AEGP_LayerSuite5` — `AEGP_GetCompNumLayers`, `AEGP_GetCompLayerByIndex`, `AEGP_GetLayerObjectType` (compare against `AEGP_ObjectType_LIGHT`), `AEGP_GetLayerToWorldXform`, `AEGP_IsVideoActive`.
- `AEGP_StreamSuite1` — `AEGP_GetLayerStreamValue` for every stream in §"Light streams" below.
- `AEGP_PFInterfaceSuite1::AEGP_GetEffectLayer` + `AEGP_LayerSuite5::AEGP_GetLayerParentComp` — gets the `CompH` from your `effect_ref`.

PiPL: set `PF_OutFlag2_I_USE_3D_LIGHTS` (and typically also
`PF_OutFlag2_I_USE_3D_CAMERA`). Without the flag, AE will not invalidate
your render when a light moves or its parameters change.

## Light streams

Units confirmed via empirical sweep against AE 2024 Classic 3D + the
`AEGP_LayerStream` enum.

| Stream | Type | Applies to | Notes |
|--------|------|------------|-------|
| `INTENSITY` | `one_d` | all | AE UI is **0-1000 %**. API returns the raw percent (e.g. 100 = 100 %). Use as `intensity_percent / 100` → multiplier 0..10. |
| `COLOR` | `color` (rgbF) | all | RGB multiplier, 0..1 float. Alpha unused. |
| `CONE_ANGLE` | `one_d` (degrees) | SPOT | full cone angle, 0..180°, default 90°. |
| `CONE_FEATHER` | `one_d` (percent) | SPOT | percentage of the cone angle. |
| `SHADOW_DARKNESS` | `one_d` (percent) | casting | 100 % = fully black shadow, 0 % = no shadow. Multiplier on shadow contribution. |
| `SHADOW_DIFFUSION` | `one_d` (pixels) | casting | soft-shadow radius in **pixels**. |
| `CASTS_SHADOWS` | `one_d` (bool) | PARALLEL/SPOT/POINT | if 0, you can skip the shadow pass entirely. |
| `LIGHT_FALLOFF_TYPE` | `one_d` (enum) | POINT/SPOT | `AEGP_LightFalloffType`: 0=NONE, 1=SMOOTH, 2=INVERSE_SQUARE_CLAMPED. **Note**: ExtendScript's `Falloff` enum is 1/2/3; SDK is 0/1/2. Always +1 when going SDK → script. |
| `LIGHT_FALLOFF_START` | `one_d` (pixels) | POINT/SPOT | "Radius" in the UI. Inside this distance the light is constant. |
| `LIGHT_FALLOFF_DISTANCE` | `one_d` (pixels) | POINT/SPOT | length of the fade zone, from start to end. |
| `SHADOW_COLOR` | `color` | all | shadow tint (default black). |

Position and orientation come from
**`AEGP_GetLayerToWorldXform`** on the light layer, returning an
`A_Matrix4` in the same conventions as the camera matrix
(see [AE Camera Matrix Conventions](../coordinates/ae-camera-matrix-conventions.md)).

**Pixels are world pixels.** AE expresses falloff and diffusion in
**comp world pixels**, the same unit as 3D layer position. If you're
respecting `in_data->downsample_x/y` for other geometry, apply the same
unscale to `LIGHT_FALLOFF_START`, `LIGHT_FALLOFF_DISTANCE`, and
`SHADOW_DIFFUSION` at fetch time.

## Light orientation: row 2 of the layer-to-world matrix

This is the answer to "which axis does the light point along, and is it
flipped or not".

Empirically: `AEGP_GetLayerToWorldXform` puts the **propagation
direction** (light → point of interest) directly in **row 2** of the
returned matrix. **No `flipZ` negation**, no sign flip — it's already
the unit vector pointing the way the light is actually shining.

```cpp
// row-vector convention, propagation direction in row 2
const float dx = mat.mat[2][0];
const float dy = mat.mat[2][1];
const float dz = mat.mat[2][2];
// (dx, dy, dz) is the unit vector from the light layer toward its POI,
// in raw AE pixel-world (no flipZ).
```

When your shader needs "**from sample point toward the light**" (the
direction Lambert's law wants), negate it once:

```wgsl
// in WGSL, given propagation dir as light_dir_prop:
let to_light_dir = -light_dir_prop;
let lambert      = max(0.0, dot(surface_normal, to_light_dir));
```

This convention does **not** match the camera convention (camera matrices
need `flipZ` on row 2 for "looking +forward"). Lights and cameras are
different geometry; treat them separately.

Per type:

| Light type | Position used? | Direction used? |
|------------|----------------|-----------------|
| PARALLEL | no (originates at infinity) | yes — only the direction |
| SPOT | yes (cone apex) | yes (cone axis) |
| POINT | yes | no |
| AMBIENT | no | no |

Empirical check: PARALLEL light at world `(960, 540, +1500)` aimed at POI
`(1960, 540, 0)`. Expected propagation direction (light → POI) is
`(+0.5547, 0, -0.832)`. Read `mat[2]`: `(+0.5547, 0, -0.832)`. ✓

## Falloff math (POINT / SPOT)

> **All formulas below are reverse-engineered empirically** from a stock
> AE 2024 solid in Classic 3D, single light, axial distances 100..1200 px,
> 8 bpc render. Numerical fits match observed pixels to within ~2 % across
> the full sample range. Adobe publishes the falloff *types* but not the
> exact math.

Shared definitions:

```
d     = length(light_pos - p)            // world pixels
start = LIGHT_FALLOFF_START
dist  = LIGHT_FALLOFF_DISTANCE
end   = start + dist
```

For all three types: `d ≤ start` returns 1 (the constant-intensity
"Radius" core).

### NONE

```
attenuation = 1
```

No distance effect.

### SMOOTH

```
if d <= start:        attenuation = 1
elif d >= end:        attenuation = 0
else:
    t = (d - start) / dist
    attenuation = 1 - smoothstep(0, 1, t)
              = 1 - (3*t² - 2*t³)
```

**Empirically verified** with `R=300, D=500`: observed sRGB byte values
at `d = 400/500/600/700` are `228/165/90/27` — match
`255 * (1 - smoothstep(t))` to within rounding.

It's the classic `3t² - 2t³` smoothstep, **not** the smoother
`6t⁵ - 15t⁴ + 10t³`.

### INVERSE_SQUARE_CLAMPED

```
if d <= start:        attenuation = 1
else:                 attenuation = (start / d)²
```

**Empirically verified** with `R=300, D=500`: observed 0..1 values at
`d = 400/500/600/800/1200` are `0.561/0.361/0.251/0.141/0.063` — match
`(R/d)²` to within ±0.002.

**Decisive finding**: `LIGHT_FALLOFF_DISTANCE` is **completely ignored**
in this mode. The inverse-square tail is left to decay asymptotically.
"Clamped" refers only to the near-clamp at `d ≤ start`.

Edge cases:

- `start = 0` (UI default for new lights): treat as `start = max(1, start)`
  to avoid divide-by-zero. Not visible to the user — at default values
  the math stays well-conditioned.
- The light **never reaches zero**, only asymptotes. Treat as "near zero"
  past `d ≈ 10 · start` (attenuation ≈ 0.01) if you want an early-out.

## Cone (SPOT)

Cone Angle is degrees, Cone Feather is a **percentage of the cone
angle**. A reasonable approximation that matches AE within a few percent:

```
half_angle  = cone_angle_deg / 2
half_feath  = cone_angle_deg * (feather_pct / 100) / 2
inner_deg   = clamp(half_angle - half_feath, 0,  90)
outer_deg   = clamp(half_angle + half_feath, 0,  90)

cos_inner   = cos(radians(inner_deg))
cos_outer   = cos(radians(outer_deg))

axis        = normalize(propagation_direction_from_row_2)
to_point    = normalize(p - light_pos)
cos_theta   = dot(to_point, axis)

cone_factor = smoothstep(cos_outer, cos_inner, cos_theta)
            // 0 outside cone+feather, 1 inside the un-feathered cone
```

The light contributes only when `cos_theta > cos_outer` (i.e., when the
sample point is inside the outer fade boundary).

**Empirical caveat**: AE's actual feather curve is slightly **asymmetric**
around the cone edge and a touch steeper than smoothstep. With
`cone_angle = 60°` and `feather = 50 %`, observed values run roughly
`20° → 0.93`, `25° → 0.58`, `30° → 0.29`, `35° → 0` — the fade band is
~18° → 33° rather than the symmetric ~22° → 38° smoothstep would predict.
The symmetric formula above is a reasonable approximation; exact match
requires a hand-fit curve.

Hard cutoff at half-angle when `feather = 0` is verified.

## How AE's stock renderer composes these

For reference, AE's Classic 3D renderer for a 2D/3D-enabled AV layer:

1. For each surface pixel, accumulate contributions from every light
   with `ACCEPTS_LIGHTS = 1` on the layer.
2. **AMBIENT**: `color * intensity * ambient_coeff`, no geometric test.
3. **PARALLEL / POINT / SPOT (diffuse)**: Lambert
   `max(0, dot(N, L)) * intensity * color * diffuse_coeff`.
4. **Specular**: Blinn-Phong when `specular_coeff > 0`. Skip for
   participating media (no shiny surface).
5. **Shadows**: if the light has `CASTS_SHADOWS = 1` and the layer has
   `ACCEPTS_SHADOWS = 1`, shadow-map and blend with `SHADOW_DARKNESS`.
6. **Falloff**: as in the formulas above, per point/spot.

You don't have to copy this — what matters is using **the same
positions, directions, colours, intensities, falloffs**. That's what
makes the plugin look "lit by the same scene" as the rest of the comp.

## Empirically verified facts

These are all sweep-verified against AE 2024 with the rig described
above:

- **Falloff Distance ignored in ISQ mode.** Setting it to 1, 500, or
  50000 produces identical pixel values.
- **Lights add linearly** in 8 bpc, no soft-clip below 100 %. Two POINT
  lights both at 30 % intensity in the same position produce 60 %
  output.
- **Downsample-invariant.** Same SMOOTH rig (R=100, D=1000, d=500)
  rendered at 1/1 and 1/4 downsample produces identical centre pixels —
  AE measures falloff distances in **world coordinates**, not screen
  pixels. **Don't rescale `LIGHT_FALLOFF_*` by downsample.**
- **8 bpc colour space.** The falloff value is written **directly** to
  the sRGB byte (no linear-to-sRGB encoding step). I.e.
  `pixel = round(255 * attenuation * lambert)`. Match this by working
  in the host project colour space, not assuming linear-RGB internally.
- **`FALLOFF_START = 0`** (UI default) is not an edge case — formula
  `1 - smoothstep((d-R)/D)` holds with `R = 0`, no special handling
  needed.
- **Intensity > 100 %** passes through linearly (multiplier > 1). Don't
  clamp inside the shader; 32-bit projects rely on HDR pass-through for
  downstream Glow / Exposure / curve grading. AE's float→int convertor
  handles the 8/16 bpc clamp on the output side.

## Fetch loop pattern

```cpp
AEGP_LayerH effectLayerH = nullptr;
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(effect_ref, &effectLayerH);

AEGP_CompH compH = nullptr;
suites.LayerSuite5()->AEGP_GetLayerParentComp(effectLayerH, &compH);

A_long numLayers = 0;
suites.LayerSuite5()->AEGP_GetCompNumLayers(compH, &numLayers);

for (A_long i = 0; i < numLayers; ++i) {
    AEGP_LayerH layerH = nullptr;
    suites.LayerSuite5()->AEGP_GetCompLayerByIndex(compH, i, &layerH);

    AEGP_ObjectType objType;
    suites.LayerSuite5()->AEGP_GetLayerObjectType(layerH, &objType);
    if (objType != AEGP_ObjectType_LIGHT) continue;

    A_Boolean videoActive = FALSE;
    suites.LayerSuite5()->AEGP_IsVideoActive(layerH, AEGP_LTimeMode_CompTime, &timeT, &videoActive);
    if (!videoActive) continue;

    AEGP_LightType lightType;
    suites.LightSuite3()->AEGP_GetLightType(layerH, &lightType);

    A_Matrix4 mat;
    suites.LayerSuite5()->AEGP_GetLayerToWorldXform(layerH, &timeT, &mat);

    // Streams: INTENSITY, COLOR, CONE_ANGLE, CONE_FEATHER,
    //          LIGHT_FALLOFF_TYPE/START/DISTANCE, CASTS_SHADOWS,
    //          SHADOW_DARKNESS, SHADOW_DIFFUSION, SHADOW_COLOR
    AEGP_StreamValue2 streamVal;
    suites.StreamSuite1()->AEGP_GetLayerStreamValue(
        layerH, AEGP_LayerStream_LIGHT_INTENSITY,
        AEGP_LTimeMode_CompTime, &timeT, FALSE, &streamVal, NULL);
    const float intensity = float(streamVal.val.one_d) / 100.f;
    suites.StreamSuite1()->AEGP_DisposeStreamValue(&streamVal);

    // ... pack into your GpuLight POD ...
    // world_pos = mat.mat[3]            (row 3 = translation)
    // world_dir_propagation = mat.mat[2]   (no negation — see "Light orientation" §)
}
```

Mix every byte of the unpacked light array into your sequence-data /
SmartFX cache key so a parameter change on any light invalidates your
render.

## Suggested GPU layout

For a portable WGSL/HLSL/GLSL/Metal layout (std140-friendly, 64 bytes
per light):

```cpp
struct GpuLight {
    float type_intensity_fallStart_fallDist[4];  // x = type code, y = intensity multiplier, z = fall_start, w = fall_distance
    float position_falloffType[4];               // xyz = world pos, w = falloff type (0/1/2)
    float direction_coneInner[4];                // xyz = world dir (unit, propagation), w = cos_inner_cone
    float color_coneOuter[4];                    // xyz = color RGB, w = cos_outer_cone
};
static_assert(sizeof(GpuLight) == 64);

struct GpuLightHeader {
    uint32_t count;
    uint32_t _pad[3];
};
```

Bind as a uniform or storage buffer with a runtime `count` so the
shader doesn't hard-code the array size. 8 lights is a comfortable
default ceiling; cost scales as `N_primary × M_shadow × K_lights` if
you do per-light shadow rays.

## Don't copy from minimal reference plugins

PrayMitive (a popular SDF-based AE sample) reads only
`INTENSITY`, `COLOR`, `LIGHT_FALLOFF_START`, and the transform — it
**does not** consume `LIGHT_FALLOFF_TYPE`, `LIGHT_FALLOFF_DISTANCE`, the
cone parameters, or the shadow streams. Its `radius := FALLOFF_START / 100`
is repurposed as a soft-shadow penumbra scale for Inigo Quilez's
`calcSoftShadows`, not as a real attenuation.

That's a deliberate first-pass sample, not an AE-accurate light model.
For production plugins that should integrate visually with stock AE
lighting, implement the real falloff and cone math above.

## Related pages

- [Lighting](lighting.md) — Q&A on AE camera+light data flowing into
  Vulkan / WebGPU / Metal.
- [AE Camera Matrix Conventions](../coordinates/ae-camera-matrix-conventions.md) —
  the row/column convention rules; same conventions apply to the
  per-light `A_Matrix4`.
- [AE Camera Pipeline](../coordinates/ae-camera-pipeline.md) — companion
  fetch sequence for camera state. Same per-comp layer enumeration
  pattern.
- [3D](../coordinates/3d.md) — Q&A on `PF_OutFlag2_I_USE_3D_LIGHTS`,
  PiPL flags, AEGP camera/light access.
