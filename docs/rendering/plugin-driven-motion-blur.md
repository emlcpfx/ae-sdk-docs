# Plugin-Driven Motion Blur

How to make a SmartFX effect produce its own motion blur — taking multiple sub-frame samples and accumulating them — rather than relying on AE's built-in temporal sampling.

> Reference implementation: [Wrap It](https://github.com/EMLcpfx/) and [Project It](https://github.com/EMLcpfx/) (camera-projection plugins). Both use the same pattern; this page describes it generically.

---

## Table of Contents

- [Why plugin-driven motion blur?](#why-plugin-driven-motion-blur)
- [The two flags](#the-two-flags)
- [Architecture](#architecture)
- [The dispatch-lambda pattern](#the-dispatch-lambda-pattern)
- [Sub-frame time stepping](#sub-frame-time-stepping)
- [Output buffer baseline](#output-buffer-baseline)
- [Accumulation buffer](#accumulation-buffer)
- [Per-bit-depth read/write](#per-bit-depth-readwrite)
- [Mesh and camera per sample](#mesh-and-camera-per-sample)
- [Cache-key invalidation](#cache-key-invalidation)
- [Pitfalls](#pitfalls)
- [Full skeleton](#full-skeleton)

---

## Why plugin-driven motion blur?

AE's layer-level motion blur is fine for 2D layers that simply translate/rotate. It is **not** enough when:

- Your effect renders something that depends on a 3D camera and you want camera-shake motion blur baked into the output.
- Your effect projects, warps, or transforms pixels in a way that isn't a simple per-pixel transformation AE can interpolate between.
- You want shutter angle / shutter phase to apply *inside* your effect's per-frame work rather than be sampled by AE.

In all those cases AE's host-side motion blur doesn't reach the right place. You need to sample inside your `SmartRender` — re-running your render N times at sub-frame times and averaging the results.

---

## The two flags

There are two separate flags worth understanding. They're often confused.

| Flag | Where it's set | What it does |
|---|---|---|
| `PF_OutFlag_I_USE_SHUTTER_ANGLE` | `out_data->out_flags` in `GlobalSetup` | Tells AE: "I know about and use the comp's shutter angle. When the comp is in motion-blur mode and my layer has MB enabled, call my Render at sub-frame `current_time` values." |
| `PF_OutFlag2_AUTOMATIC_WIDE_TIME_INPUT` | `out_data->out_flags2` in `GlobalSetup` | Tells AE: "When I check out an input layer, give me a frame at the requested time, even if that time is between frames." Lets `checkout_layer` succeed for sub-frame times. |

Both are typically set together. The shutter-angle flag is what makes AE deliver sub-frame calls; the wide-time-input flag is what lets your sub-frame checkouts succeed without snapping to the nearest whole frame.

**Important:** Setting `PF_OutFlag_I_USE_SHUTTER_ANGLE` does **not** make AE automatically blur for you. It just makes AE call your render at sub-frame times. You still have to decide what to do with each call. The pattern below sample-and-accumulates *inside one render call*, which is independent of (and composes cleanly with) AE's per-frame shutter dispatch.

---

## Architecture

The render is structured as two layers:

1. **Inner dispatch** — the actual per-sample work: scan the comp at `current_time`, build cameras/meshes, composite projections (or whatever your effect does) onto `output_world`.
2. **Outer accumulator** — captured at the top of `SmartRender`. Reads MB params; if disabled, calls the inner dispatch once. If enabled, calls it `N` times with adjusted `current_time`, restoring an output baseline before each call and accumulating each result into a float buffer.

Putting the dispatch in a lambda lets the outer accumulator hold all the captures by reference. The accumulator never has to know what specifically the dispatch does — it just needs the output buffer to be filled with a single-frame render after each call.

---

## The dispatch-lambda pattern

```cpp
static PF_Err SmartRender(PF_InData* in_data, PF_OutData*, PF_SmartRenderExtra* extra) {
    PF_EffectWorld* input_worldP  = nullptr;
    PF_EffectWorld* output_worldP = nullptr;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    extra->cb->checkout_layer_pixels(in_data->effect_ref, PARAM_INPUT, &input_worldP);
    extra->cb->checkout_output(in_data->effect_ref, &output_worldP);

    // ... read params, init output buffer to the desired baseline ...

    auto runDispatch = [&]() {
        // EVERYTHING that depends on current_time goes inside this lambda.
        // Scan the comp, build the live camera, run the per-mesh / per-slot
        // render loop. The lambda must leave output_worldP holding a single
        // complete sub-frame render — the accumulator reads from it.
        rebuildSceneAt(in_data->current_time);
        compositeAt(in_data->current_time, output_worldP);
    };

    // ... MB outer loop calls runDispatch() once or N times ...
}
```

The crucial discipline: **anything inside `runDispatch` that reads `current_time` must re-read it on every call.** The lambda captures `in_data` by reference (because it's captured from outer scope), so changes to `in_data->current_time` between calls are visible — but only if your dispatch actually reads the current value.

---

## Sub-frame time stepping

```cpp
const A_long orig_time    = in_data->current_time;
const A_long time_step    = (in_data->time_step > 0) ? in_data->time_step : 1;
const float  shutter_frac = mb_shutter_angle_deg / 360.0f;   // e.g. 180° → 0.5
const float  phase_frac   = mb_shutter_phase_deg / 360.0f;   // e.g. -90° → -0.25

for (int s = 0; s < N; ++s) {
    float t                = ((float)s + 0.5f) / (float)N - 0.5f;   // centered in [-0.5, +0.5)
    float offset_in_frames = t * shutter_frac + phase_frac * 0.5f;
    A_long offset_in_time  = (A_long)(offset_in_frames * (float)time_step);

    in_data->current_time = orig_time + offset_in_time;
    // restore baseline + dispatch + accumulate (see below)
}

in_data->current_time = orig_time;   // ALWAYS restore. AE assumes it.
```

`time_step` is the AE-domain duration of one comp frame in `in_data->time_scale` units. Multiplying a fractional-frame offset by `time_step` gives a delta in the same units as `current_time`.

Centering `t` around zero (rather than `[0, 1)`) puts the sample distribution symmetrically around the frame center. Shutter phase shifts it. With `shutter_phase = -90°` and `shutter_angle = 180°`, samples land in `[-0.5, 0)` frames — i.e., the shutter opens half a frame before frame-center and closes at frame-center (matching AE's default).

**Always restore `in_data->current_time` to `orig_time` before returning.** Don't leave it modified — AE and downstream code in `SmartRender` (including checkin) assume the original.

---

## Output buffer baseline

Each sample must start from the same baseline before compositing. If your dispatch *composites onto* whatever's in `output_world` (rather than fully overwriting it), the second sample would composite onto the first sample's result — not what you want.

```cpp
// BEFORE the MB loop, after you've initialized output_world to its baseline
// (input plate copy, or cleared transparent — whatever your effect does):
std::vector<unsigned char> baseline((size_t)rowbytes * (size_t)H);
for (int y = 0; y < H; ++y) {
    memcpy(baseline.data() + y * rowbytes,
           (char*)output_worldP->data + y * rowbytes,
           (size_t)rowbytes);
}

// INSIDE the loop, BEFORE each runDispatch() call:
for (int y = 0; y < H; ++y) {
    memcpy((char*)output_worldP->data + y * rowbytes,
           baseline.data() + y * rowbytes,
           (size_t)rowbytes);
}
runDispatch();
// ... then accumulate output_worldP into your float buffer ...
```

If your dispatch fully overwrites `output_world` (no composite-onto), you can skip the baseline save/restore. Most effects don't, so the pattern above is the safe default.

---

## Accumulation buffer

A single float-RGBA buffer the size of `output_world`. After every sample, add the current `output_world` to the accumulator weighted by `1/N`. After the last sample, write the accumulator back to `output_world` (clamping and converting per the output's pixel format).

```cpp
std::vector<float> accum((size_t)W * (size_t)H * 4u, 0.0f);
const float weight = 1.0f / (float)N;

for (int s = 0; s < N; ++s) {
    // ... restore baseline, set current_time, runDispatch() ...
    accumulateInto(accum, output_worldP, weight);
}

writeAccumulatorTo(output_worldP, accum);
```

Floats are the natural intermediate. Don't try to accumulate in the output's native format — 8-bit will saturate way before you finish, and 16-bit precision can produce visible banding.

---

## Per-bit-depth read/write

`output_worldP->rowbytes / output_worldP->width` gives bytes per pixel: 4 = ARGB32, 8 = ARGB64, 16 = ARGB128. (Use `PF_WorldSuite2::PF_GetPixelFormat` for the authoritative answer.)

```cpp
PF_PixelFormat pf;
PF_WorldSuite2* ws = nullptr;
in_data->pica_basicP->AcquireSuite(kPFWorldSuite, kPFWorldSuiteVersion2, (const void**)&ws);
ws->PF_GetPixelFormat(output_worldP, &pf);
in_data->pica_basicP->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2);

if (pf == PF_PixelFormat_ARGB128) {
    // Read PF_PixelFloat directly into accumulator
    for (int y = 0; y < H; ++y) {
        PF_PixelFloat* row = (PF_PixelFloat*)((char*)output_worldP->data + y * rowbytes);
        for (int x = 0; x < W; ++x) {
            accum[(y*W+x)*4 + 0] += row[x].alpha * weight;
            accum[(y*W+x)*4 + 1] += row[x].red   * weight;
            accum[(y*W+x)*4 + 2] += row[x].green * weight;
            accum[(y*W+x)*4 + 3] += row[x].blue  * weight;
        }
    }
} else if (pf == PF_PixelFormat_ARGB64) {
    const float inv = 1.0f / 32768.0f;  // NOT 65535
    // ... PF_Pixel16, multiply by inv * weight ...
} else {
    const float inv = 1.0f / 255.0f;
    // ... PF_Pixel8, multiply by inv * weight ...
}
```

**16-bit gotcha:** AE's 16-bit channel max is `32768` (`PF_MAX_CHAN16`), not `65535`. Easy to miss — produces 2× wrong results.

When writing the accumulator back, clamp `[0, 1]` first to handle accumulator overshoots (rounding) and then scale back to the native max with `+0.5f` for round-half-up.

---

## Mesh and camera per sample

A common bug: setting up the camera and mesh *outside* the dispatch lambda, then expecting them to be different per sample. They won't be — they're captured by reference but constructed only once.

For motion blur on the **camera**, the dispatch lambda must re-fetch the camera at `current_time`:

```cpp
auto runDispatch = [&]() {
    NativeLayerScanner scanner;
    A_Time t_now = { in_data->current_time, in_data->time_scale };
    scanner.ScanComposition(in_data, t_now, source_layer, source_param_idx);

    NativeCameraProjector cp;
    cp.SetupFromAECamera(in_data, scanner.GetSceneLayers());
    CameraInfo live_camera = cp.GetCameraInfo();
    // ... composite using live_camera ...
};
```

For motion blur on **animated layer transforms** (a moving 3D solid, for example), the same re-scan picks up fresh transforms because the scanner reads them at `t_now`. With this in place, both camera- and mesh-driven motion blur "just work."

If you want a particular reference (like a texture being projected) to **not** blur, sample it at a fixed time — `PF_CHECKOUT_PARAM(in_data, layer_idx, fixed_ref_time, ...)` rather than `current_time`. That's how the projection plugins keep the projector layer's reference frame steady while everything else blurs.

---

## Cache-key invalidation

Plugin-driven MB makes the render's result depend on params AE doesn't natively know about (`mb_samples`, `mb_shutter_angle`, etc.). Without explicit cache invalidation, AE may serve a stale cached frame after the user changes a shutter setting.

Two complementary approaches:

1. **`PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS`** when adding the params — keeps AE's own change detection working for direct user edits.
2. **`PF_OutFlag2_I_MIX_GUID_DEPENDENCIES`** + `extra->cb->GuidMixInPtr(...)` in `SmartPreRender` — mix the MB params into the cache key explicitly. See [force-rerender.md](force-rerender.md) for the pattern.

The MB params on their own are usually enough; the GUID mixin matters more when MB *interacts* with non-param state (a scanned scene transform, an external resource).

---

## Pitfalls

- **Forgetting to restore `current_time`.** AE assumes it. Symptoms range from cache pollution to misbehavior of effects that run after yours.
- **Forgetting to restore the output baseline before each sample.** First sample renders onto the input plate; second sample renders onto the first sample's output; result darkens or saturates.
- **Re-using accumulator across SmartRender calls.** Allocate per-call; AE may call SmartRender from worker threads concurrently (under MFR), and a static accumulator would be a data race. (`std::vector<float>` allocated inside the function is per-call, safe.)
- **Accumulating in native pixel format.** 8-bit overflows before you finish. Always go to float first.
- **Hard-coding sample count.** Expose it. Different shots need different sample counts for the same shutter angle.
- **Sample count = 1.** No blur, but you pay the per-frame setup cost. The outer loop should fall through to a single dispatch when N < 2 (no save/restore, no accumulator).
- **Camera/scene captured outside the lambda.** The blur loop changes `current_time` but the camera was built once at the original time, so all samples render the same. Make sure your dispatch lambda re-fetches everything time-dependent.

---

## Full skeleton

```cpp
// In GlobalSetup:
out_data->out_flags  |= PF_OutFlag_I_USE_SHUTTER_ANGLE;
out_data->out_flags2 |= PF_OutFlag2_AUTOMATIC_WIDE_TIME_INPUT
                     |  PF_OutFlag2_I_MIX_GUID_DEPENDENCIES;

// In SmartRender:
PF_EffectWorld* output_worldP = ...;
const int W = output_worldP->width;
const int H = output_worldP->height;
const A_long rowbytes = output_worldP->rowbytes;

PF_PixelFormat pf;
// ... query pf via PF_WorldSuite2::PF_GetPixelFormat ...

// Initialize output_worldP to baseline (input plate copy or clear)
initOutputBaseline(output_worldP);

auto runDispatch = [&]() {
    // Re-fetch camera + scene at current_time, composite onto output_worldP
    renderSingleSample(in_data, output_worldP);
};

bool  mb_enabled = read PARAM_MB_ENABLE;
int   mb_samples = clamp(read PARAM_MB_SAMPLES, 1, 32);
float mb_angle   = read PARAM_MB_SHUTTER_ANGLE;
float mb_phase   = read PARAM_MB_SHUTTER_PHASE;

if (!mb_enabled || mb_samples < 2) {
    runDispatch();
} else {
    const int N = mb_samples;
    std::vector<unsigned char> baseline((size_t)rowbytes * H);
    for (int y = 0; y < H; ++y) {
        memcpy(baseline.data() + y * rowbytes,
               (char*)output_worldP->data + y * rowbytes,
               (size_t)rowbytes);
    }

    std::vector<float> accum((size_t)W * H * 4, 0.0f);
    const A_long orig_time    = in_data->current_time;
    const A_long time_step    = (in_data->time_step > 0) ? in_data->time_step : 1;
    const float  shutter_frac = mb_angle / 360.0f;
    const float  phase_frac   = mb_phase / 360.0f;
    const float  weight       = 1.0f / (float)N;

    for (int s = 0; s < N; ++s) {
        float t                = ((float)s + 0.5f) / N - 0.5f;
        float offset_in_frames = t * shutter_frac + phase_frac * 0.5f;
        in_data->current_time  = orig_time
                               + (A_long)(offset_in_frames * (float)time_step);

        for (int y = 0; y < H; ++y) {
            memcpy((char*)output_worldP->data + y * rowbytes,
                   baseline.data() + y * rowbytes,
                   (size_t)rowbytes);
        }

        runDispatch();
        accumulate(accum, output_worldP, weight, pf);
    }

    writeBack(output_worldP, accum, pf);
    in_data->current_time = orig_time;
}
```

---

*Tags: `motion-blur`, `shutter-angle`, `shutter-phase`, `pf-outflag-i-use-shutter-angle`, `automatic-wide-time-input`, `smartfx`, `sub-frame-sampling`, `accumulation`, `dispatch-lambda`, `current-time-mutation`*
