# Custom UI: Drawing in the Effect Controls Window

This document covers how to implement custom UI drawing in the Effect Controls Window (ECW), including handling draw events, click events, cursor adjustment, and using the Drawbot suites for rendering.

## Architecture Overview

Custom UI in the ECW is driven by a message-based system. When your effect declares custom UI support, AE sends `PF_Cmd_EVENT` with a `PF_EventExtra*` in the `extra` parameter. You respond to specific event types to draw your UI, handle clicks, and adjust the cursor.

### Requirements to Enable Custom UI

1. Set `PF_OutFlag_CUSTOM_UI` in `out_data->out_flags` during `PF_Cmd_GLOBAL_SETUP`
2. Register the custom UI during `PF_Cmd_PARAM_SETUP` via `in_data->inter.register_ui`
3. Set `PF_PUI_CONTROL` (and/or `PF_PUI_TOPIC`) on parameters that have custom drawing

```c
// In GlobalSetup:
out_data->out_flags |= PF_OutFlag_CUSTOM_UI;

// In ParamsSetup, after adding parameters:
PF_CustomUIInfo ci;
AEFX_CLR_STRUCT(ci);

ci.events = PF_CustomEFlag_EFFECT;  // ECW events

ci.comp_ui_width    = 0;
ci.comp_ui_height   = 0;
ci.comp_ui_alignment = PF_UIAlignment_NONE;

ci.layer_ui_width   = 0;
ci.layer_ui_height  = 0;
ci.layer_ui_alignment = PF_UIAlignment_NONE;

ci.preview_ui_width  = 0;
ci.preview_ui_height = 0;
ci.layer_ui_alignment = PF_UIAlignment_NONE;

err = in_data->inter.register_ui(in_data->effect_ref, &ci);
```

### PF_CustomEFlag Values

| Flag | Value | Description |
|------|-------|-------------|
| `PF_CustomEFlag_NONE` | 0 | No events |
| `PF_CustomEFlag_COMP` | 1 << 0 | Receive events in the Comp panel |
| `PF_CustomEFlag_LAYER` | 1 << 1 | Receive events in the Layer panel |
| `PF_CustomEFlag_EFFECT` | 1 << 2 | Receive events in the Effect Controls Window |
| `PF_CustomEFlag_PREVIEW` | 1 << 3 | Receive events in the Preview panel |

For ECW-only custom UI, use `PF_CustomEFlag_EFFECT`.

## The PF_EventExtra Structure

Every `PF_Cmd_EVENT` call receives a `PF_EventExtra*` through the `extra` parameter:

```c
typedef struct {
    PF_ContextH         contextH;       // >> context handle
    PF_EventType        e_type;         // >> which event
    PF_EventUnion       u;              // >> event-specific data
    PF_EffectWindowInfo effect_win;     // >> ECW layout info (ECW events only)
    PF_EventCallbacks   cbs;            // >> coordinate transform callbacks
    PF_EventInFlags     evt_in_flags;   // >> input flags
    PF_EventOutFlags    evt_out_flags;  // << output flags (you set these)
} PF_EventExtra;
```

### Determining the Window Type

Check which window the event originates from:

```c
PF_WindowType w_type = (*event_extraP->contextH)->w_type;

switch (w_type) {
    case PF_Window_COMP:    // Composition panel
    case PF_Window_LAYER:   // Layer panel
    case PF_Window_EFFECT:  // Effect Controls Window
    case PF_Window_PREVIEW: // Preview panel
}
```

### PF_EffectWindowInfo (ECW-Specific)

When events come from the ECW, `effect_win` provides layout information:

```c
typedef struct {
    PF_ParamIndex    index;              // Which parameter
    PF_EffectArea    area;               // PF_EA_PARAM_TITLE or PF_EA_CONTROL
    PF_UnionableRect current_frame;      // The full frame of the current area
    PF_UnionableRect param_title_frame;  // The frame of the param title area
    A_long           horiz_offset;       // Horizontal offset for title drawing
} PF_EffectWindowInfo;
```

The `area` field tells you whether the event targets the title region (the label, always visible) or the control region (the area below the twirly, hidden when collapsed):

```c
if (PF_EA_CONTROL == event_extra->effect_win.area) {
    // Drawing in the control region
} else if (PF_EA_PARAM_TITLE == event_extra->effect_win.area) {
    // Drawing in the title/topic region
}
```

## Handling the Main Event Loop

Your `PF_Cmd_EVENT` handler dispatches based on `e_type`:

```c
PF_Err HandleEvent(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_LayerDef     *output,
    PF_EventExtra   *extra)
{
    PF_Err err = PF_Err_NONE;

    switch (extra->e_type) {
        case PF_Event_NEW_CONTEXT:
            // Window opened, initialize per-context state
            break;

        case PF_Event_ACTIVATE:
            // Window gained focus
            break;

        case PF_Event_DO_CLICK:
            err = DoClick(in_data, out_data, params, output, extra);
            break;

        case PF_Event_DRAG:
            err = DoDrag(in_data, out_data, params, output, extra);
            break;

        case PF_Event_DRAW:
            err = DrawEvent(in_data, out_data, params, output, extra);
            break;

        case PF_Event_ADJUST_CURSOR:
            err = AdjustCursor(in_data, out_data, params, output, extra);
            break;

        case PF_Event_KEYDOWN:
            err = HandleKeyDown(in_data, out_data, params, output, extra);
            break;

        case PF_Event_DEACTIVATE:
            // Window lost focus
            break;

        case PF_Event_CLOSE_CONTEXT:
            // Window closing, clean up per-context state
            break;

        case PF_Event_MOUSE_EXITED:
            // Mouse left the view (comp/layer only)
            break;

        default:
            break;
    }
    return err;
}
```

## Drawing with Drawbot Suites

AE provides the Drawbot suite family for all custom UI drawing. Platform-native drawing APIs (GDI, CoreGraphics) must not be used.

### The Four Drawbot Suites

| Suite | Constant | Purpose |
|-------|----------|---------|
| `DRAWBOT_DrawbotSuite1` | `kDRAWBOT_DrawSuite` | Get supplier and surface from a draw ref |
| `DRAWBOT_SupplierSuite1` | `kDRAWBOT_SupplierSuite` | Create pens, brushes, fonts, paths, images |
| `DRAWBOT_SurfaceSuite2` | `kDRAWBOT_SurfaceSuite` | Draw to the surface (fill, stroke, text, images) |
| `DRAWBOT_PathSuite1` | `kDRAWBOT_PathSuite` | Build paths (move, line, bezier, arc, rect) |

Additional suites:
| Suite | Constant | Purpose |
|-------|----------|---------|
| `DRAWBOT_PenSuite1` | `kDRAWBOT_PenSuite` | Configure pen properties (dash patterns) |
| `DRAWBOT_ImageSuite1` | `kDRAWBOT_ImageSuite` | Configure image properties (scale factor) |

### Acquiring the Drawing Reference

You need a `DRAWBOT_DrawRef` to begin drawing. Obtain it from `PF_EffectCustomUISuite`:

```c
PF_Err DrawEvent(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_LayerDef     *output,
    PF_EventExtra   *event_extra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    // Get the drawing reference
    DRAWBOT_DrawRef drawing_ref = NULL;
    ERR(suites.EffectCustomUISuite1()->PF_GetDrawingReference(
        event_extra->contextH, &drawing_ref));

    if (!err && drawing_ref) {
        // Get supplier and surface
        DRAWBOT_SupplierRef supplier_ref = NULL;
        DRAWBOT_SurfaceRef  surface_ref  = NULL;

        ERR(suites.DrawbotSuiteCurrent()->GetSupplier(drawing_ref, &supplier_ref));
        ERR(suites.DrawbotSuiteCurrent()->GetSurface(drawing_ref, &surface_ref));

        // ... draw here ...
    }

    // Signal that we handled the event
    event_extra->evt_out_flags = PF_EO_HANDLED_EVENT;
    return err;
}
```

> **Important**: Do NOT call `ReleaseObject` on `supplier_ref` or `surface_ref` -- you did not create them. Only release objects you create with `NewXxx` functions.

### Using the AEFX Convenience Functions

The SDK provides convenience functions to acquire all Drawbot suites at once:

```c
DRAWBOT_Suites drawbotSuites;
ERR(AEFX_AcquireDrawbotSuites(in_data, out_data, &drawbotSuites));

// Use drawbotSuites.drawbot_suiteP, drawbotSuites.supplier_suiteP, etc.

// Must match with release:
ERR(AEFX_ReleaseDrawbotSuites(in_data, out_data));
```

The `DRAWBOT_Suites` structure bundles all suite pointers:

```c
typedef struct {
    DRAWBOT_DrawbotSuiteCurrent*  drawbot_suiteP;
    DRAWBOT_SupplierSuiteCurrent* supplier_suiteP;
    DRAWBOT_SurfaceSuiteCurrent*  surface_suiteP;
    DRAWBOT_PathSuiteCurrent*     path_suiteP;
    DRAWBOT_PenSuiteCurrent*      pen_suiteP;
    DRAWBOT_ImageSuiteCurrent*    image_suiteP;
} DRAWBOT_Suites;
```

## Drawing Shapes

### Drawing a Filled Rectangle

```c
// Define color (RGBA, 0.0-1.0)
DRAWBOT_ColorRGBA color = {0.2f, 0.5f, 0.8f, 1.0f};

// Define rectangle (left, top, width, height -- NOT right/bottom)
DRAWBOT_RectF32 rect;
rect.left   = event_extra->effect_win.current_frame.left + 0.5f;
rect.top    = event_extra->effect_win.current_frame.top + 0.5f;
rect.width  = (float)(event_extra->effect_win.current_frame.right
            - event_extra->effect_win.current_frame.left);
rect.height = (float)(event_extra->effect_win.current_frame.bottom
            - event_extra->effect_win.current_frame.top);

// Paint directly (no brush/path needed)
surface_suiteP->PaintRect(surface_ref, &color, &rect);
```

> **Pitfall**: `DRAWBOT_RectF32` uses `left, top, width, height` -- NOT `left, top, right, bottom`. This is a common source of confusion. The comment in the SDK header itself notes "not right" and "not bottom".

### Drawing a Stroked Path

```c
// Create a pen (color + stroke width)
DRAWBOT_ColorRGBA pen_color = {1.0f, 1.0f, 1.0f, 1.0f};
DRAWBOT_PenRef pen_ref = NULL;
supplier_suiteP->NewPen(supplier_ref, &pen_color, 2.0f, &pen_ref);

// Create a path
DRAWBOT_PathRef path_ref = NULL;
supplier_suiteP->NewPath(supplier_ref, &path_ref);

// Build the path
path_suiteP->MoveTo(path_ref, 10.0f, 10.0f);
path_suiteP->LineTo(path_ref, 100.0f, 10.0f);
path_suiteP->LineTo(path_ref, 100.0f, 50.0f);
path_suiteP->Close(path_ref);

// Stroke it
surface_suiteP->StrokePath(surface_ref, pen_ref, path_ref);

// Release what you created
supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)path_ref);
supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)pen_ref);
```

### Drawing a Filled Path

```c
DRAWBOT_ColorRGBA brush_color = {1.0f, 0.0f, 0.0f, 0.5f};
DRAWBOT_BrushRef brush_ref = NULL;
supplier_suiteP->NewBrush(supplier_ref, &brush_color, &brush_ref);

DRAWBOT_PathRef path_ref = NULL;
supplier_suiteP->NewPath(supplier_ref, &path_ref);

// Add an arc (circle)
DRAWBOT_PointF32 center = {50.0f, 50.0f};
path_suiteP->AddArc(path_ref, &center, 25.0f, 0.0f, 360.0f);

surface_suiteP->FillPath(surface_ref, brush_ref, path_ref,
                          kDRAWBOT_FillType_EvenOdd);

supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)path_ref);
supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)brush_ref);
```

### Drawing with Bezier Curves

```c
path_suiteP->MoveTo(path_ref, start_x, start_y);

DRAWBOT_PointF32 cp1 = {ctrl1_x, ctrl1_y};
DRAWBOT_PointF32 cp2 = {ctrl2_x, ctrl2_y};
DRAWBOT_PointF32 end = {end_x, end_y};

path_suiteP->BezierTo(path_ref, &cp1, &cp2, &end);
```

### Dashed Lines

```c
DRAWBOT_PenRef pen_ref = NULL;
supplier_suiteP->NewPen(supplier_ref, &color, 1.0f, &pen_ref);

// Dash pattern: 5 pixels on, 3 pixels off
float dashes[] = {5.0f, 3.0f};
pen_suiteP->SetDashPattern(pen_ref, dashes, 2);

surface_suiteP->StrokePath(surface_ref, pen_ref, path_ref);
```

## Drawing Text

```c
// Create a font
float font_size = 0.0f;
supplier_suiteP->GetDefaultFontSize(supplier_ref, &font_size);

DRAWBOT_FontRef font_ref = NULL;
supplier_suiteP->NewDefaultFont(supplier_ref, font_size, &font_ref);

// Create a brush for the text color
DRAWBOT_ColorRGBA text_color = {1.0f, 1.0f, 1.0f, 1.0f};
DRAWBOT_BrushRef text_brush = NULL;
supplier_suiteP->NewBrush(supplier_ref, &text_color, &text_brush);

// Convert string to UTF-16 (required)
DRAWBOT_UTF16Char unicode_str[256];
// On Windows: wcscpy_s((wchar_t*)unicode_str, 256, L"Hello World");
// On Mac: use CFString conversion (see SDK examples)

DRAWBOT_PointF32 origin = {10.0f, 30.0f};

surface_suiteP->DrawString(
    surface_ref,
    text_brush,
    font_ref,
    unicode_str,
    &origin,
    kDRAWBOT_TextAlignment_Left,
    kDRAWBOT_TextTruncation_None,
    0.0f);  // truncation width (0 = no truncation)

supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)font_ref);
supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)text_brush);
```

### Text Alignment Options

| Constant | Description |
|----------|-------------|
| `kDRAWBOT_TextAlignment_Left` | Left-aligned from origin |
| `kDRAWBOT_TextAlignment_Center` | Centered on origin |
| `kDRAWBOT_TextAlignment_Right` | Right-aligned from origin |

### Text Truncation Options

| Constant | Description |
|----------|-------------|
| `kDRAWBOT_TextTruncation_None` | No truncation |
| `kDRAWBOT_TextTruncation_End` | Truncate at end |
| `kDRAWBOT_TextTruncation_EndEllipsis` | Truncate with "..." |
| `kDRAWBOT_TextTruncation_PathEllipsis` | Ellipsis in middle (preserves filename) |

## Drawing Images

```c
// Prepare pixel data (BGRA premultiplied is typically preferred)
int img_width = 64, img_height = 64;
int row_bytes = img_width * 4;
unsigned char *pixels = /* your pixel buffer */;

DRAWBOT_ImageRef image_ref = NULL;
supplier_suiteP->NewImageFromBuffer(
    supplier_ref,
    img_width, img_height, row_bytes,
    kDRAWBOT_PixelLayout_32BGRA_Premul,
    pixels,
    &image_ref);

DRAWBOT_PointF32 img_origin = {10.0f, 10.0f};
surface_suiteP->DrawImage(surface_ref, image_ref, &img_origin, 1.0f);

supplier_suiteP->ReleaseObject((DRAWBOT_ObjectRef)image_ref);
```

### Checking Preferred Pixel Layout

Different platforms may prefer different channel orders:

```c
DRAWBOT_Boolean prefers_bgra = false;
supplier_suiteP->PrefersPixelLayoutBGRA(supplier_ref, &prefers_bgra);

DRAWBOT_PixelLayout layout = prefers_bgra
    ? kDRAWBOT_PixelLayout_32BGRA_Premul
    : kDRAWBOT_PixelLayout_32ARGB_Premul;
```

## C++ RAII Wrappers

The SDK provides C++ wrapper classes that automatically manage object lifetimes:

```c
// These auto-release when they go out of scope
DRAWBOT_PenP   penP(supplier_suiteP, supplier_ref, &color, 2.0f);
DRAWBOT_BrushP brushP(supplier_suiteP, supplier_ref, &color);
DRAWBOT_PathP  pathP(supplier_suiteP, supplier_ref);
DRAWBOT_FontP  fontP(supplier_suiteP, supplier_ref, font_size);

// Use them directly as their ref types (implicit conversion)
surface_suiteP->StrokePath(surface_ref, penP, pathP);

// State stack scoper -- pushes on construction, pops on destruction
DRAWBOT_SaveAndRestoreStateStack state(surface_suiteP, surface_ref);
```

## Handling Click Events

### PF_Event_DO_CLICK

```c
typedef struct {
    A_u_long     when;
    PF_Point     screen_point;      // Mouse position in frame coords
    A_long       num_clicks;        // 1 = single, 2 = double click
    PF_Modifiers modifiers;         // Shift, Cmd/Ctrl, etc.
    A_intptr_t   continue_refcon[4]; // Your state for drag continuation
    PF_Boolean   send_drag;         // Set TRUE to receive PF_Event_DRAG
    PF_Boolean   last_time;         // TRUE on final drag message
} PF_DoClickEventInfo;
```

To initiate a drag from a click:

```c
PF_Err DoClick(PF_InData *in_data, PF_OutData *out_data,
               PF_ParamDef *params[], PF_LayerDef *output,
               PF_EventExtra *extra)
{
    PF_Err err = PF_Err_NONE;

    if (PF_Window_EFFECT == (*extra->contextH)->w_type) {
        if (PF_EA_CONTROL == extra->effect_win.area) {
            PF_Point mouse = extra->u.do_click.screen_point;

            // Hit test against your control areas
            if (PointInMyControl(mouse, extra->effect_win.current_frame)) {
                // Start a drag
                extra->u.do_click.send_drag = TRUE;
                extra->u.do_click.continue_refcon[0] = MY_DRAG_ID;
                extra->u.do_click.continue_refcon[1] = mouse.h;
                extra->u.do_click.continue_refcon[2] = mouse.v;

                extra->evt_out_flags = PF_EO_HANDLED_EVENT;
            }
        }
    }
    return err;
}
```

### PF_Event_DRAG

Drag events reuse `PF_DoClickEventInfo`. Your `continue_refcon` values persist across the drag:

```c
PF_Err DoDrag(PF_InData *in_data, PF_OutData *out_data,
              PF_ParamDef *params[], PF_LayerDef *output,
              PF_EventExtra *extra)
{
    PF_Err err = PF_Err_NONE;
    PF_Point mouse = extra->u.do_click.screen_point;

    if (extra->u.do_click.continue_refcon[0] == MY_DRAG_ID) {
        // Calculate delta from initial click
        A_long dx = mouse.h - (A_long)extra->u.do_click.continue_refcon[1];

        // Update parameter based on drag
        params[MY_PARAM]->u.fs_d.value += dx * 0.5;
        params[MY_PARAM]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;

        // Continue dragging
        extra->u.do_click.send_drag = TRUE;
        extra->u.do_click.continue_refcon[1] = mouse.h;

        if (extra->u.do_click.last_time) {
            // Drag ended
            extra->u.do_click.send_drag = FALSE;
        }

        extra->evt_out_flags = PF_EO_HANDLED_EVENT;
    }
    return err;
}
```

> **Important**: When you set `PF_ChangeFlag_CHANGED_VALUE` on a parameter during drag, AE creates an undoable change. The initial DO_CLICK and subsequent DRAGs are grouped into a single undo step.

## Handling Cursor Adjustment

`PF_Event_ADJUST_CURSOR` is sent whenever the mouse moves over your custom UI area:

```c
typedef struct {
    PF_Point     screen_point;  // Current mouse position
    PF_Modifiers modifiers;     // Current modifier keys
    PF_CursorType set_cursor;   // Set this to change the cursor
} PF_AdjustCursorEventInfo;
```

```c
PF_Err AdjustCursor(PF_InData *in_data, PF_OutData *out_data,
                    PF_ParamDef *params[], PF_LayerDef *output,
                    PF_EventExtra *extra)
{
    // Change cursor based on modifier keys
    if (extra->u.adjust_cursor.modifiers & PF_Mod_SHIFT_KEY) {
        extra->u.adjust_cursor.set_cursor = PF_Cursor_EYEDROPPER;
    } else if (extra->u.adjust_cursor.modifiers & PF_Mod_CMD_CTRL_KEY) {
        extra->u.adjust_cursor.set_cursor = PF_Cursor_CROSSHAIRS;
    } else {
        extra->u.adjust_cursor.set_cursor = PF_Cursor_ARROW;
    }

    extra->evt_out_flags = PF_EO_HANDLED_EVENT;
    return PF_Err_NONE;
}
```

### Cursor Type Constants (Selection)

| Cursor | Description |
|--------|-------------|
| `PF_Cursor_NONE` | Do not override (let AE decide) |
| `PF_Cursor_CUSTOM` | You set the cursor yourself via OS calls |
| `PF_Cursor_ARROW` | Standard arrow |
| `PF_Cursor_CROSSHAIRS` | Crosshair |
| `PF_Cursor_EYEDROPPER` | Eyedropper |
| `PF_Cursor_HAND` | Hand (pan) |
| `PF_Cursor_PEN` | Pen tool |
| `PF_Cursor_MAGNIFY` | Magnifying glass |
| `PF_Cursor_FINGER_POINTER` | Pointing finger |

### Modifier Key Constants

| Modifier | Description |
|----------|-------------|
| `PF_Mod_NONE` | No modifiers |
| `PF_Mod_CMD_CTRL_KEY` | Cmd (Mac) / Ctrl (Windows) |
| `PF_Mod_SHIFT_KEY` | Shift |
| `PF_Mod_CAPS_LOCK_KEY` | Caps Lock |
| `PF_Mod_OPT_ALT_KEY` | Option (Mac) / Alt (Windows) |
| `PF_Mod_MAC_CONTROL_KEY` | Control key (Mac only) |

## Event Output Flags

Set `evt_out_flags` before returning from your event handler:

| Flag | Value | Meaning |
|------|-------|---------|
| `PF_EO_NONE` | 0 | Did not handle the event |
| `PF_EO_HANDLED_EVENT` | 1 << 0 | Event was handled; AE will not process it further |
| `PF_EO_ALWAYS_UPDATE` | 1 << 1 | Force a re-render of the composition |
| `PF_EO_NEVER_UPDATE` | 1 << 2 | Suppress re-render |
| `PF_EO_UPDATE_NOW` | 1 << 3 | Update the view immediately (with `PF_InvalidateRect`) |

## Surface State Management

When modifying surface properties (clipping, transforms, anti-aliasing), always push and pop the state stack:

```c
surface_suiteP->PushStateStack(surface_ref);

// Modify surface state
surface_suiteP->SetAntiAliasPolicy(surface_ref, kDRAWBOT_AntiAliasPolicy_Med);

DRAWBOT_Rect32 clip_rect = {10, 10, 200, 100};
surface_suiteP->Clip(surface_ref, supplier_ref, &clip_rect);

// Draw within the modified state
surface_suiteP->FillPath(surface_ref, brush_ref, path_ref,
                          kDRAWBOT_FillType_Default);

// Restore previous state
surface_suiteP->PopStateStack(surface_ref);
```

Or use the C++ RAII scoper:

```c
{
    DRAWBOT_SaveAndRestoreStateStack state(surface_suiteP, surface_ref);
    surface_suiteP->SetAntiAliasPolicy(surface_ref, kDRAWBOT_AntiAliasPolicy_High);
    // ... draw ...
}  // state automatically restored here
```

## Coordinate System in the ECW

In the ECW, coordinates in `effect_win.current_frame` are in screen pixels relative to the effect panel. The center of a pixel is at `(x + 0.5, y + 0.5)` in the Drawbot drawing model:

```c
// Correct: offset by 0.5 for crisp lines
rectR.left = event_extra->effect_win.current_frame.left + 0.5f;
rectR.top  = event_extra->effect_win.current_frame.top + 0.5f;
```

> **Pitfall**: If your lines look blurry, you are probably drawing at integer coordinates. Add 0.5 to align with pixel centers for 1px-wide crisp lines.

## PF_PUI Flags Reference

These flags on `PF_ParamDef.ui_flags` control how parameters interact with custom UI:

| Flag | Value | Description |
|------|-------|-------------|
| `PF_PUI_TOPIC` | 1 << 0 | Custom drawing in the title row (always visible) |
| `PF_PUI_CONTROL` | 1 << 1 | Custom drawing in the control area (below twirly) |
| `PF_PUI_STD_CONTROL_ONLY` | 1 << 2 | Standard UI control with no data stream |
| `PF_PUI_NO_ECW_UI` | 1 << 3 | Hide parameter from ECW entirely |
| `PF_PUI_ECW_SEPARATOR` | 1 << 4 | Draw thick line above this parameter |
| `PF_PUI_DISABLED` | 1 << 5 | Gray out the parameter UI |
| `PF_PUI_DONT_ERASE_TOPIC` | 1 << 6 | AE will not clear the topic area before drawing |
| `PF_PUI_DONT_ERASE_CONTROL` | 1 << 7 | AE will not clear the control area before drawing |
| `PF_PUI_INVISIBLE` | 1 << 9 | Hide parameter from both ECW and Timeline |

When using `PF_PUI_CONTROL`, set `ui_width` and `ui_height` on the `PF_ParamDef`:

```c
def.ui_flags  = PF_PUI_CONTROL;
def.ui_width  = 200;
def.ui_height = 100;
```

## Getting App Theme Colors

To make your custom UI blend with AE's current brightness theme:

```c
AEGP_SuiteHandler suites(in_data->pica_basicP);

PF_App_Color app_color;
suites.AppSuite6()->PF_AppGetColor(PF_App_Color_FILL, &app_color);
// Convert to DRAWBOT_ColorRGBA as needed
```

## Common Pitfalls

1. **Memory leaks from unreleased Drawbot objects.** Every `NewPen`, `NewBrush`, `NewPath`, `NewDefaultFont`, `NewImageFromBuffer` must have a matching `ReleaseObject`. Use C++ RAII wrappers to avoid this.

2. **Not setting `PF_EO_HANDLED_EVENT`.** If you draw but forget to set this flag, AE may overdraw your content or process the event twice.

3. **Drawing outside your frame bounds.** In the ECW, always respect `effect_win.current_frame`. Drawing outside this rectangle will corrupt neighboring parameter UI.

4. **Handling click events for the wrong window type.** An effect with both ECW and comp overlay UI will receive events from both windows. Always check `(*contextH)->w_type` before processing.

5. **Not checking for NULL `drawing_ref`.** `PF_GetDrawingReference` can return NULL. Always verify before attempting to draw.

6. **Thread safety with MFR.** As of AE 13.5 and later, UI selectors run on the main thread but render selectors can run on any thread. Do not access UI state from render calls. With `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`, `sequence_data` is read-only during render.
