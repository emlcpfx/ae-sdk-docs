# Live ECW Data: Async Upstream Frames and Live Previews

> **Full working source:** [`ECW_Histo`](https://github.com/emlcpfx/AE_ECW_Examples/tree/main/ECW_Histo) in the [AE_ECW_Examples](https://github.com/emlcpfx/AE_ECW_Examples) repo — a complete, compilable plugin (live histogram + Levels + a second live-preview canvas). See `ECW_Histo_UI.cpp` for the full async-frame request and the transient-sequence-data cache.

This is the modern (AE 13.5+) pattern for a **data-driven** custom UI in the Effect Controls Window: a control that displays something computed from the actual rendered frame — a live histogram, a vectorscope, a waveform, a thumbnail. The trick is requesting the upstream frame **asynchronously from inside `PF_Event_DRAW`** via the render async manager, computing your visualization on the UI thread, and caching the result so the panel never flickers while async renders are in flight.

This is the highest-value technique on this site for interactive scopes, because the obvious approaches (rendering synchronously in DRAW, or stashing render-thread results for the UI to read) are both broken since the AE 13.5 UI/render thread split. This page is the missing recipe.

It assumes [Custom UI: Drawing in the ECW](../guides/custom-ui-drawing.md) (the event model and Drawbot) and complements the general [Async Q&A](async.md), which covers AEGP-side async rendering but not the *in-DRAW custom-UI* case. Drawbot helpers come from the [Widget Gallery](../guides/ecw-widget-gallery.md).

## Why the old approaches no longer work

Before AE 13.5, plugins often computed UI data during render (you have the frame in hand there) and stashed it in `sequence_data` for the DRAW handler to read back. **Since AE 13.5 the UI thread and render threads are separate**, and with `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`:

- **Render-thread writes to `sequence_data` for UI are gone.** Render runs on worker threads with read-only sequence data; you cannot safely hand UI state from a render call to a draw call through it.
- **Rendering synchronously from the UI thread can deadlock.** You must not call blocking render functions inside `PF_Event_DRAW`.

The supported replacement: inside DRAW, ask the **render async manager** for the frame you need. If it is already cached, you get it immediately; if not, the manager schedules a render and **re-fires `PF_Event_DRAW`** when it is ready. Either way you draw the *last good* cached result so the panel stays stable.

## Enabling the async manager

Set `PF_OutFlag2_CUSTOM_UI_ASYNC_MANAGER` in `GlobalSetup` (and the matching bit in the PiPL), and register with AEGP so you can build render options against your own effect:

```c
// GlobalSetup
out_data->out_flags  |= PF_OutFlag_CUSTOM_UI | PF_OutFlag_DEEP_COLOR_AWARE | /* ... */;
out_data->out_flags2 |= PF_OutFlag2_SUPPORTS_SMART_RENDER
                      | PF_OutFlag2_FLOAT_COLOR_AWARE
                      | PF_OutFlag2_CUSTOM_UI_ASYNC_MANAGER     // <-- async frame request in DRAW
                      | PF_OutFlag2_SUPPORTS_THREADED_RENDERING;

// Register so the custom UI can request the upstream frame. Premiere lacks the AEGP suites.
if (in_data->appl_id != 'PrMr') {
    AEFX_SuiteScoper<AEGP_UtilitySuite3> u_suite(in_data, kAEGPUtilitySuite, kAEGPUtilitySuiteVersion3);
    ERR(u_suite->AEGP_RegisterWithAEGP(NULL, NAME, &G_my_plugin_id));   // stores into a global plugin id
}
```

In the PiPL, the corresponding `AE_Effect_Global_OutFlags_2` bit is `PF_OutFlag2_CUSTOM_UI_ASYNC_MANAGER (1 << 24)`. The flags **must** match `out_data->out_flags2` exactly or AE will not enable the feature.

## Requesting the upstream frame inside DRAW

This is the core call. Five suites cooperate:

1. `AEGP_PFInterfaceSuite1` — gets an `AEGP_EffectRefH` for the running effect.
2. `AEGP_LayerRenderOptionsSuite1` — builds render options for the layer **upstream of this effect** (i.e. the input to your effect) and lets you downsample.
3. `PF_EffectCustomUISuite2` — gives you the per-context `PF_AsyncManagerP`.
4. `AEGP_RenderAsyncManagerSuite1` — the actual checkout-or-render call.
5. (Later, in the draw) `AEGP_RenderSuite4` / `AEGP_WorldSuite3` — pull the world out of the receipt.

```c
static PF_Err
RequestAsyncFrameForPreview(PF_InData *in_data, PF_OutData *out_data,
                            PF_EventExtra *event_extra,
                            AEGP_FrameReceiptH *frame_receiptPH)
{
    PF_Err err = PF_Err_NONE;

    AEFX_SuiteScoper<AEGP_EffectSuite3>             e_suite  (in_data, kAEGPEffectSuite,             kAEGPEffectSuiteVersion3);
    AEFX_SuiteScoper<AEGP_LayerRenderOptionsSuite1> lro_suite(in_data, kAEGPLayerRenderOptionsSuite, kAEGPLayerRenderOptionsSuiteVersion1);
    AEFX_SuiteScoper<AEGP_PFInterfaceSuite1>        pf_suite (in_data, kAEGPPFInterfaceSuite,        kAEGPPFInterfaceSuiteVersion1);
    AEFX_SuiteScoper<PF_EffectCustomUISuite2>       pfcu_suite(in_data, kPFEffectCustomUISuite,      kPFEffectCustomUISuiteVersion2);
    AEFX_SuiteScoper<AEGP_RenderAsyncManagerSuite1> ram_suite(in_data, kAEGPRenderAsyncManagerSuite, kAEGPRenderAsyncManagerSuiteVersion1);

    AEGP_EffectRefH          effectH     = NULL;
    AEGP_LayerRenderOptionsH layer_ropsH = NULL;

    // Build render options for the frame feeding INTO this effect.
    ERR(pf_suite->AEGP_GetNewEffectForEffect(G_my_plugin_id, in_data->effect_ref, &effectH));
    ERR(lro_suite->AEGP_NewFromUpstreamOfEffect(G_my_plugin_id, effectH, &layer_ropsH));
    ERR(lro_suite->AEGP_SetDownsampleFactor(layer_ropsH, PREVIEW_DOWNSAMPLE, PREVIEW_DOWNSAMPLE));

    // The async manager is owned by AE - do NOT dispose it.
    PF_AsyncManagerP async_mgrP = NULL;
    ERR(pfcu_suite->PF_GetContextAsyncManager(in_data, event_extra, &async_mgrP));

    // Checkout if cached, else schedule a render. PURPOSE id lets the manager
    // group/cancel stale requests of the same kind.
    #define PURPOSE_HISTO 1
    ERR(ram_suite->AEGP_CheckoutOrRender_LayerFrame_AsyncManager(
            async_mgrP, PURPOSE_HISTO, layer_ropsH, frame_receiptPH));

    if (effectH)     e_suite->AEGP_DisposeEffect(effectH);   // dispose what you created
    if (layer_ropsH) lro_suite->AEGP_Dispose(layer_ropsH);
    return err;
}
```

Key points:

- **`AEGP_NewFromUpstreamOfEffect`** targets the input to *your* effect — exactly what a scope wants to analyze (the footage before your adjustment). Use `AEGP_NewFromDownstreamOfEffect` for the post-effect result instead.
- **Downsample aggressively.** A histogram or thumbnail does not need full resolution. `AEGP_SetDownsampleFactor(opts, 8, 8)` renders at 1/8 size — fast, and bins/thumbnails are statistically fine.
- **The `purpose` integer** is a request identity. The manager uses it to coalesce and cancel superseded requests of the same purpose automatically — you do not track request IDs yourself.
- **Lifetime:** you own `effectH` and `layer_ropsH` (dispose them). You do **not** own the `PF_AsyncManagerP` (it is per-context, managed by AE). The frame receipt is checked back in after you read it (below).
- **`PF_GetContextAsyncManager`** is on `PF_EffectCustomUISuite2` (version 2), not version 1.

## Reading the receipt and caching the result

Back in the draw handler: lock your transient cache, request the frame, and if a world comes back, compute fresh and copy it into the cache. If the receipt is empty (render still pending), you skip the recompute and draw the last cached value.

```c
ECW_SeqData *seqP = in_data->sequence_data
    ? reinterpret_cast<ECW_SeqData*>(PF_LOCK_HANDLE(in_data->sequence_data)) : NULL;

if (!err && seqP) {
    AEGP_FrameReceiptH receiptH = NULL;
    ERR(RequestAsyncFrameForPreview(in_data, out_data, event_extra, &receiptH));
    if (!err && receiptH) {
        AEFX_SuiteScoper<AEGP_RenderSuite4> r_suite(in_data, kAEGPRenderSuite, kAEGPRenderSuiteVersion4);
        AEGP_WorldH worldH = NULL;
        ERR(r_suite->AEGP_GetReceiptWorld(receiptH, &worldH));
        if (!err && worldH) {                       // receipt can be valid but EMPTY (render pending)
            PF_EffectWorld world;
            AEFX_SuiteScoper<AEGP_WorldSuite3> w_suite(in_data, kAEGPWorldSuite, kAEGPWorldSuiteVersion3);
            ERR(w_suite->AEGP_FillOutPFEffectWorld(worldH, &world));   // AEGP world -> familiar PF_EffectWorld

            ECW_SeqData fresh; AEFX_CLR_STRUCT(fresh); fresh.magic = ECW_HISTO_MAGIC;
            ERR(ComputeHistogramFromFrame(in_data, &world, channel, &fresh));
            if (!err) {                              // commit only a fully-computed result
                memcpy(seqP->bins, fresh.bins, sizeof(seqP->bins));
                seqP->peak   = fresh.peak;
                seqP->validB = TRUE;
            }
        }
        r_suite->AEGP_CheckinFrame(receiptH);        // always check the receipt back in
    }
}
// ... draw seqP (the cached histogram) below, whether or not we recomputed this pass ...
if (seqP) PF_UNLOCK_HANDLE(in_data->sequence_data);
```

`AEGP_FillOutPFEffectWorld` converts the AEGP `AEGP_WorldH` into a `PF_EffectWorld`, so your pixel code can use the same `worldP->data` / `worldP->rowbytes` / `PF_GetPixelFormat` access it uses everywhere else.

> **Always `AEGP_CheckinFrame` the receipt**, even on the empty-world path, or you leak the render. Pair the request and check-in in the same draw pass.

## Caching in transient sequence data (anti-flicker)

The cache must be **transient sequence data** — allocated in `SEQUENCE_SETUP`, recomputed on demand, never persisted. Tag it with a magic number so you can sanity-check the handle, and a `validB` flag so the first draw (before any frame has arrived) draws blank instead of garbage.

```c
#define ECW_HISTO_MAGIC '...'
typedef struct {
    A_long     magic;
    PF_Boolean validB;                            // false until the first successful compute
    A_u_long   bins[HISTO_CHANNELS][HISTO_BINS];  // the cached visualization
    A_u_long   peak;                              // display scale
} ECW_SeqData;

// SEQUENCE_SETUP / SEQUENCE_RESETUP
static PF_Err SequenceSetup(PF_InData *in_data, PF_OutData *out_data)
{
    PF_Err err = PF_Err_NONE;
    if (in_data->sequence_data) { PF_DISPOSE_HANDLE(in_data->sequence_data); out_data->sequence_data = NULL; }
    PF_Handle seqH = PF_NEW_HANDLE(sizeof(ECW_SeqData));
    if (seqH) {
        ECW_SeqData *s = reinterpret_cast<ECW_SeqData*>(PF_LOCK_HANDLE(seqH));
        if (s) { AEFX_CLR_STRUCT(*s); s->magic = ECW_HISTO_MAGIC; s->validB = FALSE;
                 in_data->sequence_data = out_data->sequence_data = seqH; PF_UNLOCK_HANDLE(seqH); }
    } else err = PF_Err_OUT_OF_MEMORY;
    return err;
}

// SEQUENCE_FLATTEN: transient cache, nothing meaningful to persist - just hand it back.
static PF_Err SequenceFlatten(PF_InData *in_data, PF_OutData *out_data)
{
    if (in_data->sequence_data) out_data->sequence_data = in_data->sequence_data;
    return PF_Err_NONE;
}
```

Why this is what stops the flicker: the async manager re-fires DRAW several times for one logical frame (request scheduled, then ready). Without a cache, the intermediate draws (empty world) would blank the panel and it would strobe. By **only overwriting the cache when a full compute succeeds** and **always drawing the cache**, the panel shows the previous good visualization until the new one is ready, then updates in one clean step.

## Computing the visualization from the world

Once you have a `PF_EffectWorld`, it is ordinary pixel iteration — handle all three depths (the async frame can come back 8/16/32 depending on the project), normalize to 0..1, and bin. Use `rowbytes` for row stepping, never `width * pixelsize`.

```c
PF_Err
ComputeHistogramFromFrame(PF_InData *in_data, PF_EffectWorld *worldP,
                          A_long shown_channel, ECW_SeqData *seqP)
{
    PF_Err err = PF_Err_NONE;
    if (!seqP) return PF_Err_BAD_CALLBACK_PARAM;
    AEFX_CLR_STRUCT(seqP->bins); seqP->peak = 0;
    if (!worldP || !worldP->data) return err;

    AEFX_SuiteScoper<PF_WorldSuite2> wsP(in_data, kPFWorldSuite, kPFWorldSuiteVersion2);
    PF_PixelFormat fmt = PF_PixelFormat_INVALID;
    ERR(wsP->PF_GetPixelFormat(worldP, &fmt));

    for (A_long y = 0; !err && y < worldP->height; ++y) {
        char *row = (char*)worldP->data + (size_t)worldP->rowbytes * y;   // rowbytes!
        for (A_long x = 0; x < worldP->width; ++x) {
            switch (fmt) {
                case PF_PixelFormat_ARGB128: { PF_PixelFloat *p = (PF_PixelFloat*)row + x; BinPixel(seqP, p->red, p->green, p->blue); break; }
                case PF_PixelFormat_ARGB64:  { PF_Pixel16   *p = (PF_Pixel16*)row + x;   BinPixel(seqP, p->red/(float)PF_MAX_CHAN16, p->green/(float)PF_MAX_CHAN16, p->blue/(float)PF_MAX_CHAN16); break; }
                case PF_PixelFormat_ARGB32:  { PF_Pixel8    *p = (PF_Pixel8*)row + x;    BinPixel(seqP, p->red/(float)PF_MAX_CHAN8,  p->green/(float)PF_MAX_CHAN8,  p->blue/(float)PF_MAX_CHAN8);  break; }
                default: err = PF_Err_BAD_CALLBACK_PARAM; break;
            }
        }
    }
    // Display scale = peak of the shown channel, ignoring the pure black/white
    // end spikes so the midtones stay readable.
    const A_u_long *shown = seqP->bins[ShownBinSet(shown_channel)];
    for (int i = 1; i < HISTO_BINS - 1; ++i) if (shown[i] > seqP->peak) seqP->peak = shown[i];
    if (seqP->peak == 0) seqP->peak = 1;
    return err;
}
```

## Drawing the histogram with overlay markers

Draw the cached bins as filled vertical rects, then overlay other parameters as markers — here, Levels Input Black/White/Gamma as full-height lines and Output ticks below. A `sqrt` scaling keeps small counts visible (a standard scope convention):

```c
float peakF = (float)seqP->peak;
for (int i = 0; !err && i < HISTO_BINS; ++i) {
    float norm = sqrtf((float)seqP->bins[set][i] / peakF);   // sqrt: lift small counts
    if (norm > 1.f) norm = 1.f;
    float barH = norm * canvasH;
    if (barH < 0.5f) continue;
    DRAWBOT_RectF32 r = { canvasL + (float)i, canvasT + (canvasH - barH), 1.0f, barH };
    // ... NewPath + AddRect + FillPath with the channel-colored brush ...
}

// Levels markers read live from the sliders and draw over the histogram.
float inB = (float)params[PARAM_IN_BLACK]->u.fs_d.value / 255.f;
FillVLine(db, sup, surf, canvasL + inB * canvasW, canvasT, canvasH, 1.5f, &black);
// ... inW (white), gamma handle, output ticks ...
```

Reading the slider params directly in DRAW (no checkout needed for the current value during a UI event) makes the markers track the controls instantly as the user scrubs them.

## A second canvas: a procedural live preview sharing the render generator

The same effect can carry a *second* custom-UI canvas that previews something the render generates — for example a turbulence-noise thumbnail. This needs **no async frame** (the generator is procedural), and its highest-value idea is the same as the curve/gradient editors: **the preview and the render call one shared function**, so the thumbnail is exactly what the footage will get.

Put the generator in the header as `static inline` so both `.cpp` files share it:

```c
typedef struct { float amount; float invScale; int octaves; float evoX, evoY; } NoiseParams;

// Shared by render AND preview. lx,ly are LAYER px so feature size is resolution-independent.
static inline float TurbNoise(float lx, float ly, const NoiseParams *N)
{
    float x = lx * N->invScale + N->evoX, y = ly * N->invScale + N->evoY;
    float sum = 0.f, amp = 1.f, norm = 0.f;
    int oct = N->octaves < 1 ? 1 : (N->octaves > 8 ? 8 : N->octaves);
    for (int o = 0; o < oct; ++o) {
        sum  += amp * fabsf(2.f * ECW_vnoise(x, y) - 1.f);   // ECW_vnoise: stateless value noise
        norm += amp; x *= 2.f; y *= 2.f; amp *= 0.5f;
    }
    return norm > 0.f ? sum / norm : 0.f;
}
```

The preview canvas samples `TurbNoise` per pixel into a buffer and blits it once with `NewImageFromBuffer` — one `DrawImage` stays smooth while dragging, and `24RGB` gives a clean 256-level grayscale (no posterization, no premultiply ambiguity, and avoids the AE 2025 multi-image bug by using a single image per context):

```c
int SZ = UI_NOISE_SIZE;
float pxToLayer = NOISE_PREVIEW_SPAN / (float)SZ;          // map preview px -> layer px window
std::vector<unsigned char> pix((size_t)SZ * SZ * 3);
for (int py = 0; py < SZ; ++py) {
    float ly = ((float)py + 0.5f) * pxToLayer;
    for (int px = 0; px < SZ; ++px) {
        float lx = ((float)px + 0.5f) * pxToLayer;
        unsigned char v = (unsigned char)(TurbNoise(lx, ly, &N) * 255.f + 0.5f);
        size_t i = ((size_t)py * SZ + px) * 3;
        pix[i] = pix[i + 1] = pix[i + 2] = v;
    }
}
DRAWBOT_ImageRef img = NULL;
ERR(db->supplier_suiteP->NewImageFromBuffer(supplier_ref, SZ, SZ, SZ * 3, kDRAWBOT_PixelLayout_24RGB, pix.data(), &img));
DRAWBOT_PointF32 origin = { L, T };
ERR(db->surface_suiteP->DrawImage(surface_ref, img, &origin, 1.0f));
if (img) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)img));
```

The two canvases coexist via `effect_win.index` dispatch (see the [Widget Gallery](../guides/ecw-widget-gallery.md#routing-events-to-the-right-widget-effect_winindex)):

```c
switch (event_extra->effect_win.index) {
    case PARAM_HISTO_UI: ERR(DrawHistogram  (in_data, out_data, params, event_extra, &db, sup, surf)); break;  // async frame
    case PARAM_NOISE_UI: ERR(DrawNoisePreview(in_data, out_data, params, event_extra, &db, sup, surf)); break; // procedural
    default: break;
}
```

The render reads the same params (mapping output-buffer pixels back to layer px via `in_data->downsample_*` and `output_origin_*` so feature size matches the full-res preview) and calls the identical `TurbNoise`. Preview and output cannot drift.

## Matching the render to the preview coordinate space

For a procedural preview to match the render, both must evaluate the generator in the **same coordinate space**. The preview samples in layer pixels directly; the render must convert its output-buffer coordinates back to full-res layer pixels:

```c
// In the render refcon setup:
R.coordMulX = (float)in_data->downsample_x.den / (float)(in_data->downsample_x.num ? in_data->downsample_x.num : 1);
R.coordMulY = (float)in_data->downsample_y.den / (float)(in_data->downsample_y.num ? in_data->downsample_y.num : 1);
R.originX   = (float)in_data->output_origin_x;
R.originY   = (float)in_data->output_origin_y;

// Per pixel (x,y in output buffer):
float lx = ((float)x + R.originX) * R.coordMulX;   // -> full-res layer px
float ly = ((float)y + R.originY) * R.coordMulY;
float n  = TurbNoise(lx, ly, &R.N);
```

Without this mapping, the pattern's feature size and offset would change with resolution/region-of-interest and the preview would not match the rendered footage.

## Pitfalls

1. **Forgetting `PF_OutFlag2_CUSTOM_UI_ASYNC_MANAGER`** (or a PiPL/`out_flags2` mismatch) — `PF_GetContextAsyncManager` returns nothing and DRAW never gets a frame.
2. **Not registering with AEGP** (`AEGP_RegisterWithAEGP` into your global plugin id) — `AEGP_GetNewEffectForEffect` and friends need it. Skip on Premiere (`appl_id == 'PrMr'`).
3. **Treating an empty receipt as an error.** A valid receipt with a NULL world means the render is pending — draw the cache and try again next DRAW. Only recompute when a world is actually returned.
4. **Leaking the receipt** — always `AEGP_CheckinFrame`. Also dispose the `effectH` and `layer_ropsH` you create; do **not** dispose the `PF_AsyncManagerP`.
5. **Persisting the cache.** Keep it transient — recompute from the async frame. Flatten just hands the handle back; nothing in it is meaningful across save/load.
6. **Computing at full resolution.** Always downsample (`AEGP_SetDownsampleFactor`); a scope or thumbnail does not need every pixel and full-res renders stall the UI.
7. **Synchronous rendering in DRAW.** Never call a blocking render from the UI thread in AE 13.5+ — it can deadlock. The async manager is the only safe path.
8. **Preview/render drift.** Share one generator function (and one coordinate convention) between preview and render — the whole point of a live preview is that it is truthful.

## Related reading

- [Custom UI: Drawing in the ECW](../guides/custom-ui-drawing.md) — event model, the brief async-manager mention this page expands on.
- [Async (Q&A)](async.md) — AEGP-side async frame rendering on the idle hook (a different context from in-DRAW custom UI).
- [ECW Widget Gallery](../guides/ecw-widget-gallery.md) — `NewImageFromBuffer`, multi-canvas dispatch, Drawbot helpers.
- [ECW Curve / Response Editor](../guides/ecw-curve-editor.md) and [ECW Gradient Editor](../guides/ecw-gradient-editor.md) — the shared-evaluator-between-UI-and-render principle, for arb-backed editors.
