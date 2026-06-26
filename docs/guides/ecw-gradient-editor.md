# ECW Multi-Stop Gradient Editor

This guide builds a multi-stop **color gradient editor** in the Effect Controls Window, like the gradient widgets in EmberGen, Houdini, or AE's own Gradient Map: a per-pixel gradient bar, triangle handles per stop, drag-to-move clamped between neighbours, double-click to open the host color picker, click-the-bar-to-add (sampling the ramp so it does not jump), Alt-click or drag-out to delete, all persisted as a variable-length stop list in arb data with undo/redo, and applied at render time as an MFR-safe gradient-map across 8/16/32-bit.

It assumes [Custom Parameter UI](custom-param-ui.md) (arb-data callbacks) and [Custom UI: Drawing in the ECW](custom-ui-drawing.md) (events + Drawbot). It shares its structure with the [Curve Editor](ecw-curve-editor.md) — read that first if you want the simpler arb-handle-drag pattern; this page adds color, the host color picker, and the gradient-map render. Drawbot helpers come from the [Widget Gallery](ecw-widget-gallery.md).

## Data model: a fixed-capacity stop list

The gradient is a fixed-capacity **POD** array of stops, so arb flatten/copy/compare are `memcpy` and field compares. A variable-length list (`count <= GRAD_MAX_STOPS`) lives inside a fixed struct — the same trick the curve editor uses.

```c
#define GRAD_MAX_STOPS  16
#define GRAD_MIN_STOPS  2

typedef struct {
    float pos;          // 0..1 position along the bar
    float r, g, b;      // 0..1 color
} GradStop;

typedef struct {
    A_long   version;
    A_long   count;                  // GRAD_MIN_STOPS .. GRAD_MAX_STOPS
    A_long   selected;               // UI selection (index, or -1) - UI state only
    GradStop stops[GRAD_MAX_STOPS];  // kept sorted ascending by pos
} GradientArb;
```

Add it as an arb canvas param (see [Custom Parameter UI](custom-param-ui.md#adding-an-arbitrary-data-parameter)) sized for the bar plus the handle strip below it:

```c
AEFX_CLR_STRUCT(def);
ERR(CreateDefaultArb(in_data, out_data, &def.u.arb_d.dephault));
PF_ADD_ARBITRARY2("Gradient",
                  UI_GRAD_WIDTH  + 2 * UI_GRAD_GUTTER,
                  UI_GRAD_HEIGHT + 2 * UI_GRAD_GUTTER, 0,
                  PF_PUI_CONTROL | PF_PUI_DONT_ERASE_CONTROL,
                  def.u.arb_d.dephault, GRAD_UI_DISK_ID, GRAD_ARB_REFCON);
```

The default is a five-stop ramp (here a fire palette: black to dark-red to orange to yellow to white-hot) so the control reads as a gradient on first apply.

## The shared evaluator

As with the curve editor, **one function samples the gradient**, used by the bar drawing, the add-stop color sampling, and the render. Put it in the header so both `.cpp` files share it; this is what keeps the on-screen bar and the rendered output identical. It is a plain linear interpolation between the two bracketing stops, with the ends held flat:

```c
static inline void SampleGradient(const GradientArb *gP, float t,
                                  float *rO, float *gO, float *bO)
{
    int n = gP->count;
    if (n <= 0) { *rO = *gO = *bO = 0.f; return; }
    if (t <= gP->stops[0].pos)     { *rO = gP->stops[0].r;     *gO = gP->stops[0].g;     *bO = gP->stops[0].b;     return; }
    if (t >= gP->stops[n-1].pos)   { *rO = gP->stops[n-1].r;   *gO = gP->stops[n-1].g;   *bO = gP->stops[n-1].b;   return; }
    for (int i = 1; i < n; ++i) {
        if (t <= gP->stops[i].pos) {
            const GradStop *a = &gP->stops[i-1];
            const GradStop *b = &gP->stops[i];
            float span = b->pos - a->pos;
            float f = (span > 1e-6f) ? (t - a->pos) / span : 0.f;
            *rO = a->r + (b->r - a->r) * f;
            *gO = a->g + (b->g - a->g) * f;
            *bO = a->b + (b->b - a->b) * f;
            return;
        }
    }
    *rO = gP->stops[n-1].r; *gO = gP->stops[n-1].g; *bO = gP->stops[n-1].b;
}
```

## Layout helpers

Derive the bar rect and the handle strip from the on-screen control frame (frame coords; Drawbot pixel centers at +0.5). `StopX` maps a stop position to a screen x; `HitHandle` and `InBar` are the hit tests.

```c
typedef struct {
    float barL, barT, barW, barH;   // gradient bar rect
    float handTopY, handBotY;       // handle triangle apex (at bar) / base (below)
} GradLayout;

static void ComputeLayout(const PF_Rect *cf, GradLayout *L)
{
    L->barL = (float)cf->left + UI_GRAD_GUTTER + 0.5f;
    L->barT = (float)cf->top  + UI_GRAD_GUTTER + 0.5f;
    L->barW = (float)UI_GRAD_WIDTH;
    L->barH = (float)GRAD_BAR_H;
    L->handTopY = L->barT + L->barH;
    L->handBotY = L->handTopY + (float)GRAD_HANDLE_H;
}

static inline float StopX(const GradLayout *L, float pos) { return L->barL + pos * L->barW; }

static int HitHandle(const GradLayout *L, const GradientArb *g, A_long mx, A_long my)
{
    if ((float)my < L->handTopY - 2.f || (float)my > L->handBotY + 2.f) return -1;  // wrong row
    for (int i = 0; i < g->count; ++i)
        if (fabsf((float)mx - StopX(L, g->stops[i].pos)) <= GRAD_HANDLE_HALF + 2.f) return i;
    return -1;
}

static inline PF_Boolean InBar(const GradLayout *L, A_long mx, A_long my)
{
    return (float)mx >= L->barL && (float)mx <= L->barL + L->barW &&
           (float)my >= L->barT && (float)my <= L->barT + L->barH;
}
```

## Drawing: per-pixel bar + triangle handles

Drawbot has no gradient primitive, so the bar is drawn as one 1px-wide filled strip per column, each colored by `SampleGradient`. (For a wider bar, build an RGB buffer and use `NewImageFromBuffer` instead — see the [Widget Gallery image blit](ecw-widget-gallery.md#widget-image-blit-with-newimagefrombuffer). At 256px wide, strips are fine.) Each stop gets a filled triangle handle whose fill *is* the stop color, with a black outline (white and thicker when selected).

```c
GradientArb *g = reinterpret_cast<GradientArb*>(PF_LOCK_HANDLE(params[GRAD_UI]->u.arb_d.value));
if (!err && g) {
    // Bar: one 1px strip per column.
    for (int px = 0; !err && px < UI_GRAD_WIDTH; ++px) {
        float t = (float)px / (float)(UI_GRAD_WIDTH - 1);
        float r, gr, b; SampleGradient(g, t, &r, &gr, &b);
        DRAWBOT_ColorRGBA col = { r, gr, b, 1.0f };
        DRAWBOT_BrushRef brush = NULL; DRAWBOT_PathRef path = NULL;
        DRAWBOT_RectF32 strip = { L.barL + (float)px, L.barT, 1.0f, L.barH };
        ERR(db.supplier_suiteP->NewBrush(supplier, &col, &brush));
        ERR(db.supplier_suiteP->NewPath(supplier, &path));
        ERR(db.path_suiteP->AddRect(path, &strip));
        ERR(db.surface_suiteP->FillPath(surface, brush, path, kDRAWBOT_FillType_Default));
        if (path)  ERR2(db.supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)path));
        if (brush) ERR2(db.supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)brush));
    }

    // Stop handles: filled triangle (apex up at the bar), color = the stop color.
    for (int i = 0; !err && i < g->count; ++i) {
        float x = StopX(&L, g->stops[i].pos);
        DRAWBOT_ColorRGBA fill = { clampf(g->stops[i].r,0,1), clampf(g->stops[i].g,0,1), clampf(g->stops[i].b,0,1), 1.0f };
        PF_Boolean sel = (i == g->selected);
        DRAWBOT_ColorRGBA edge = sel ? DRAWBOT_ColorRGBA{1,1,1,1} : DRAWBOT_ColorRGBA{0,0,0,1};

        DRAWBOT_PathRef path = NULL; DRAWBOT_BrushRef brush = NULL; DRAWBOT_PenRef pen = NULL;
        ERR(db.supplier_suiteP->NewPath(supplier, &path));
        ERR(db.path_suiteP->MoveTo(path, x, L.handTopY));
        ERR(db.path_suiteP->LineTo(path, x - GRAD_HANDLE_HALF, L.handBotY));
        ERR(db.path_suiteP->LineTo(path, x + GRAD_HANDLE_HALF, L.handBotY));
        ERR(db.path_suiteP->Close(path));
        ERR(db.supplier_suiteP->NewBrush(supplier, &fill, &brush));
        ERR(db.surface_suiteP->FillPath(surface, brush, path, kDRAWBOT_FillType_Default));
        ERR(db.supplier_suiteP->NewPen(supplier, &edge, sel ? 2.0f : 1.0f, &pen));
        ERR(db.surface_suiteP->StrokePath(surface, pen, path));
        if (pen)   ERR2(db.supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)pen));
        if (brush) ERR2(db.supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)brush));
        if (path)  ERR2(db.supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)path));
    }
}
if (g) PF_UNLOCK_HANDLE(params[GRAD_UI]->u.arb_d.value);
```

## Stop-list edits

Insert keeps the list sorted; delete protects the minimum count:

```c
static int InsertStop(GradientArb *g, float pos, float r, float gg, float b)
{
    if (g->count >= GRAD_MAX_STOPS) return -1;
    int i = 0;
    while (i < g->count && g->stops[i].pos < pos) ++i;
    for (int k = g->count; k > i; --k) g->stops[k] = g->stops[k - 1];
    g->stops[i] = { pos, r, gg, b };
    g->count++;
    return i;
}

static void DeleteStop(GradientArb *g, int i)
{
    if (g->count <= GRAD_MIN_STOPS || i < 0 || i >= g->count) return;
    for (int k = i; k < g->count - 1; ++k) g->stops[k] = g->stops[k + 1];
    g->count--;
}
```

## Click: select, add, delete, and the color picker

The click handler reads `num_clicks` and `modifiers` to disambiguate four gestures on one canvas:

- **Double-click a handle** -> open the host color picker for that stop.
- **Alt-click a handle** -> delete it.
- **Single-click a handle** -> select + start a drag (selection alone does not dirty the data).
- **Click the bar** -> add a stop there, colored by sampling the ramp at that point so the gradient does not visually jump.

```c
GradientArb *g = reinterpret_cast<GradientArb*>(PF_LOCK_HANDLE(arbH));
PF_Boolean dirty = FALSE, redraw = FALSE, grabbed = FALSE; A_long grab_index = -1;
if (g) {
    int hit = HitHandle(&L, g, mx, my);

    if (hit >= 0 && clicks >= 2) {
        // ── Double-click -> host color picker. UNLOCK across the modal dialog. ──
        PF_PixelFloat samp; AEFX_CLR_STRUCT(samp);
        samp.alpha = 1.f; samp.red = g->stops[hit].r; samp.green = g->stops[hit].g; samp.blue = g->stops[hit].b;
        PF_UNLOCK_HANDLE(arbH); g = NULL;                 // must not hold the lock during the dialog

        PF_PixelFloat picked; AEFX_CLR_STRUCT(picked);
        PF_Err pErr = PF_Err_NONE;
        if (in_data->appl_id != 'PrMr')                   // Premiere lacks this picker
            pErr = suites.AppSuite6()->PF_AppColorPickerDialog("Stop Color", &samp, TRUE, &picked);

        g = reinterpret_cast<GradientArb*>(PF_LOCK_HANDLE(arbH));   // re-lock
        if (g && !pErr && hit < g->count) {               // pErr != 0 means cancelled
            g->stops[hit].r = clampf(picked.red,   0.f, 1.f);
            g->stops[hit].g = clampf(picked.green, 0.f, 1.f);
            g->stops[hit].b = clampf(picked.blue,  0.f, 1.f);
            g->selected = hit;
            dirty = TRUE;
        }
    } else if (hit >= 0 && (mods & PF_Mod_OPT_ALT_KEY)) {
        DeleteStop(g, hit);
        if (g->selected >= g->count) g->selected = -1;
        dirty = TRUE;
    } else if (hit >= 0) {
        g->selected = hit;                                // selection only - no change_flags
        redraw = TRUE; grabbed = TRUE; grab_index = hit;
    } else if (InBar(&L, mx, my)) {
        float t = clampf(((float)mx - L.barL) / L.barW, 0.f, 1.f);
        float r, gr, b; SampleGradient(g, t, &r, &gr, &b);  // sample the ramp -> no jump
        int ni = InsertStop(g, t, r, gr, b);
        if (ni >= 0) { g->selected = ni; dirty = TRUE; grabbed = TRUE; grab_index = ni; }
    }
    if (g) PF_UNLOCK_HANDLE(arbH);
}

if (dirty) params[GRAD_UI]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
if (dirty || redraw) { PF_Rect r = *cf; ERR(suites.AppSuite4()->PF_InvalidateRect(event_extra->contextH, &r)); }
if (grabbed) {
    event_extra->u.do_click.send_drag = TRUE;
    event_extra->u.do_click.continue_refcon[1] = grab_index;   // drag target
}
event_extra->evt_out_flags |= PF_EO_HANDLED_EVENT | PF_EO_UPDATE_NOW;
```

> **Unlock the arb handle before the modal color picker.** `PF_AppColorPickerDialog` is a blocking modal. Holding the locked arb handle across it risks deadlock and blocks any reentrant access. Snapshot the current color into a local, unlock, run the dialog, then re-lock and write the result back. Check `in_data->appl_id != 'PrMr'` first — Premiere does not provide this picker. A non-zero return means the user cancelled; leave the stop unchanged.

`PF_AppColorPickerDialog(title, &startColor, useStartColor, &outColor)` lives on `AppSuite6` and works in normalized `PF_PixelFloat` (0..1) — no fixed/8-bit conversion needed.

## Drag: move clamped between neighbours, or drag-out to delete

The drag moves the grabbed stop's position, clamped strictly between its neighbours so the sort order (and the index) stays valid. Dragging the handle clearly out the bottom of the control deletes it — a natural gesture borrowed from native gradient editors.

```c
A_long idx = (A_long)event_extra->u.do_click.continue_refcon[1];
GradientArb *g = reinterpret_cast<GradientArb*>(PF_LOCK_HANDLE(params[GRAD_UI]->u.arb_d.value));
PF_Boolean changed = FALSE;
if (g && idx >= 0 && idx < g->count) {
    if ((float)my > (float)cf->bottom + 6.f && g->count > GRAD_MIN_STOPS) {
        DeleteStop(g, idx);
        g->selected = -1;
        event_extra->u.do_click.continue_refcon[1] = -1;   // stop responding to this drag
    } else {
        float t = clampf(((float)mx - L.barL) / L.barW, 0.f, 1.f);
        float lo = (idx > 0)            ? g->stops[idx - 1].pos + 0.001f : 0.f;
        float hi = (idx < g->count - 1) ? g->stops[idx + 1].pos - 0.001f : 1.f;
        g->stops[idx].pos = clampf(t, lo, hi);
    }
    changed = TRUE;
}
if (g) PF_UNLOCK_HANDLE(params[GRAD_UI]->u.arb_d.value);

if (changed) {
    params[GRAD_UI]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
    PF_Rect inval = *cf; ERR(suites.AppSuite4()->PF_InvalidateRect(event_extra->contextH, &inval));
}
event_extra->evt_out_flags |= PF_EO_HANDLED_EVENT | PF_EO_UPDATE_NOW;
```

> **Endpoints are not specially pinned here** (unlike the curve editor) — the gradient's first and last stops can move, because a gradient may legitimately not start at 0 or end at 1. The `lo`/`hi` clamp against neighbours is all that is needed to keep the list sorted; `SampleGradient` holds the ends flat for `t` outside the stop range.

## Undo, change-flags, and the arb handlers

The same discipline as the curve editor:

- **Selection (`redraw`) only invalidates the canvas**; it never sets `PF_ChangeFlag_CHANGED_VALUE`, so it does not pollute undo or re-render.
- **Every real edit** (add, delete, move, recolor) sets `CHANGED_VALUE`; a click-then-drag gesture coalesces into one undo step automatically.
- The arb callbacks (`COPY`/`FLATTEN`/`UNFLATTEN` are `memcpy`; `INTERP` blends stop-for-stop when both keyframes share a count, else holds the left; `COMPARE` diffs `count` + stop fields and ignores `selected`) are exactly the [Custom Parameter UI](custom-param-ui.md) pattern for a fixed-size POD.

## Render: the gradient map (MFR-safe, 8/16/32)

The render snapshots the gradient into a `refcon`, picks a driving channel (a "Map By" popup: luminance / R / G / B / alpha), maps that channel through the gradient, and blends with the original by a "Blend with Original" amount. All state is in the stack `refcon` passed to the iterate suites, so it is MFR-safe.

```c
typedef struct {
    GradientArb g;
    float       blend;   // 0..1
    int         mapBy;   // MAP_* - which channel drives the lookup
} GradRefcon;

static inline void MapPixel(const GradRefcon *R, float fr, float fg, float fb, float fa,
                            float *or_, float *og, float *ob)
{
    float t;
    switch (R->mapBy) {
        case MAP_RED:   t = fr; break;
        case MAP_GREEN: t = fg; break;
        case MAP_BLUE:  t = fb; break;
        case MAP_ALPHA: t = fa; break;
        default:        t = 0.2126f*fr + 0.7152f*fg + 0.0722f*fb; break;   // luminance
    }
    float gr, gg, gb;
    SampleGradient(&R->g, clamp01(t), &gr, &gg, &gb);
    *or_ = fr + (gr - fr) * R->blend;       // blend toward the mapped color
    *og  = fg + (gg - fg) * R->blend;
    *ob  = fb + (gb - fb) * R->blend;
}
```

Snapshot the arb and the two scalars in `SmartRender`, then dispatch per depth:

```c
GradRefcon R; AEFX_CLR_STRUCT(R);

PF_ParamDef arb; AEFX_CLR_STRUCT(arb);
ERR(PF_CHECKOUT_PARAM(in_data, GRAD_UI, in_data->current_time,
                      in_data->time_step, in_data->time_scale, &arb));
if (!err && arb.u.arb_d.value) {
    GradientArb *g = reinterpret_cast<GradientArb*>(PF_LOCK_HANDLE(arb.u.arb_d.value));
    if (g) { R.g = *g; PF_UNLOCK_HANDLE(arb.u.arb_d.value); }   // copy out, then unlock
}
ERR2(PF_CHECKIN_PARAM(in_data, &arb));
// ... check out GRAD_BLEND -> R.blend, GRAD_MAP_BY -> R.mapBy ...

switch (format) {
    case PF_PixelFormat_ARGB128: ERR(suites.IterateFloatSuite2()->iterate(in_data, 0, h, inputP, NULL, &R, GradMapFuncFloat, outputP)); break;
    case PF_PixelFormat_ARGB64:  ERR(suites.Iterate16Suite2()  ->iterate(in_data, 0, h, inputP, NULL, &R, GradMapFunc16,    outputP)); break;
    case PF_PixelFormat_ARGB32:  ERR(suites.Iterate8Suite2()   ->iterate(in_data, 0, h, inputP, NULL, &R, GradMapFunc8,     outputP)); break;
    default: err = PF_Err_BAD_CALLBACK_PARAM; break;
}
```

The per-depth functions normalize the input to 0..1, call `MapPixel`, clamp, and scale back (`PF_MAX_CHAN8` / `PF_MAX_CHAN16`); the float path passes through unscaled. Alpha is copied straight through (the map only touches RGB). Because `SampleGradient` is the *same* function the bar draws with, the rendered map matches the on-screen gradient exactly.

## Extension ideas

- **Per-stop alpha** — add an `a` field to `GradStop`, draw a checkerboard behind the bar, and let the render map alpha too. Bump `version` and migrate in `UNFLATTEN`.
- **Eyedropper add** — sample a footage pixel for a new stop's color via `PF_GetColorAtGlobalPoint` instead of the ramp.
- **Portable LUT export** — `SampleGradient` is pure and pointer-free, so it lifts directly into a shader/compute LUT bake for a GPU path.

## Related reading

- [Custom Parameter UI](custom-param-ui.md) — the arb-data callback system this gradient persists through.
- [ECW Curve / Response Editor](ecw-curve-editor.md) — the sibling arb-handle-drag editor.
- [ECW Widget Gallery](ecw-widget-gallery.md) — Drawbot helpers, `NewImageFromBuffer`, multi-canvas dispatch.
- [Color Picker](../parameters/color-picker.md) and [Color parameters](../parameters/color-param.md) — the host color-picker API in more depth.
