# ECW Widget Gallery: Reusable Drawbot Controls

This guide is a cookbook of concrete, cherry-pickable custom-UI **widgets** for the Effect Controls Window (ECW), all drawn with Drawbot: an XY pad, a rotary knob/dial, a click-to-toggle button, a primitives-and-text reference card, and a `NewImageFromBuffer` image blit. The material here builds directly on the general mechanics in [Custom UI: Drawing in the Effect Controls Window](custom-ui-drawing.md) and [Custom Parameter UI](custom-param-ui.md) — read those first for the `register_ui` / event-dispatch / Drawbot-suite fundamentals. This page does **not** repeat them; it shows how to assemble those primitives into finished, interactive controls, and the patterns that make a *gallery* of independent widgets coexist in one effect.

The widgets below come from a small open sample effect that puts every control on its own canvas in one Effect Controls panel, so each one can be lifted out in isolation.

## The one-canvas-per-widget pattern

The key structural idea: **each widget is a separate custom-UI canvas**, and a canvas is just a parameter whose `ui_flags` reserve a drawing rectangle. A canvas that holds no value of its own is a `PF_ADD_NULL` param; a canvas backed by persisted data is a `PF_ADD_ARBITRARY2` param (see the [curve editor](ecw-curve-editor.md) and [gradient editor](ecw-gradient-editor.md)).

Reserve a canvas by setting `PF_PUI_CONTROL` plus a width/height on the `PF_ParamDef` before adding a NULL param:

```c
// PF_PUI_CONTROL + ui_width/ui_height on a NULL param reserve an ECW canvas.
// PF_PUI_DONT_ERASE_CONTROL: we paint our own background every draw, so we tell
// AE not to pre-clear the area (avoids a flash before our PaintRect lands).
#define ADD_CANVAS(NAME, W, H, ID)                              \
    AEFX_CLR_STRUCT(def);                                       \
    def.ui_flags  = PF_PUI_CONTROL | PF_PUI_DONT_ERASE_CONTROL; \
    def.ui_width  = (W) + 2 * UI_GUTTER;                        \
    def.ui_height = (H) + 2 * UI_GUTTER;                        \
    PF_ADD_NULL((NAME), (ID));
```

Then in `PF_Cmd_PARAMS_SETUP`, lay the widgets out interleaved with their backing value params:

```c
ADD_CANVAS("XY Pad", UI_XYPAD_SZ, UI_XYPAD_SZ, XYPAD_UI_DISK_ID);
PF_ADD_FLOAT_SLIDERX("Pad X", 0, 1, 0, 1, 0.5, PF_Precision_THOUSANDTHS, 0, 0, PAD_X_DISK_ID);
PF_ADD_FLOAT_SLIDERX("Pad Y", 0, 1, 0, 1, 0.5, PF_Precision_THOUSANDTHS, 0, 0, PAD_Y_DISK_ID);

ADD_CANVAS("Knob", UI_KNOB_SZ, UI_KNOB_SZ, KNOB_UI_DISK_ID);
PF_ADD_FLOAT_SLIDERX("Angle", -180, 180, -180, 180, 0, PF_Precision_TENTHS, 0, 0, KNOB_ANGLE_DISK_ID);

ADD_CANVAS("Button", UI_BTN_W, UI_BTN_H, BTN_UI_DISK_ID);
PF_ADD_CHECKBOX("Button State", "On", FALSE, 0, BTN_STATE_DISK_ID);

// One register_ui call covers every canvas the effect owns.
PF_CustomUIInfo ci;
AEFX_CLR_STRUCT(ci);
ci.events = PF_CustomEFlag_EFFECT;          // ECW only
err = (*(in_data->inter.register_ui))(in_data->effect_ref, &ci);
```

Each widget canvas is a *separate* parameter, and the value it edits is a *different*, ordinary parameter (a slider, a checkbox, or an arb). The canvas draws the value and turns mouse gestures into value edits; the value param is what gets keyframed, saved, and read at render time. This separation is what lets the same canvas code drive any standard parameter type.

> **Disk IDs are forever.** Each canvas and value param gets a stable `_DISK_ID` (passed as the last arg to the `PF_ADD_*` macros). Never renumber them once shipped — saved projects bind to those IDs. Reorder the visual layout freely; just keep the IDs fixed.

## Routing events to the right widget: `effect_win.index`

Because every widget is its own parameter/canvas, every `PF_Event_DRAW`, `DO_CLICK`, and `DRAG` carries the **parameter index of the canvas it targets** in `event_extra->effect_win.index`. That single field is the multiplexer for the whole gallery. Dispatch on it:

```c
// DRAW: acquire the drawing ref once, then fan out by which canvas AE is asking for.
static PF_Err
DrawEvent(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_EventExtra *ev)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;

    // Only ever draw inside an ECW control area.
    if (ev->effect_win.area != PF_EA_CONTROL ||
        (*ev->contextH)->w_type != PF_Window_EFFECT) return err;

    DRAWBOT_DrawRef draw = NULL; DRAWBOT_SupplierRef sup = NULL; DRAWBOT_SurfaceRef surf = NULL;
    DRAWBOT_Suites db;
    ERR(AEFX_AcquireDrawbotSuites(in_data, out_data, &db));

    PF_EffectCustomUISuite1 *cui = NULL;
    ERR(AEFX_AcquireSuite(in_data, out_data, kPFEffectCustomUISuite,
                          kPFEffectCustomUISuiteVersion1, NULL, (void**)&cui));
    if (!err && cui) {
        ERR((*cui->PF_GetDrawingReference)(ev->contextH, &draw));
        AEFX_ReleaseSuite(in_data, out_data, kPFEffectCustomUISuite, kPFEffectCustomUISuiteVersion1, NULL);
    }
    ERR(db.drawbot_suiteP->GetSupplier(draw, &sup));
    ERR(db.drawbot_suiteP->GetSurface(draw, &surf));

    if (!err) {
        switch (ev->effect_win.index) {           // <-- the multiplexer
            case G_PRIM_UI:  ERR(DrawPrimitives(in_data, params, ev, &db, sup, surf)); break;
            case G_IMG_UI:   ERR(DrawImageBlit (in_data, params, ev, &db, sup, surf)); break;
            case G_XYPAD_UI: ERR(DrawXYPad     (in_data, params, ev, &db, sup, surf)); break;
            case G_KNOB_UI:  ERR(DrawKnob      (in_data, params, ev, &db, sup, surf)); break;
            case G_BTN_UI:   ERR(DrawButton    (in_data, params, ev, &db, sup, surf)); break;
            case G_CURVE_UI: ERR(DrawCurve     (in_data, params, ev, &db, sup, surf)); break;
            default: break;
        }
    }
    ERR2(AEFX_ReleaseDrawbotSuites(in_data, out_data));
    ev->evt_out_flags = PF_EO_HANDLED_EVENT;
    return err;
}
```

`DO_CLICK` and `DRAG` dispatch the same way. Note the click/drag handlers gate on `effect_win.area == PF_EA_CONTROL` and route by `effect_win.index`:

```c
case PF_Event_DO_CLICK:
    if (extra->effect_win.area == PF_EA_CONTROL) {
        switch (extra->effect_win.index) {
            case G_XYPAD_UI: err = PadFromMouse (in_data, params, extra); extra->u.do_click.send_drag = TRUE; break;
            case G_KNOB_UI:  err = KnobFromMouse(in_data, params, extra); extra->u.do_click.send_drag = TRUE; break;
            case G_BTN_UI:   err = ToggleButton (in_data, params, extra); break;
            case G_CURVE_UI: err = ClickCurve   (in_data, params, extra); break;  // sets its own flags
            default: break;
        }
        extra->evt_out_flags |= PF_EO_HANDLED_EVENT | PF_EO_UPDATE_NOW;
    }
    break;
```

> **Why this matters.** The general ECW docs show a *single* custom control. Real plugins (and the gradient/histogram/curve samples) carry several. `effect_win.index` is the supported, documented way to tell them apart — there is no per-param event callback. Acquire the Drawbot suites **once** in the top-level draw handler and pass them down to the per-widget draw functions, rather than re-acquiring per widget.

## Shared Drawbot helpers

Drawbot's create/use/release dance (every `NewPen`/`NewBrush`/`NewPath` needs a matching `ReleaseObject`) is verbose. A handful of thin helpers collapse it. Lift these into any plugin; the rest of the gallery is written against them.

```c
static PF_Err dbFillRect(DRAWBOT_Suites *db, DRAWBOT_SupplierRef sup, DRAWBOT_SurfaceRef surf,
                         float x, float y, float w, float h, const DRAWBOT_ColorRGBA *c)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    DRAWBOT_PathRef p = NULL; DRAWBOT_BrushRef b = NULL;
    DRAWBOT_RectF32 r = { x, y, w, h };              // NOTE: x, y, WIDTH, HEIGHT (not right/bottom)
    ERR(db->supplier_suiteP->NewPath(sup, &p));
    ERR(db->supplier_suiteP->NewBrush(sup, c, &b));
    ERR(db->path_suiteP->AddRect(p, &r));
    ERR(db->surface_suiteP->FillPath(surf, b, p, kDRAWBOT_FillType_Default));
    if (b) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)b));
    if (p) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)p));
    return err;
}

static PF_Err dbCircle(DRAWBOT_Suites *db, DRAWBOT_SupplierRef sup, DRAWBOT_SurfaceRef surf,
                       float cx, float cy, float r, const DRAWBOT_ColorRGBA *c, PF_Boolean fill, float lw)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    DRAWBOT_PathRef p = NULL;
    DRAWBOT_PointF32 ctr = { cx, cy };
    ERR(db->supplier_suiteP->NewPath(sup, &p));
    ERR(db->path_suiteP->AddArc(p, &ctr, r, 0.f, 360.f));
    ERR(db->path_suiteP->Close(p));                 // close each arc (see Drawbot AddArc gotcha)
    if (fill) {
        DRAWBOT_BrushRef b = NULL;
        ERR(db->supplier_suiteP->NewBrush(sup, c, &b));
        ERR(db->surface_suiteP->FillPath(surf, b, p, kDRAWBOT_FillType_Default));
        if (b) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)b));
    } else {
        DRAWBOT_PenRef pen = NULL;
        ERR(db->supplier_suiteP->NewPen(sup, c, lw, &pen));
        ERR(db->surface_suiteP->StrokePath(surf, pen, p));
        if (pen) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)pen));
    }
    if (p) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)p));
    return err;
}
```

The full helper set in the sample is `dbFillRect`, `dbStrokeRect`, `dbCircle`, `dbLine`, and `dbLabel` (text), plus two layout helpers covered next. `dbStrokeRect` / `dbLine` follow the same shape as `dbFillRect` but build a pen instead of a brush.

> **`AddArc` + `Close`.** Always `Close` a path after each arc/circle. Without it, multiple arcs in one path are joined by a connecting line (a documented Drawbot gotcha) — see the [Drawbot Q&A](../advanced/drawbot.md).

## Coordinate origin and self-painted background

ECW frame coordinates are screen pixels relative to the panel, and Drawbot's pixel centers sit at `(x + 0.5)`. Two tiny helpers handle the per-canvas origin (inset by a gutter, plus the half-pixel for crisp 1px lines) and the background fill:

```c
// Top-left of the inner draw area from the control frame.
static void CanvasOrigin(const PF_Rect *cf, float *L, float *T)
{
    *L = (float)cf->left + UI_GUTTER + 0.5f;
    *T = (float)cf->top  + UI_GUTTER + 0.5f;
}

static PF_Err PaintBG(DRAWBOT_Suites *db, DRAWBOT_SurfaceRef surf, const PF_Rect *cf, float gray)
{
    DRAWBOT_ColorRGBA bg = { gray, gray, gray, 1.0f };
    DRAWBOT_RectF32 panel = { (float)cf->left + 0.5f, (float)cf->top + 0.5f,
                              (float)(cf->right - cf->left), (float)(cf->bottom - cf->top) };
    return db->surface_suiteP->PaintRect(surf, &bg, &panel);   // PaintRect: no path/brush needed
}
```

Every widget calls `PaintBG` first (paired with `PF_PUI_DONT_ERASE_CONTROL` on the canvas param) so it owns its full rectangle and never flickers.

## Text with the UTF-16 conversion

`DrawString` requires UTF-16, and the conversion differs by platform. Wrap it once:

```c
static void ToUTF16(const wchar_t *in, A_UTF16Char *out)
{
#ifdef AE_OS_MAC
    int len = (int)wcslen(in);
    CFRange range = {0, len};
    CFStringRef s = CFStringCreateWithBytes(kCFAllocatorDefault, (const UInt8*)in,
                            len * sizeof(wchar_t), kCFStringEncodingUTF32LE, false);
    CFStringGetBytes(s, range, kCFStringEncodingUTF16, 0, false,
                     (UInt8*)out, len * sizeof(A_UTF16Char), NULL);
    out[len] = 0;
    CFRelease(s);
#else
    size_t len = wcslen(in);
    wcscpy_s((wchar_t*)out, len + 1, in);    // Windows wchar_t is already UTF-16
#endif
}

static PF_Err dbLabel(DRAWBOT_Suites *db, DRAWBOT_SupplierRef sup, DRAWBOT_SurfaceRef surf,
                      float x, float y, const wchar_t *text, const DRAWBOT_ColorRGBA *c)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    float fsize = 0.f;
    DRAWBOT_FontRef font = NULL; DRAWBOT_BrushRef brush = NULL;
    ERR(db->supplier_suiteP->GetDefaultFontSize(sup, &fsize));
    ERR(db->supplier_suiteP->NewDefaultFont(sup, fsize, &font));
    ERR(db->supplier_suiteP->NewBrush(sup, c, &brush));
    A_UTF16Char buf[256];
    ToUTF16(text, buf);
    DRAWBOT_PointF32 org = { x, y };
    ERR(db->surface_suiteP->DrawString(surf, brush, font, buf, &org,
                                       kDRAWBOT_TextAlignment_Default, kDRAWBOT_TextTruncation_None, 0.f));
    if (brush) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)brush));
    if (font)  ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)font));
    return err;
}
```

A `swprintf` into a `wchar_t[]` buffer plus `dbLabel` is enough for live numeric readouts (`L"X %.2f  Y %.2f"`).

## Widget: XY pad

A box you drag a dot in, writing two `0..1` float params. The draw maps the two values into the box (with Y flipped so up is +Y), and the click/drag maps the mouse back to the values.

```c
static PF_Err
DrawXYPad(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev,
          DRAWBOT_Suites *db, DRAWBOT_SupplierRef sup, DRAWBOT_SurfaceRef surf)
{
    PF_Err err = PF_Err_NONE;
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float L, T; CanvasOrigin(cf, &L, &T);
    float S = (float)UI_XYPAD_SZ;
    ERR(PaintBG(db, surf, cf, 0.12f));

    DRAWBOT_ColorRGBA frame = {0.4f,0.4f,0.4f,1.f}, grid = {0.22f,0.22f,0.22f,1.f}, dot = {1.f,0.55f,0.15f,1.f};
    ERR(dbStrokeRect(db, sup, surf, L, T, S, S, &frame, 1.f));
    ERR(dbLine(db, sup, surf, L + S*.5f, T, L + S*.5f, T + S, &grid, 1.f));   // crosshair
    ERR(dbLine(db, sup, surf, L, T + S*.5f, L + S, T + S*.5f, &grid, 1.f));

    float px = clampf((float)params[G_PAD_X]->u.fs_d.value, 0.f, 1.f);
    float py = clampf((float)params[G_PAD_Y]->u.fs_d.value, 0.f, 1.f);
    float dx = L + px * S;
    float dy = T + (1.f - py) * S;            // Y up
    ERR(dbCircle(db, sup, surf, dx, dy, 5.f, &dot, TRUE, 0.f));
    return err;
}

// Mouse -> two params. Used for both DO_CLICK and DRAG.
static PF_Err
PadFromMouse(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float L, T; CanvasOrigin(cf, &L, &T);
    float S = (float)UI_XYPAD_SZ;
    float mx = (float)ev->u.do_click.screen_point.h;
    float my = (float)ev->u.do_click.screen_point.v;
    params[G_PAD_X]->u.fs_d.value = clampf((mx - L) / S, 0.f, 1.f);
    params[G_PAD_Y]->u.fs_d.value = clampf(1.f - (my - T) / S, 0.f, 1.f);   // Y up
    params[G_PAD_X]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
    params[G_PAD_Y]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
    PF_Rect inval = *cf;
    ERR(suites.AppSuite4()->PF_InvalidateRect(ev->contextH, &inval));        // force redraw now
    return err;
}
```

The same `PadFromMouse` is wired to both `DO_CLICK` (which also sets `send_drag = TRUE`) and `DRAG`, so a click jumps the dot and a drag scrubs it. Setting `PF_ChangeFlag_CHANGED_VALUE` is what makes the edit re-render and join the undo stack; the click+drags coalesce into one undo step.

## Widget: knob / dial

A rotary control. The indicator is computed from the angle param with `sin`/`cos`; the drag converts the mouse vector to an angle with `atan2`. The convention here is **0 degrees = up (12 o'clock), clockwise positive** — note `atan2f(dx, -dy)`, which rotates the usual math convention so 0 points up.

```c
static PF_Err
DrawKnob(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev,
         DRAWBOT_Suites *db, DRAWBOT_SupplierRef sup, DRAWBOT_SurfaceRef surf)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float L, T; CanvasOrigin(cf, &L, &T);
    float S = (float)UI_KNOB_SZ, cx = L + S*.5f, cy = T + S*.5f, R = S*.42f;
    ERR(PaintBG(db, surf, cf, 0.12f));

    DRAWBOT_ColorRGBA face = {0.20f,0.20f,0.22f,1.f}, rim = {0.45f,0.45f,0.45f,1.f}, accent = {0.3f,0.8f,0.95f,1.f};
    ERR(dbCircle(db, sup, surf, cx, cy, R, &face, TRUE, 0.f));
    ERR(dbCircle(db, sup, surf, cx, cy, R, &rim,  FALSE, 1.5f));

    float deg = (float)params[G_KNOB_ANGLE]->u.fs_d.value;   // -180..180, 0 = up
    float rad = deg * 3.14159265f / 180.f;
    float ix = cx + sinf(rad) * (R - 4.f);
    float iy = cy - cosf(rad) * (R - 4.f);                   // -cos: 0deg points up
    ERR(dbLine(db, sup, surf, cx, cy, ix, iy, &accent, 2.5f));

    // Sweep arc from 12 o'clock to the current angle. AddArc: 0deg = 3 o'clock,
    // so top is -90deg; the angle adds clockwise from there.
    {
        DRAWBOT_PathRef p = NULL; DRAWBOT_PenRef pen = NULL;
        DRAWBOT_PointF32 ctr = { cx, cy };
        ERR(db->supplier_suiteP->NewPath(sup, &p));
        ERR(db->path_suiteP->AddArc(p, &ctr, R + 5.f, -90.f, deg));
        ERR(db->supplier_suiteP->NewPen(sup, &accent, 2.f, &pen));
        ERR(db->surface_suiteP->StrokePath(surf, pen, p));
        if (pen) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)pen));
        if (p)   ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)p));
    }
    return err;
}

static PF_Err
KnobFromMouse(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float L, T; CanvasOrigin(cf, &L, &T);
    float S = (float)UI_KNOB_SZ, cx = L + S*.5f, cy = T + S*.5f;
    float dx = (float)ev->u.do_click.screen_point.h - cx;
    float dy = (float)ev->u.do_click.screen_point.v - cy;
    float deg = atan2f(dx, -dy) * 180.f / 3.14159265f;       // 0 = up, clockwise positive
    params[G_KNOB_ANGLE]->u.fs_d.value = clampf(deg, -180.f, 180.f);
    params[G_KNOB_ANGLE]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
    PF_Rect inval = *cf;
    ERR(suites.AppSuite4()->PF_InvalidateRect(ev->contextH, &inval));
    return err;
}
```

> **AddArc angle convention.** `AddArc(path, &center, radius, startDeg, sweepDeg)` measures `0deg` at 3 o'clock and sweeps clockwise. To start a gauge at the top, start at `-90`. This is independent of the knob's own "0 = up" *value* convention, which is purely a choice of how the param maps to the indicator.

## Widget: button

A click target that flips a checkbox param. The fill color reflects state; the toggle happens on `DO_CLICK` only (no drag).

```c
static PF_Err
DrawButton(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev,
           DRAWBOT_Suites *db, DRAWBOT_SupplierRef sup, DRAWBOT_SurfaceRef surf)
{
    PF_Err err = PF_Err_NONE;
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float L, T; CanvasOrigin(cf, &L, &T);
    float W = (float)UI_BTN_W, H = (float)UI_BTN_H;
    ERR(PaintBG(db, surf, cf, 0.12f));

    PF_Boolean on = params[G_BTN_STATE]->u.bd.value;
    DRAWBOT_ColorRGBA fill = { 0.25f, 0.25f, 0.27f, 1.f };
    if (on) { fill.red = 0.20f; fill.green = 0.55f; fill.blue = 0.30f; }   // pressed/on visual
    DRAWBOT_ColorRGBA edge = {0.5f,0.5f,0.5f,1.f}, white = {0.95f,0.95f,0.95f,1.f};
    ERR(dbFillRect(db, sup, surf, L, T, W, H, &fill));
    ERR(dbStrokeRect(db, sup, surf, L, T, W, H, &edge, 1.f));
    ERR(dbLabel(db, sup, surf, L + 12.f, T + H*.5f + 4.f, on ? L"BUTTON: ON" : L"BUTTON: OFF", &white));
    return err;
}

static PF_Err
ToggleButton(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    params[G_BTN_STATE]->u.bd.value = !params[G_BTN_STATE]->u.bd.value;
    params[G_BTN_STATE]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
    PF_Rect inval = ev->effect_win.current_frame;
    ERR(suites.AppSuite4()->PF_InvalidateRect(ev->contextH, &inval));
    return err;
}
```

This is the custom-drawn equivalent of a momentary [button parameter](../parameters/button-param.md) — use it when you want full control over the button's look or want it to live inside a custom canvas alongside other widgets.

## Widget: image blit with `NewImageFromBuffer`

Drawbot cannot draw gradients, photos, or any per-pixel content directly — you build a CPU pixel buffer and hand it to `NewImageFromBuffer`, then `DrawImage`. This is the technique behind the gradient bar, the noise thumbnail in the [live-preview guide](../advanced/ecw-async-frame-and-live-preview.md), and any thumbnail or color wheel.

The one trap is **channel order**: the buffer layout must match the `DRAWBOT_PixelLayout` enum you declare. The sample makes the order self-verifying by writing a red band on the left, green in the middle, blue on the right — if a band shows the wrong color, the layout enum (or byte order) is wrong.

```c
static PF_Err
DrawImageBlit(PF_InData *in_data, PF_ParamDef *params[], PF_EventExtra *ev,
              DRAWBOT_Suites *db, DRAWBOT_SupplierRef sup, DRAWBOT_SurfaceRef surf)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    const PF_Rect *cf = &ev->effect_win.current_frame;
    float L, T; CanvasOrigin(cf, &L, &T);
    ERR(PaintBG(db, surf, cf, 0.10f));

    const int W = UI_IMG_W, H = UI_IMG_H - 16;
    std::vector<A_u_char> buf((size_t)W * H * 4, 0);
    for (int y = 0; y < H; ++y) {
        A_u_char ramp = (A_u_char)((float)y / (float)(H - 1) * 255.f);
        for (int x = 0; x < W; ++x) {
            size_t i = ((size_t)y * W + x) * 4;
            A_u_char r = 0, g = 0, b = 0;
            int third = x * 3 / W;
            if (third == 0) r = ramp; else if (third == 1) g = ramp; else b = ramp;
            // Byte order MUST match the layout enum below: here, A,R,G,B.
            buf[i + 0] = 255;  buf[i + 1] = r;  buf[i + 2] = g;  buf[i + 3] = b;
        }
    }
    DRAWBOT_ImageRef img = NULL;
    ERR(db->supplier_suiteP->NewImageFromBuffer(sup, W, H, W * 4,
                kDRAWBOT_PixelLayout_32ARGB_Straight, buf.data(), &img));
    DRAWBOT_PointF32 org = { L, T };
    ERR(db->surface_suiteP->DrawImage(surf, img, &org, 1.0f));     // 1.0f = full opacity
    if (img) ERR2(db->supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)img));
    return err;
}
```

Notes on `NewImageFromBuffer`:

- **`rowbytes`** is the third argument (`W * 4` for a tightly packed 32-bit buffer; `W * 3` for `kDRAWBOT_PixelLayout_24RGB`). Pass the real stride if your buffer is padded.
- **Layout enums** include `kDRAWBOT_PixelLayout_32ARGB_Straight`, `..._32ARGB_Premul`, `..._32BGRA_*`, and `kDRAWBOT_PixelLayout_24RGB`. You can query the host's preferred 32-bit order with `PrefersPixelLayoutBGRA` and pick ARGB vs BGRA accordingly. For an opaque grayscale or RGB thumbnail, `24RGB` is simplest and avoids any premultiply ambiguity.
- **`DrawImage` opacity < 1.0 darkens** the result (a documented Drawbot quirk); draw at `1.0f` and bake any fade into the buffer.
- **AE 2025 multi-image caveat.** On AE 2025, compositing *more than one* retained `DRAWBOT_ImageRef` per plugin context can corrupt the UI (a known regression — see the [Drawbot Q&A](../advanced/drawbot.md)). Prefer one image per context, and release each `DRAWBOT_ImageRef` immediately after `DrawImage` as shown.

## Driving the render from widget values

Because each widget edits an ordinary parameter, the render reads those parameters the usual way — `PF_CHECKOUT_PARAM` in `SmartRender`, nothing custom-UI-specific. The gallery's render, for instance, reads the knob angle and turns it into a brightness multiply:

```c
PF_ParamDef ang; AEFX_CLR_STRUCT(ang);
ERR(PF_CHECKOUT_PARAM(in_data, G_KNOB_ANGLE, in_data->current_time,
                      in_data->time_step, in_data->time_scale, &ang));
float mult = 1.f + (float)ang.u.fs_d.value / 180.f;   // -180 -> 0x, 0 -> 1x, +180 -> 2x
ERR2(PF_CHECKIN_PARAM(in_data, &ang));
```

For arb-backed widgets (curve, gradient), the render checks out the arb param and locks the handle the same way the draw handler does — see the [curve editor](ecw-curve-editor.md) and [gradient editor](ecw-gradient-editor.md), which both share a single evaluation function between draw and render so the preview always matches the output.

## Cursor feedback

Set a cursor for the whole effect in `PF_Event_ADJUST_CURSOR`. For per-widget cursors, branch on `effect_win.index` the same way the draw/click handlers do:

```c
case PF_Event_ADJUST_CURSOR:
    extra->u.adjust_cursor.set_cursor = PF_Cursor_EYEDROPPER;
    break;
```

## Checklist for adding a widget

1. Add a canvas param (`ADD_CANVAS` / `PF_ADD_NULL`, or `PF_ADD_ARBITRARY2` if it owns data) with a fresh enum slot and a stable disk ID.
2. Add its backing value param(s) right after it.
3. Write `DrawX` (reads the value, draws via the helpers), and `XFromMouse` (reads the mouse, writes the value + `PF_ChangeFlag_CHANGED_VALUE` + `PF_InvalidateRect`).
4. Add `case G_X_UI:` to the `DrawEvent`, `DO_CLICK`, and (if draggable) `DRAG` switches.
5. Bump `out_data->num_params`.

## Related reading

- [Custom UI: Drawing in the Effect Controls Window](custom-ui-drawing.md) — the underlying event model and Drawbot suite API.
- [Custom Parameter UI](custom-param-ui.md) — the arb-data callbacks the curve/gradient widgets rely on.
- [ECW Curve / Response Editor](ecw-curve-editor.md) and [ECW Gradient Editor](ecw-gradient-editor.md) — the two complex arb-backed widgets in full.
- [Live ECW data via async upstream frames](../advanced/ecw-async-frame-and-live-preview.md) — data-driven canvases (histogram, procedural preview).
- [Drawbot Q&A](../advanced/drawbot.md) — brightness, `AddArc`, and the AE 2025 multi-image notes.
