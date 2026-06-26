# Overlay UI: Drawing on the Composition Panel

This document covers how to draw interactive overlays on the Composition and Layer panels -- the handles, outlines, and controls that appear directly over the video output. This is distinct from ECW (Effect Controls Window) custom UI, which draws in the parameter panel.

## When to Use Overlay UI

Overlay UI is used for:
- Draggable control points (like the center point of a blur)
- Bounding boxes and outlines showing effect regions
- On-screen handles for adjusting radius, angle, or other spatial parameters
- Visual guides, grids, or wireframes overlaid on the composition

The AE SDK examples CCU (Comp Composition UI) demonstrates all of these patterns.

## Registration

To receive overlay events, register for comp and/or layer window events during `PF_Cmd_PARAM_SETUP`:

```c
// In GlobalSetup:
out_data->out_flags |= PF_OutFlag_CUSTOM_UI;

// In ParamsSetup, after adding all parameters:
PF_CustomUIInfo ci;
AEFX_CLR_STRUCT(ci);

ci.events = PF_CustomEFlag_COMP | PF_CustomEFlag_LAYER;

// Width/height of 0 means the overlay covers the entire panel
ci.comp_ui_width     = 0;
ci.comp_ui_height    = 0;
ci.comp_ui_alignment = PF_UIAlignment_NONE;

ci.layer_ui_width    = 0;
ci.layer_ui_height   = 0;
ci.layer_ui_alignment = PF_UIAlignment_NONE;

err = in_data->inter.register_ui(in_data->effect_ref, &ci);
```

### Event Flags

| Flag | Events Received |
|------|----------------|
| `PF_CustomEFlag_COMP` | Draw/click/drag/cursor events in the Composition panel |
| `PF_CustomEFlag_LAYER` | Draw/click/drag/cursor events in the Layer panel |
| `PF_CustomEFlag_COMP \| PF_CustomEFlag_LAYER` | Events in both panels |

You can combine these with `PF_CustomEFlag_EFFECT` if you also want ECW custom UI.

## Drawing Overlays

### The Drawing Pipeline

When drawing overlays, you must transform coordinates from layer space (where your parameters live) to frame space (where the panel draws). The transform chain differs by window type:

**Comp panel**: Layer Space -> Comp Space -> Frame Space
**Layer panel**: Layer Space -> Frame Space

```c
PF_Err DrawEvent(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_LayerDef     *output,
    PF_EventExtra   *event_extraP)
{
    PF_Err err = PF_Err_NONE;

    PF_WindowType w_type = (*event_extraP->contextH)->w_type;

    // Only draw in comp or layer windows
    if (w_type != PF_Window_COMP && w_type != PF_Window_LAYER) {
        return err;
    }

    AEGP_SuiteHandler suites(in_data->pica_basicP);

    DRAWBOT_DrawRef drawing_ref = NULL;
    ERR(suites.EffectCustomUISuite1()->PF_GetDrawingReference(
        event_extraP->contextH, &drawing_ref));

    if (!err && drawing_ref) {
        // Transform and draw your overlay
        DrawMyOverlay(in_data, out_data, params, event_extraP, drawing_ref);
    }

    event_extraP->evt_out_flags = PF_EO_HANDLED_EVENT;
    return err;
}
```

### Transforming Points from Layer to Frame

Here is the core pattern used by the CCU SDK example. Every point you want to draw must be transformed through the appropriate chain:

```c
// Transform a single point from layer space to screen/frame coordinates
void LayerPointToFrame(
    PF_InData       *in_data,
    PF_EventExtra   *event_extraP,
    PF_FpLong       layer_x,
    PF_FpLong       layer_y,
    float           *frame_x,
    float           *frame_y)
{
    PF_FixedPoint pt;
    pt.x = FLOAT2FIX(layer_x);
    pt.y = FLOAT2FIX(layer_y);

    // Step 1: If comp window, transform layer -> comp
    if (PF_Window_COMP == (*event_extraP->contextH)->w_type) {
        event_extraP->cbs.layer_to_comp(
            event_extraP->cbs.refcon,
            event_extraP->contextH,
            in_data->current_time,
            in_data->time_scale,
            &pt);
    }

    // Step 2: Transform to frame (view) coordinates
    event_extraP->cbs.source_to_frame(
        event_extraP->cbs.refcon,
        event_extraP->contextH,
        &pt);

    *frame_x = FIX_2_FLOAT(pt.x);
    *frame_y = FIX_2_FLOAT(pt.y);
}
```

### Drawing a Complete Overlay Example

This example draws a rectangle with corner handles at the position defined by effect parameters:

```c
void DrawMyOverlay(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_EventExtra   *event_extraP,
    DRAWBOT_DrawRef drawing_ref)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    DRAWBOT_Suites db;
    AEFX_AcquireDrawbotSuites(in_data, out_data, &db);

    DRAWBOT_SupplierRef supplier_ref = NULL;
    DRAWBOT_SurfaceRef  surface_ref  = NULL;
    db.drawbot_suiteP->GetSupplier(drawing_ref, &supplier_ref);
    db.drawbot_suiteP->GetSurface(drawing_ref, &surface_ref);

    // Get parameter values (layer-space coordinates)
    PF_FpLong cx = FIX_2_FLOAT(params[MY_CENTER]->u.td.x_value);
    PF_FpLong cy = FIX_2_FLOAT(params[MY_CENTER]->u.td.y_value);
    PF_FpLong radius = params[MY_RADIUS]->u.fs_d.value;

    // Define corners in layer space
    struct { PF_FpLong x, y; } corners[4] = {
        {cx - radius, cy - radius},
        {cx + radius, cy - radius},
        {cx + radius, cy + radius},
        {cx - radius, cy + radius}
    };

    // Transform all corners to frame space
    float frame_pts[4][2];
    for (int i = 0; i < 4; i++) {
        LayerPointToFrame(in_data, event_extraP,
                          corners[i].x, corners[i].y,
                          &frame_pts[i][0], &frame_pts[i][1]);
    }

    // Draw the bounding rectangle using OverlayTheme suite
    DRAWBOT_PathP pathP(db.supplier_suiteP, supplier_ref);
    db.path_suiteP->MoveTo(pathP, frame_pts[0][0], frame_pts[0][1]);
    db.path_suiteP->LineTo(pathP, frame_pts[1][0], frame_pts[1][1]);
    db.path_suiteP->LineTo(pathP, frame_pts[2][0], frame_pts[2][1]);
    db.path_suiteP->LineTo(pathP, frame_pts[3][0], frame_pts[3][1]);
    db.path_suiteP->Close(pathP);

    // Stroke with themed colors (adapts to app brightness)
    suites.EffectCustomUIOverlayThemeSuite1()->PF_StrokePath(
        drawing_ref, pathP, TRUE);  // TRUE = also draw shadow

    // Draw handles at each corner
    for (int i = 0; i < 4; i++) {
        A_FloatPoint center = {frame_pts[i][0], frame_pts[i][1]};
        suites.EffectCustomUIOverlayThemeSuite1()->PF_FillVertex(
            drawing_ref, &center, TRUE);
    }

    AEFX_ReleaseDrawbotSuites(in_data, out_data);
}
```

## PF_EffectCustomUIOverlayThemeSuite

This suite provides drawing functions that match AE's native overlay style, automatically adapting to the user's brightness preference. Use this suite for all comp/layer window overlays to ensure visual consistency with AE's built-in tools.

### Suite Definition

```c
#define kPFEffectCustomUIOverlayThemeSuite             "PF Effect Custom UI Overlay Theme Suite"
#define kPFEffectCustomUIOverlayThemeSuiteVersion1     1  // frozen in AE 10.0
```

### Functions

#### Getting Theme Colors and Sizes

```c
// Get the foreground color for overlay drawing
DRAWBOT_ColorRGBA fg_color;
suites.EffectCustomUIOverlayThemeSuite1()->PF_GetPreferredForegroundColor(&fg_color);

// Get the shadow color (for drop-shadow behind overlays)
DRAWBOT_ColorRGBA shadow_color;
suites.EffectCustomUIOverlayThemeSuite1()->PF_GetPreferredShadowColor(&shadow_color);

// Get the stroke width used for both foreground and shadow strokes
float stroke_width;
suites.EffectCustomUIOverlayThemeSuite1()->PF_GetPreferredStrokeWidth(&stroke_width);

// Get the size of vertex handles (squares at control points)
float vertex_size;
suites.EffectCustomUIOverlayThemeSuite1()->PF_GetPreferredVertexSize(&vertex_size);

// Get the offset for shadow drawing
A_LPoint shadow_offset;
suites.EffectCustomUIOverlayThemeSuite1()->PF_GetPreferredShadowOffset(&shadow_offset);
```

#### Themed Drawing Functions

These functions draw using the theme colors and stroke widths automatically:

```c
// Stroke a path with themed colors
// draw_shadowB: TRUE to draw a shadow beneath the stroke
PF_StrokePath(drawing_ref, path_ref, draw_shadowB);

// Fill a path with themed colors
PF_FillPath(drawing_ref, path_ref, draw_shadowB);

// Draw a square vertex handle (the small squares at control points)
A_FloatPoint center = {x, y};
PF_FillVertex(drawing_ref, &center, draw_shadowB);
```

When `draw_shadowB` is `TRUE`, the function draws a shadow-colored version offset by `PF_GetPreferredShadowOffset` before drawing the foreground. This gives overlays a readable appearance over both light and dark backgrounds.

> **Pitfall**: `PF_EffectCustomUIOverlayThemeSuite` is not supported in Premiere Pro. Check `in_data->appl_id != 'PrMr'` before using it, and fall back to manual Drawbot colors for Premiere.

## Handling Mouse Clicks on Overlays

### Hit Testing

Mouse coordinates arrive in frame space (`screen_point`). To check if a click hit your overlay controls, either:

1. Transform your control positions to frame space and compare, or
2. Transform the mouse position to layer space and compare there

The first approach is simpler for hit testing:

```c
PF_Err DoClick(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_LayerDef     *output,
    PF_EventExtra   *event_extraP)
{
    PF_Err err = PF_Err_NONE;

    PF_WindowType w_type = (*event_extraP->contextH)->w_type;
    if (w_type != PF_Window_COMP && w_type != PF_Window_LAYER) {
        return err;
    }

    PF_Point mouse = event_extraP->u.do_click.screen_point;

    // Get control point position in layer space
    PF_FpLong cx = FIX_2_FLOAT(params[MY_CENTER]->u.td.x_value);
    PF_FpLong cy = FIX_2_FLOAT(params[MY_CENTER]->u.td.y_value);

    // Transform to frame space
    float frame_cx, frame_cy;
    LayerPointToFrame(in_data, event_extraP, cx, cy, &frame_cx, &frame_cy);

    // Hit test with slop (tolerance)
    float dx = (float)mouse.h - frame_cx;
    float dy = (float)mouse.v - frame_cy;
    float dist = fabsf(dx) + fabsf(dy);  // Manhattan distance

    if (dist < 6.0f) {  // 6-pixel hit slop
        // Start drag
        event_extraP->u.do_click.send_drag = TRUE;
        event_extraP->u.do_click.continue_refcon[0] = 1;  // drag ID
        event_extraP->evt_out_flags = PF_EO_HANDLED_EVENT;
    }

    return err;
}
```

### Processing Drags

During a drag in the comp/layer window, convert the mouse position back to layer space to update parameters:

```c
PF_Err DoDrag(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_LayerDef     *output,
    PF_EventExtra   *event_extraP)
{
    PF_Err err = PF_Err_NONE;

    if (event_extraP->u.do_click.continue_refcon[0] != 1) {
        return err;
    }

    PF_Point mouse = event_extraP->u.do_click.screen_point;

    // Convert frame coordinates back to layer space
    PF_FixedPoint layer_pt;
    layer_pt.x = INT2FIX(mouse.h);
    layer_pt.y = INT2FIX(mouse.v);

    // Frame -> source
    event_extraP->cbs.frame_to_source(
        event_extraP->cbs.refcon,
        event_extraP->contextH,
        &layer_pt);

    // If comp window, also comp -> layer
    if (PF_Window_COMP == (*event_extraP->contextH)->w_type) {
        event_extraP->cbs.comp_to_layer(
            event_extraP->cbs.refcon,
            event_extraP->contextH,
            in_data->current_time,
            in_data->time_scale,
            &layer_pt);
    }

    // Update the point parameter
    params[MY_CENTER]->u.td.x_value = layer_pt.x;
    params[MY_CENTER]->u.td.y_value = layer_pt.y;
    params[MY_CENTER]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;

    // Continue drag
    event_extraP->u.do_click.send_drag = TRUE;

    if (event_extraP->u.do_click.last_time) {
        event_extraP->u.do_click.send_drag = FALSE;
    }

    event_extraP->evt_out_flags = PF_EO_HANDLED_EVENT;
    return err;
}
```

## Using the Layer-to-Comp Matrix Directly

For complex overlays that need to transform many points, getting the matrix once is more efficient than calling `layer_to_comp` per point:

```c
PF_FloatMatrix l2c;
event_extraP->cbs.get_layer2comp_xform(
    event_extraP->cbs.refcon,
    event_extraP->contextH,
    in_data->current_time,
    in_data->time_scale,
    &l2c);

// Transform a point manually using the 3x3 matrix
// l2c.mat[row][col] -- affine 3x3
float src_x = 100.0f, src_y = 200.0f;
float dst_x = l2c.mat[0][0] * src_x + l2c.mat[0][1] * src_y + l2c.mat[0][2];
float dst_y = l2c.mat[1][0] * src_x + l2c.mat[1][1] * src_y + l2c.mat[1][2];
```

You can also apply a Drawbot surface transform for direct drawing in layer space:

```c
surface_suiteP->PushStateStack(surface_ref);

// Set up the transform so you can draw in layer space directly
DRAWBOT_MatrixF32 xform;
// ... fill xform from the l2c matrix and source_to_frame ...
surface_suiteP->Transform(surface_ref, &xform);

// Now draw in layer space -- coordinates are automatically transformed
// ... draw paths, shapes, text ...

surface_suiteP->PopStateStack(surface_ref);
```

## Checking the PF_EI_DONT_DRAW Flag

AE may send draw events where it only wants hit testing but no actual drawing (for optimization). Check `evt_in_flags`:

```c
if (event_extraP->evt_in_flags & PF_EI_DONT_DRAW) {
    // Skip all drawing, but still handle hit tests
    return err;
}
```

This flag is described in the header as: "don't draw controls in comp or layer window."

## Displaying Information in the Info Panel

The event callbacks include functions for displaying text in AE's info panel (the bar at the bottom showing cursor position, color values, etc.):

```c
// Display two lines of text
event_extraP->cbs.info_draw_text(
    event_extraP->cbs.refcon,
    "X: 100.0  Y: 200.0",   // line 1
    "Radius: 50.0");          // line 2

// Display a color swatch
PF_Pixel color = {255, 128, 64, 255};  // ARGB
event_extraP->cbs.info_draw_color(event_extraP->cbs.refcon, color);
```

For three lines and formatted text, use `PF_AdvAppSuite`:

```c
suites.AdvAppSuite2()->PF_InfoDrawText3(
    "Line 1",
    "Line 2",
    "Line 3");
```

## Async Manager for Custom UI (AE 13.5+)

Since AE 13.5, the UI thread and render thread are separated. To display rendered frames in your custom UI overlay (such as a preview thumbnail), use the async render manager:

1. Set `PF_OutFlag2_CUSTOM_UI_ASYNC_MANAGER` during `PF_Cmd_GLOBAL_SETUP`
2. During `PF_Event_DRAW`, request async renders
3. AE calls `PF_Event_DRAW` again when renders complete

```c
// Get the async manager
PF_AsyncManagerP manager = NULL;
suites.EffectCustomUISuite2()->PF_GetContextAsyncManager(
    in_data, event_extraP, &manager);

if (manager) {
    // Request frames and draw what's available
    // The manager automatically cancels stale requests
}
```

> **Important**: Do not render synchronously from the UI thread in AE 13.5+. This can cause deadlocks. Always use the async manager for frame rendering in custom UI.

## Complete Event Dispatch Pattern

Here is the recommended top-level event handler that correctly dispatches for both overlay and ECW events:

```c
PF_Err HandleEvent(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_LayerDef     *output,
    PF_EventExtra   *event_extraP)
{
    PF_Err err = PF_Err_NONE;

    // Handle initial click -> drag transition
    if (PF_Event_DO_CLICK == event_extraP->e_type) {
        if (event_extraP->u.do_click.send_drag) {
            event_extraP->e_type = PF_Event_DRAG;
        }
    }

    switch (event_extraP->e_type) {

        case PF_Event_NEW_CONTEXT:
            // Allocate per-context state if needed
            break;

        case PF_Event_ACTIVATE:
            break;

        case PF_Event_DO_CLICK:
            err = DoClick(in_data, out_data, params, output, event_extraP);
            break;

        case PF_Event_DRAG:
            err = DoDrag(in_data, out_data, params, output, event_extraP);
            break;

        case PF_Event_DRAW:
            err = DrawEvent(in_data, out_data, params, output, event_extraP);
            break;

        case PF_Event_ADJUST_CURSOR:
            err = AdjustCursor(in_data, out_data, params, output, event_extraP);
            break;

        case PF_Event_DEACTIVATE:
            break;

        case PF_Event_CLOSE_CONTEXT:
            // Free per-context state
            break;

        case PF_Event_MOUSE_EXITED:
            // Mouse left the comp/layer window
            break;

        default:
            break;
    }

    return err;
}
```

## Common Pitfalls

1. **Drawing without transforming coordinates.** Layer-space coordinates are not frame-space coordinates. Your overlay will appear in the wrong position (or not at all) if you draw raw parameter values without running them through `layer_to_comp` and `source_to_frame`.

2. **Using the wrong transform for the window.** In the layer window, `layer_to_comp` is unnecessary and will produce incorrect results. Always check `(*contextH)->w_type`.

3. **Not accounting for pixel aspect ratio.** When converting scalar distances (like a radius), multiply horizontal distances by `in_data->pixel_aspect_ratio.den / in_data->pixel_aspect_ratio.num` to display correctly. The CCU example demonstrates this.

4. **Forgetting PF_EO_HANDLED_EVENT on draw.** If you draw but don't set this flag, AE may overdraw or ignore your overlay.

5. **Using OverlayThemeSuite in Premiere Pro.** This suite is not supported in Premiere. Check `in_data->appl_id` before using it.

6. **Synchronous render calls in custom UI.** Since AE 13.5, never call blocking render functions from the UI thread. Use `PF_AsyncManagerP` instead.

7. **Treating `send_drag` incorrectly.** The first time `PF_Event_DO_CLICK` is sent with `send_drag` already TRUE, it means a drag continuation (the SDK re-dispatches the event). The CCU example checks this and routes to the drag handler.
