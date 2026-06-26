# ECW Curve / Response Editor

This guide builds a draggable, ARB-backed **curve editor** in the Effect Controls Window: control points stored in arbitrary data, hit-testing and dragging handles, adding points by clicking, deleting with Alt-click, pinned endpoints, smooth cubic-Hermite interpolation, and baking the curve to a 256-entry LUT that the render shares with the editor so the preview and the output never disagree.

It assumes you have read [Custom Parameter UI](custom-param-ui.md) (the full arb-data callback system) and [Custom UI: Drawing in the ECW](custom-ui-drawing.md) (event dispatch and Drawbot). This page focuses on the parts those don't cover: the *editor* logic and the *shared evaluation function*. The reusable Drawbot helpers (`dbLine`, `dbCircle`, `CanvasOrigin`, `PaintBG`) come from the [Widget Gallery](ecw-widget-gallery.md).

## Data model: control points in arb data

A curve is a small, fixed-capacity, **POD** struct — no pointers — so the arb flatten/copy/compare handlers are plain `memcpy`/field compares (see [Custom Parameter UI](custom-param-ui.md#implementing-each-callback) for the callback bodies). Keeping the array capacity fixed is the trick that makes a *variable*-length list trivially serializable.

```c
#define CURVE_MAX_PTS    16
#define CURVE_MIN_PTS    2
#define CURVE_ARB_VERSION 1

typedef struct { float x, y; } CurvePt;     // both 0..1

typedef struct {
    A_long  version;
    A_long  count;                 // CURVE_MIN_PTS .. CURVE_MAX_PTS
    A_long  selected;              // UI selection (index, or -1) - UI state, not visible data
    CurvePt pts[CURVE_MAX_PTS];    // kept sorted ascending by x; pts[0].x = 0, last.x = 1
} CurveArb;
```

Two invariants the editor maintains and the rest of the code relies on:

- **Sorted by x.** Insert keeps the array ordered, so segment lookup is a simple left-to-right scan.
- **Endpoints pinned in x.** `pts[0].x == 0` and `pts[count-1].x == 1` always; only their Y is editable. This guarantees the curve spans the full `[0,1]` input range.

The default is the identity diagonal `(0,0)-(1,1)` — a neutral response curve, so applying the effect changes nothing until the user edits it:

```c
AEFX_CLR_STRUCT(*c);
c->version  = CURVE_ARB_VERSION;
c->selected = -1;
c->count    = 2;
c->pts[0]   = { 0.f, 0.f };
c->pts[1]   = { 1.f, 1.f };
```

> **`selected` is UI state, not data.** Include it in the struct so it survives draws, but **exclude it from the arb `COMPARE` handler** — selecting a point must not mark the parameter changed (no re-render, no undo step). The compare only diffs `count` and the `pts` x/y values.

## Adding the curve param

Add it as an arb param with a `PF_PUI_CONTROL` canvas, just like any custom-UI arb (see [Custom Parameter UI](custom-param-ui.md#adding-an-arbitrary-data-parameter)):

```c
AEFX_CLR_STRUCT(def);
ERR(CreateDefaultCurve(in_data, out_data, &def.u.arb_d.dephault));
PF_ADD_ARBITRARY2("Curve / Response",
                  UI_CURVE_SZ + 2 * UI_GUTTER, UI_CURVE_SZ + 2 * UI_GUTTER, 0,
                  PF_PUI_CONTROL | PF_PUI_DONT_ERASE_CONTROL,
                  def.u.arb_d.dephault, CURVE_UI_DISK_ID, CURVE_ARB_REFCON);
```

`CURVE_ARB_REFCON` is a unique cookie the arb callbacks check so they only act on *this* effect's arb data.

## The shared evaluation function (the centerpiece)

The single most important design decision: **one function evaluates the curve, and both the editor and the render call it.** Put it in the header as a `static inline` so both `.cpp` files share it. If the editor drew the curve one way and the render computed pixels another way, the preview would lie. Sharing one evaluator makes that class of bug impossible.

The interpolation is monotone-ish **cubic Hermite** with finite-difference tangents (Catmull-Rom-style), clamped to `[0,1]`:

```c
static inline float ecwSafeDen(float d) { return (d < 1e-5f && d > -1e-5f) ? 1e-5f : d; }

// Finite-difference slope (dy/dx) at point i - the Hermite tangent.
static inline float CurveTangent(const CurveArb *c, int i)
{
    int n = c->count;
    if (n < 2) return 0.f;
    if (i <= 0)     return (c->pts[1].y   - c->pts[0].y)   / ecwSafeDen(c->pts[1].x   - c->pts[0].x);
    if (i >= n - 1) return (c->pts[n-1].y - c->pts[n-2].y) / ecwSafeDen(c->pts[n-1].x - c->pts[n-2].x);
    return                 (c->pts[i+1].y - c->pts[i-1].y) / ecwSafeDen(c->pts[i+1].x - c->pts[i-1].x);
}

// Smooth cubic-Hermite interpolation, clamped to [0,1]. Shared by editor + render.
static inline float SampleCurve(const CurveArb *c, float x)
{
    int n = c->count;
    if (n <= 0) return x;
    if (n == 1) return c->pts[0].y;
    if (x <= c->pts[0].x)   return c->pts[0].y;
    if (x >= c->pts[n-1].x) return c->pts[n-1].y;

    int i = 0;
    while (i < n - 1 && x > c->pts[i+1].x) ++i;       // find segment [i, i+1]
    float x0 = c->pts[i].x, x1 = c->pts[i+1].x, y0 = c->pts[i].y, y1 = c->pts[i+1].y;
    float h = ecwSafeDen(x1 - x0);
    float t = (x - x0) / h;
    float m0 = CurveTangent(c, i), m1 = CurveTangent(c, i + 1);
    float t2 = t*t, t3 = t2*t;
    float y = (2*t3 - 3*t2 + 1) * y0 + (t3 - 2*t2 + t) * h * m0 +
              (-2*t3 + 3*t2)    * y1 + (t3 - t2)       * h * m1;
    return y < 0.f ? 0.f : (y > 1.f ? 1.f : y);
}
```

`ecwSafeDen` guards against a zero-width segment (two points sharing an x) — the editor prevents that, but the evaluator stays robust if a keyframe-interpolated curve momentarily produces one.

## Drawing the editor

The draw locks the arb handle, strokes the curve as a dense polyline sampled from `SampleCurve`, then draws each control point (the selected one highlighted). Note the Y flip: data Y of 0 is at the bottom of the graph, so screen Y is `gy + (1 - y) * g`.

```c
static PF_Err
DrawCurve(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev,
          DRAWBOT_Suites *db, DRAWBOT_SupplierRef sup, DRAWBOT_SurfaceRef surf)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float gx, gy, g; CurveGraph(cf, &gx, &gy, &g);     // graph origin + size (uses CanvasOrigin)
    ERR(PaintBG(db, surf, cf, 0.10f));

    DRAWBOT_ColorRGBA frame={0.4f,0.4f,0.4f,1.f}, grid={0.2f,0.2f,0.2f,1.f},
                      accent={0.3f,0.8f,0.95f,1.f}, white={0.95f,0.95f,0.95f,1.f};
    ERR(dbStrokeRect(db, sup, surf, gx, gy, g, g, &frame, 1.f));
    for (int q = 1; q < 4; ++q) {                       // quarter gridlines
        ERR(dbLine(db, sup, surf, gx + g*q/4.f, gy, gx + g*q/4.f, gy + g, &grid, 1.f));
        ERR(dbLine(db, sup, surf, gx, gy + g*q/4.f, gx + g, gy + g*q/4.f, &grid, 1.f));
    }

    CurveArb *c = reinterpret_cast<CurveArb*>(PF_LOCK_HANDLE(params[G_CURVE_UI]->u.arb_d.value));
    if (!err && c) {
        // The curve as one stroked polyline, 2px/sample across the graph width.
        DRAWBOT_PathRef p = NULL; DRAWBOT_PenRef pen = NULL;
        ERR(db->supplier_suiteP->NewPath(sup, &p));
        float y0 = SampleCurve(c, 0.f);
        ERR(db->path_suiteP->MoveTo(p, gx, gy + (1.f - y0) * g));
        for (int px = 2; px <= UI_CURVE_SZ; px += 2) {
            float t  = (float)px / (float)UI_CURVE_SZ;
            float yy = SampleCurve(c, t);
            ERR(db->path_suiteP->LineTo(p, gx + (float)px, gy + (1.f - yy) * g));
        }
        ERR(db->supplier_suiteP->NewPen(sup, &accent, 2.f, &pen));
        ERR(db->surface_suiteP->StrokePath(surf, pen, p));
        if (pen) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)pen));
        if (p)   ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)p));

        // Control points; selected = solid white core.
        for (int i = 0; !err && i < c->count; ++i) {
            float ptx = gx + c->pts[i].x * g;
            float pty = gy + (1.f - c->pts[i].y) * g;     // Y up
            PF_Boolean sel = (i == c->selected);
            ERR(dbCircle(db, sup, surf, ptx, pty, 4.f, sel ? &white : &accent, TRUE, 0.f));
            ERR(dbCircle(db, sup, surf, ptx, pty, 4.f, &white, FALSE, 1.f));
        }
    }
    if (c) PF_UNLOCK_HANDLE(params[G_CURVE_UI]->u.arb_d.value);
    return err;
}
```

## Editing: insert, delete, hit-test

Two small list operations keep the array sorted and the endpoints protected:

```c
static int InsertCurvePt(CurveArb *c, float x, float y)
{
    if (c->count >= CURVE_MAX_PTS) return -1;
    int i = 0;
    while (i < c->count && c->pts[i].x < x) ++i;          // find sorted slot
    for (int k = c->count; k > i; --k) c->pts[k] = c->pts[k - 1];
    c->pts[i].x = x; c->pts[i].y = y;
    c->count++;
    return i;                                             // index of the new point
}

static void DeleteCurvePt(CurveArb *c, int i)
{
    // Keep endpoints (i == 0 / i == count-1) and never drop below CURVE_MIN_PTS.
    if (c->count <= CURVE_MIN_PTS || i <= 0 || i >= c->count - 1) return;
    for (int k = i; k < c->count - 1; ++k) c->pts[k] = c->pts[k + 1];
    c->count--;
}
```

## Click: select / grab / add / delete

The click handler hit-tests handles first (squared-distance against each point), then decides:

- **Alt-click an interior handle** -> delete it.
- **Click any handle** -> select + start a drag (selection alone is not a data change).
- **Click empty graph area** -> insert a point there and start dragging it.

Adding a point on click **samples the existing curve at that x** for the new Y, so the point lands *on* the curve and the shape doesn't jump.

```c
static PF_Err
ClickCurve(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float gx, gy, g; CurveGraph(cf, &gx, &gy, &g);
    float mx = (float)ev->u.do_click.screen_point.h;
    float my = (float)ev->u.do_click.screen_point.v;
    PF_Modifiers mods = ev->u.do_click.modifiers;
    PF_Handle arbH = params[G_CURVE_UI]->u.arb_d.value;

    CurveArb *c = reinterpret_cast<CurveArb*>(PF_LOCK_HANDLE(arbH));
    PF_Boolean dirty = FALSE, redraw = FALSE, grab = FALSE; A_long grabidx = -1;
    if (c) {
        int hit = -1;
        for (int i = 0; i < c->count; ++i) {
            float dx = mx - (gx + c->pts[i].x * g);
            float dy = my - (gy + (1.f - c->pts[i].y) * g);
            if (dx*dx + dy*dy <= 42.f) { hit = i; break; }   // ~6.5px hit radius
        }
        if (hit >= 0) {
            PF_Boolean interior = (hit > 0 && hit < c->count - 1);
            if ((mods & PF_Mod_OPT_ALT_KEY) && interior) {
                DeleteCurvePt(c, hit);
                if (c->selected >= c->count) c->selected = -1;
                dirty = TRUE;                                  // data changed
            } else {
                c->selected = hit; redraw = TRUE;             // selection only
                grab = TRUE; grabidx = hit;
            }
        } else if (mx >= gx && mx <= gx + g && my >= gy && my <= gy + g) {
            float nx = clampf((mx - gx) / g, 0.f, 1.f);
            float ny = clampf(1.f - (my - gy) / g, 0.f, 1.f);
            int ni = InsertCurvePt(c, nx, ny);
            if (ni >= 0) { c->selected = ni; dirty = TRUE; grab = TRUE; grabidx = ni; }
        }
        PF_UNLOCK_HANDLE(arbH);
    }

    if (dirty) params[G_CURVE_UI]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;   // re-render + undo
    if (dirty || redraw) { PF_Rect r = *cf; ERR(suites.AppSuite4()->PF_InvalidateRect(ev->contextH, &r)); }
    if (grab) {                                                 // hand off to DRAG
        ev->u.do_click.send_drag = TRUE;
        ev->u.do_click.continue_refcon[1] = grabidx;           // which point the drag moves
    }
    ev->evt_out_flags |= PF_EO_HANDLED_EVENT | PF_EO_UPDATE_NOW;
    return err;
}
```

> **`continue_refcon` carries the drag target.** When a click starts a drag, stash the grabbed point index in `ev->u.do_click.continue_refcon[1]`. AE preserves the `continue_refcon[]` array across the whole drag, so the `DRAG` handler knows which point to move without any global state — important for MFR-safe, re-entrant UI.

## Drag: move with clamping and live endpoint pinning

The drag reads the stashed index, moves that point, and enforces the invariants: endpoints keep their fixed X (only Y moves); interior points are clamped strictly between their neighbours so the array stays sorted and indices stay valid mid-drag.

```c
static PF_Err
DragCurve(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float gx, gy, g; CurveGraph(cf, &gx, &gy, &g);
    float mx = (float)ev->u.do_click.screen_point.h;
    float my = (float)ev->u.do_click.screen_point.v;
    A_long idx = (A_long)ev->u.do_click.continue_refcon[1];

    CurveArb *c = reinterpret_cast<CurveArb*>(PF_LOCK_HANDLE(params[G_CURVE_UI]->u.arb_d.value));
    PF_Boolean dirty = FALSE;
    if (c && idx >= 0 && idx < c->count) {
        float ny = clampf(1.f - (my - gy) / g, 0.f, 1.f);
        c->pts[idx].y = ny;
        if (idx == 0)                 c->pts[idx].x = 0.f;      // endpoints pinned in X
        else if (idx == c->count - 1) c->pts[idx].x = 1.f;
        else {
            float lo = c->pts[idx-1].x + 0.001f;               // strictly between neighbours
            float hi = c->pts[idx+1].x - 0.001f;
            c->pts[idx].x = clampf((mx - gx) / g, lo, hi);
        }
        dirty = TRUE;
    }
    if (c) PF_UNLOCK_HANDLE(params[G_CURVE_UI]->u.arb_d.value);

    if (dirty) {
        params[G_CURVE_UI]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
        PF_Rect r = *cf; ERR(suites.AppSuite4()->PF_InvalidateRect(ev->contextH, &r));
    }
    ev->evt_out_flags |= PF_EO_HANDLED_EVENT | PF_EO_UPDATE_NOW;
    return err;
}
```

## Undo grouping and `PF_ChangeFlag`

The `PF_ChangeFlag_CHANGED_VALUE` flag is what turns an in-place arb edit into a committed, undoable change. Two rules from the handlers above:

- **Set it only on real data changes.** Selecting a handle sets `redraw` (just `PF_InvalidateRect`), never `CHANGED_VALUE` — otherwise clicking to select would litter the undo history and force re-renders.
- **Drags coalesce automatically.** The initial `DO_CLICK` (which may add or grab a point) and every subsequent `DRAG` that set `CHANGED_VALUE` are grouped by AE into **one** undo step, because they belong to one click-drag gesture. You do not create undo steps manually; AE does it from the change-flag stream.

`PF_EO_UPDATE_NOW` in `evt_out_flags` asks AE to refresh the view immediately so the curve feels live under the cursor.

## Baking to a 256-LUT and rendering

The render checks out the arb param, locks the handle, and bakes `SampleCurve` into a 256-entry lookup table once per frame — far cheaper than evaluating the spline per pixel. The same `SampleCurve` the editor draws with fills the LUT, so output equals preview.

```c
// In SmartRender, after checking out the input/output worlds:
GalRefcon R; AEFX_CLR_STRUCT(R);

PF_ParamDef crv; AEFX_CLR_STRUCT(crv);
ERR(PF_CHECKOUT_PARAM(in_data, G_CURVE_UI, in_data->current_time,
                      in_data->time_step, in_data->time_scale, &crv));
if (!err && crv.u.arb_d.value) {
    CurveArb *c = reinterpret_cast<CurveArb*>(PF_LOCK_HANDLE(crv.u.arb_d.value));
    if (c) {
        for (int i = 0; i < 256; ++i) R.lut[i] = SampleCurve(c, (float)i / 255.f);
        PF_UNLOCK_HANDLE(crv.u.arb_d.value);
    }
}
ERR2(PF_CHECKIN_PARAM(in_data, &crv));
```

Apply the LUT per pixel with **linear interpolation** between entries so 16- and 32-bit footage does not band on the 256-step table:

```c
static inline float ApplyLut(const GalRefcon *R, float v)
{
    v = clamp01(v) * 255.f;
    int i = (int)v;
    if (i >= 255) return R->lut[255];
    float f = v - (float)i;
    return R->lut[i] + (R->lut[i + 1] - R->lut[i]) * f;
}
```

The per-pixel functions are dispatched per depth through the iterate suites (`Iterate8Suite2` / `Iterate16Suite2` / `IterateFloatSuite2`), which keeps the render MFR-safe — the LUT lives in a stack `refcon` passed to `iterate`, with no shared mutable state. See the [Widget Gallery render section](ecw-widget-gallery.md#driving-the-render-from-widget-values) for the dispatch skeleton.

> **Thread safety.** With `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`, never write the arb handle during render — lock, read into your `refcon`, unlock. All mutation happens in the UI selectors (`PF_Event_*`), where there is a single UI thread.

## Extension ideas

- **Per-segment interpolation modes** (linear / smooth / hold) by adding a `mode` byte to `CurvePt` and branching inside `SampleCurve`. Remember to bump `version` and migrate in the arb `UNFLATTEN` handler.
- **Keyframe-animatable curve** — the arb `INTERP` handler already blends point-for-point when both keyframes share a count; hold the left side otherwise. (See [Custom Parameter UI](custom-param-ui.md#pf_arbitrary_interp_func).)
- **Multiple channels** (separate R/G/B curves) by widening the arb struct to three point arrays.

## Related reading

- [Custom Parameter UI](custom-param-ui.md) — the arb-data callbacks (new/dispose/copy/flatten/interp/compare) this editor's data rides on.
- [ECW Gradient Editor](ecw-gradient-editor.md) — the same arb + handle-drag pattern for color stops.
- [ECW Widget Gallery](ecw-widget-gallery.md) — the Drawbot helpers and multi-canvas dispatch used here.
