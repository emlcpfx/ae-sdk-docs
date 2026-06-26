# Mask Handling in SmartFX Effects

How to make a SmartFX effect respect the masks on *other* layers it samples — specifically when those layers are referenced as layer params (3D source layers, projection sources, etc.) rather than as the main input.

> This page focuses on the case where the effect needs to know which parts of a *referenced layer* are masked-in vs. masked-out. For the trivial case (masks on the effect's own input layer), see [layer-checkout-patterns.md](../layers/layer-checkout-patterns.md) — `checkout_layer_pixels(in_data->effect_ref, 0, &input)` already returns the masked input.

---

## Table of Contents

- [The problem](#the-problem)
- [Three modes, ranked](#three-modes-ranked)
- [Mode 1: None](#mode-1-none)
- [Mode 2: AEGP RenderAndCheckout](#mode-2-aegp-renderandcheckout)
- [Mode 3: Read the outlines, rasterize yourself](#mode-3-read-the-outlines-rasterize-yourself)
- [Choosing between the modes](#choosing-between-the-modes)
- [Pitfalls](#pitfalls)

---

## The problem

SmartFX renders read pixels through two checkout APIs:

| API | Returns |
|---|---|
| `PF_CHECKOUT_PARAM(in_data, idx, ...)` for a layer param | **Unmasked** raw layer pixels |
| `extra->cb->checkout_layer_pixels(in_data->effect_ref, idx, ...)` | Same raw pixels (the SmartFX equivalent of the above) |
| `extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, ...)` for `param[0]` (the auto-input) | **Masked** pixels — input layer's masks/effects applied |

So if your effect samples e.g. a 3D solid layer as a layer param (param > 0), the alpha channel you get from those APIs is the **layer's raw alpha** — no masks, no feather, no effects-stack output. Masks the user drew on that solid are invisible to you.

When you want to honor those masks (so your effect respects what the user shaped on the source layer), you have three options.

---

## Three modes, ranked

| Mode | Effort | MFR-safe | Renders | Output |
|---|---|---|---|---|
| **1. None** | None | ✅ | Anything | No mask — full-layer projection |
| **2. AEGP RenderAndCheckout** | Low | ⚠️ Limited | The fully-composited layer with its masks/effects baked in | A `PF_EffectWorld` you can sample like any layer |
| **3. Read outlines, rasterize** | High | ✅ | A binary/feathered alpha buffer derived from the mask outlines themselves | A custom alpha buffer indexed by mesh UV (or however you like) |

Each has its place. A production plugin often ships all three as a user-facing popup so users can pick based on their scene.

---

## Mode 1: None

Don't query the layer's mask state at all. Pass `nullptr` as the mask argument to your renderer. The renderer projects/samples the layer alpha as-is.

Use when:
- The user wants the full layer projected regardless of masks (e.g., they're using the source layer as a 3D placeholder and masks would just confuse things).
- You want a fast, predictable mode that always works.

No SDK code needed — just don't do the work.

---

## Mode 2: AEGP RenderAndCheckout

Ask AE to render the referenced layer for you — masks, effects, and everything baked into the alpha — then sample the resulting `PF_EffectWorld`.

### How

```cpp
AEGP_SuiteHandler suites(in_data->pica_basicP);

// 1. Resolve the layer param to an AEGP_LayerH.
AEGP_EffectRefH eff_ref = nullptr;
suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(
    S_my_id, in_data->effect_ref, &eff_ref);

AEGP_StreamRefH stream_ref = nullptr;
suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(
    S_my_id, eff_ref, ae_param_index_of_layer, &stream_ref);

A_Time layer_time = { in_data->current_time, in_data->time_scale };
AEGP_StreamValue sv = {};
suites.StreamSuite2()->AEGP_GetNewStreamValue(
    S_my_id, stream_ref, AEGP_LTimeMode_LayerTime, &layer_time, FALSE, &sv);
AEGP_LayerIDVal layer_id = sv.val.layer_id;
suites.StreamSuite2()->AEGP_DisposeStreamValue(&sv);
suites.StreamSuite2()->AEGP_DisposeStream(stream_ref);
suites.EffectSuite4()->AEGP_DisposeEffect(eff_ref);

AEGP_LayerH effect_layer = nullptr;
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &effect_layer);
AEGP_CompH comp = nullptr;
suites.LayerSuite8()->AEGP_GetLayerParentComp(effect_layer, &comp);
AEGP_LayerH source_layer = nullptr;
suites.LayerSuite8()->AEGP_GetLayerFromLayerID(comp, layer_id, &source_layer);

// 2. Configure the render options.
AEGP_LayerRenderOptionsH opts = nullptr;
suites.LayerRenderOptionsSuite2()->AEGP_NewFromLayer(S_my_id, source_layer, &opts);

A_Time comp_time = layer_time;
suites.PFInterfaceSuite1()->AEGP_ConvertEffectToCompTime(
    in_data->effect_ref, in_data->current_time, layer_time.scale, &comp_time);
suites.LayerRenderOptionsSuite2()->AEGP_SetTime(opts, comp_time);
suites.LayerRenderOptionsSuite2()->AEGP_SetMatteMode(opts, AEGP_MatteMode_STRAIGHT);

// Match world type to your output's bit depth.
AEGP_WorldType wt = output_is_32bit ? AEGP_WorldType_32
                  : output_is_16bit ? AEGP_WorldType_16
                                    : AEGP_WorldType_8;
suites.LayerRenderOptionsSuite2()->AEGP_SetWorldType(opts, wt);

// 3. Render + check out the frame.
AEGP_FrameReceiptH receipt = nullptr;
suites.RenderSuite5()->AEGP_RenderAndCheckoutLayerFrame(
    opts, nullptr, nullptr, &receipt);
suites.LayerRenderOptionsSuite2()->AEGP_Dispose(opts);

// 4. Wrap as a PF_EffectWorld your renderer can sample.
AEGP_WorldH receipt_world = nullptr;
suites.RenderSuite5()->AEGP_GetReceiptWorld(receipt, &receipt_world);
PF_EffectWorld masked_world = {};
suites.WorldSuite3()->AEGP_FillOutPFEffectWorld(receipt_world, &masked_world);

// ... your renderer reads masked_world.alpha and composites accordingly ...

// 5. ALWAYS check the receipt back in.
suites.RenderSuite5()->AEGP_CheckinFrame(receipt);
```

### Caveats

- **Threading.** `AEGP_RenderAndCheckoutLayerFrame` is **not safe to call from worker threads under MFR.** It re-enters AE's render pipeline. If your `SmartRender` is multi-threaded (`PF_OutFlag2_SUPPORTS_THREADED_RENDERING`), wrap the AEGP call in `try { ... } catch (...) { ... }` and fall back to no-mask on throw. Some plugins disable MFR while this mode is active.
- **Cost.** Re-rendering the source layer is expensive — full effect stack, full comp dependencies, full pixel buffer. A cheap effect that uses this mode pays a heavy cost per frame.
- **Sync version recommended.** There's also `AEGP_RenderAndCheckoutLayerFrame_Async`; do not use it. There's a documented memory-leak bug. See [async-render.md](async-render.md).
- **Time argument is COMP time.** `AEGP_SetTime` expects comp time, not effect/layer time. Convert with `AEGP_ConvertEffectToCompTime` if you're starting from `in_data->current_time`.

---

## Mode 3: Read the outlines, rasterize yourself

Walk the source layer's mask streams via `AEGP_MaskSuite6` + `AEGP_MaskOutlineSuite3`, snapshot each mask outline as a tesselated polyline, then rasterize the polylines yourself into a per-source alpha buffer.

### Why

- **MFR-safe.** The mask-stream reads happen *outside* the SmartRender worker threads (you do them in PreRender, or before the render dispatch fires off). The rasterization is per-source and pure-math — no AE state.
- **Fast.** The rasterized buffer is at the source layer's native resolution (or wherever you want); subsequent compositing samples it bilinearly. Much cheaper per pixel than re-rendering the layer.
- **Geometry only.** You're sampling the mask *shape*, not the layer's compositing alpha. Effects on the source layer that modify alpha (e.g. a `Set Matte` upstream) won't appear in the result. If that matters, Mode 2 is the only option.

### The extraction step

```cpp
// Walk every active source layer at current_time, snapshot each mask outline
// to a tesselated polyline. Run from PreRender (single-threaded vs. AEGP).

struct MaskOutlineSnapshot {
    int   mode;          // PF_MaskMode_ADD, _SUBTRACT, _INTERSECT, ...
    int   invert;
    float opacity;       // 0..1
    float feather_px;
    float expansion_px;
    int   num_vertices;
    std::vector<float> verts_x, verts_y;  // in layer pixel space
};

struct PerSourceMaskSet {
    int  source_slot;
    int  layer_width, layer_height;
    std::vector<MaskOutlineSnapshot> outlines;
};

// For each source layer:
A_long num_masks = 0;
suites.MaskSuite6()->AEGP_GetLayerNumMasks(source_layer, &num_masks);

for (A_long mi = 0; mi < num_masks; ++mi) {
    AEGP_MaskRefH mask = nullptr;
    suites.MaskSuite6()->AEGP_GetLayerMaskByIndex(source_layer, mi, &mask);

    PF_MaskMode mode = PF_MaskMode_NONE;
    suites.MaskSuite6()->AEGP_GetMaskMode(mask, &mode);
    if (mode == PF_MaskMode_NONE) { suites.MaskSuite6()->AEGP_DisposeMask(mask); continue; }

    A_Boolean inverted = FALSE;
    suites.MaskSuite6()->AEGP_GetMaskInvert(mask, &inverted);

    // Per-mask streams: opacity, feather, expansion are simple one_d floats.
    // Read each via AEGP_GetNewMaskStream + AEGP_GetNewStreamValue at layer time.

    // The OUTLINE stream returns an AEGP_MaskOutlineValH.
    AEGP_StreamRefH outline_stream = nullptr;
    suites.StreamSuite2()->AEGP_GetNewMaskStream(
        plugin_id, mask, AEGP_MaskStream_OUTLINE, &outline_stream);
    AEGP_StreamValue ov = {};
    suites.StreamSuite2()->AEGP_GetNewStreamValue(
        plugin_id, outline_stream, AEGP_LTimeMode_LayerTime, &t, FALSE, &ov);
    AEGP_MaskOutlineValH outlineH = ov.val.mask;

    // Tesselate each cubic Bezier segment.
    A_long num_segs = 0;
    suites.MaskOutlineSuite3()->AEGP_GetMaskOutlineNumSegments(outlineH, &num_segs);

    const int STEPS = 8;  // subdivision per segment
    for (A_long seg = 0; seg < num_segs; ++seg) {
        AEGP_MaskVertex v0, v1;
        suites.MaskOutlineSuite3()->AEGP_GetMaskOutlineVertexInfo(outlineH, seg, &v0);
        suites.MaskOutlineSuite3()->AEGP_GetMaskOutlineVertexInfo(outlineH, seg + 1, &v1);

        // PF_PathVertex / AEGP_MaskVertex: x/y are absolute layer-space coords.
        // tan_out on v0 and tan_in on v1 are RELATIVE to their vertex.
        double p0x = v0.x,                  p0y = v0.y;
        double p1x = v0.x + v0.tan_out_x,   p1y = v0.y + v0.tan_out_y;
        double p2x = v1.x + v1.tan_in_x,    p2y = v1.y + v1.tan_in_y;
        double p3x = v1.x,                  p3y = v1.y;

        for (int k = 0; k < STEPS; ++k) {
            double tt = (double)k / STEPS, mt = 1.0 - tt;
            double b0 = mt*mt*mt, b1 = 3*mt*mt*tt, b2 = 3*mt*tt*tt, b3 = tt*tt*tt;
            float x = (float)(b0*p0x + b1*p1x + b2*p2x + b3*p3x);
            float y = (float)(b0*p0y + b1*p1y + b2*p2y + b3*p3y);
            // ... store as a polyline vertex ...
        }
    }

    suites.StreamSuite2()->AEGP_DisposeStreamValue(&ov);
    suites.StreamSuite2()->AEGP_DisposeStream(outline_stream);
    suites.MaskSuite6()->AEGP_DisposeMask(mask);
}
```

The tangents are the gotcha. `AEGP_MaskVertex::tan_out_*` and `tan_in_*` are *relative* offsets from the vertex position, not absolute control points. The cubic Bezier control points for segment `[seg, seg+1]` are:

- P0 = v[seg].xy
- P1 = v[seg].xy + v[seg].tan_out
- P2 = v[seg+1].xy + v[seg+1].tan_in
- P3 = v[seg+1].xy

### The rasterization step

The snapshot is comp-time-independent (just polylines in layer space), so you can rasterize once at SmartRender entry — per source layer — and sample the resulting alpha buffer from your renderer. A point-in-polygon test per pixel works fine for sub-1k-vertex polylines.

For mask compositing semantics, walk the outlines in declaration order, applying each mask's mode (`PF_MaskMode_ADD`, `_SUBTRACT`, `_INTERSECT`, `_LIGHTEN`, `_DARKEN`, `_DIFFERENCE`) to the running alpha. Feather is a per-mask uniform radius; expansion grows/shrinks the outline. Both are approximations of AE's real (per-vertex feather, multi-stage convolution) behavior, but they're enough for most use cases.

### Threading

The extraction step uses AEGP. AEGP **is not safe to call from worker threads.** Run it from PreRender (single-threaded) and stash the result somewhere SmartRender can find it:

- In `extra->output->pre_render_data` (the canonical SmartFX path). You'll need a free callback registered if your data owns heap.
- In a sequence-data field if you're willing to use `PF_OutFlag2_MUTABLE_RENDER_SEQUENCE_DATA_SLOWER`.

Rasterization is pure math and can happen in SmartRender (per-thread, or once per-call before the per-thread work).

### Geometry-only ≠ alpha-only

Mode 3 sees mask geometry, not the layer's resulting alpha after the effect stack. If the source layer has effects upstream of yours that modify alpha (Set Matte, Levels, Track Matte, etc.), Mode 3 won't see them. Mode 2 is the only way to get the fully composited alpha.

---

## Choosing between the modes

```
              ┌── User won't draw masks on sources ───────────► Mode 1
              │
Per-frame ────┤                            ┌── Effects on source matter ─► Mode 2
              │                            │
              └── Masks DO need to be ─────┤
                  honored                  └── Just geometry needed ──► Mode 3
                                                                       (MFR-safe)
```

In practice plugins often ship all three behind a popup so users can pick the right tradeoff per shot.

---

## Pitfalls

- **Forgetting to dispose AEGP handles.** Every `AEGP_GetNew*` has a matching `AEGP_Dispose*`. Receipts MUST be checked back in via `AEGP_CheckinFrame`. Wrap in try/catch and dispose on every exit path.
- **Calling Mode 2's `AEGP_RenderAndCheckoutLayerFrame` from a render thread.** Throws "AEGP magic error (5027 :: 10)" under MFR. Catch and fall back; or disable MFR.
- **Async render.** `AEGP_RenderAndCheckoutLayerFrame_Async` leaks. Stick with the sync version (see [async-render.md](async-render.md)).
- **Setting Mode 2 time as effect time, not comp time.** `AEGP_SetTime` wants comp time. Convert with `AEGP_ConvertEffectToCompTime`.
- **Mode 3: ignoring the tangent convention.** `tan_out_x/y` are *relative*. Treating them as absolute control points gives wildly wrong Bezier curves.
- **Mode 3: forgetting the mask mode.** `PF_MaskMode_NONE` (= disabled) is a real value; skip those masks.
- **Mode 3: stashing AEGP-derived data on `extra->output->pre_render_data` without a free callback.** You'll leak on every render — or worse, double-free if AE disposes it without telling you.

---

*Tags: `masks`, `mask-mode`, `aegp-mask-suite`, `aegp-renderandcheckout`, `aegp-maskoutlinesuite`, `mask-rasterization`, `mask-outline`, `bezier-tesselation`, `smartfx`, `mfr-safety`, `layer-checkout`, `pf-maskmode`*
