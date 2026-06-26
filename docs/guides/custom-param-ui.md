# Custom Parameter UI: Arbitrary Data and Custom Controls

This document covers `PF_Param_ARBITRARY_DATA`, the callback system for managing arbitrary data, and how to combine arb data with custom UI controls in the Effect Controls Window.

## What is Arbitrary Data?

`PF_Param_ARBITRARY_DATA` (commonly called "arb data") is a parameter type that lets you store any custom data structure as a keyframeable parameter. Unlike built-in types (sliders, colors, points), arb data has no standard UI -- you define its data format, serialization, interpolation, and (optionally) visual representation.

Common uses:
- Color palettes (like the ColorGrid SDK example)
- Complex multi-value settings that don't fit standard parameter types
- Curve editors (bezier control point arrays)
- Custom enum/struct combinations that must be keyframed as a unit

## Adding an Arbitrary Data Parameter

During `PF_Cmd_PARAM_SETUP`, add the parameter with `PF_ADD_ARBITRARY`:

```c
// Define a default instance of your data
PF_ArbitraryH default_arbH = NULL;
err = CreateDefaultArb(in_data, out_data, &default_arbH);

if (!err) {
    PF_ParamDef def;
    AEFX_CLR_STRUCT(def);

    def.param_type = PF_Param_ARBITRARY_DATA;

    // The arb-specific fields
    def.u.arb_d.id       = MY_ARB_ID;    // Your identifier (A_short)
    def.u.arb_d.pad      = 0;
    def.u.arb_d.dephault = default_arbH;  // Host takes ownership
    def.u.arb_d.value    = NULL;          // Always NULL at add time

    // UI flags -- MUST set one of these in AE:
    // PF_PUI_CONTROL  - you will draw a custom control
    // PF_PUI_TOPIC    - you will draw in the title area
    // PF_PUI_NO_ECW_UI - no ECW UI at all
    def.ui_flags  = PF_PUI_CONTROL;
    def.ui_width  = 200;  // Width of your custom control area
    def.ui_height = 100;  // Height of your custom control area

    // Supervise to get USER_CHANGED_PARAM notifications
    def.flags = PF_ParamFlag_SUPERVISE;

    strncpy(def.name, "My Custom Data", PF_MAX_EFFECT_PARAM_NAME_LEN);

    err = PF_ADD_PARAM(in_data, -1, &def);
}
```

> **Important**: In AE, `PF_Param_ARBITRARY_DATA` requires at least one of `PF_PUI_CONTROL`, `PF_PUI_TOPIC`, or `PF_PUI_NO_ECW_UI`. Omitting all three causes undefined behavior. In Premiere Pro 8.0+, you can omit these flags to get a keyframe track without custom UI.

## The Arb Data Callback System

When AE needs to manipulate your arb data (copy, serialize, interpolate, etc.), it sends `PF_Cmd_ARBITRARY_CALLBACK` with a `PF_ArbParamsExtra*` in the `extra` parameter. You dispatch based on `which_function`:

```c
PF_Err HandleArbitraryCB(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ParamDef         *params[],
    PF_LayerDef         *output,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;

    switch (arbExtra->which_function) {
        case PF_Arbitrary_NEW_FUNC:
            err = ArbNew(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_DISPOSE_FUNC:
            err = ArbDispose(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_COPY_FUNC:
            err = ArbCopy(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_FLAT_SIZE_FUNC:
            err = ArbFlatSize(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_FLATTEN_FUNC:
            err = ArbFlatten(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_UNFLATTEN_FUNC:
            err = ArbUnflatten(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_INTERP_FUNC:
            err = ArbInterpolate(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_COMPARE_FUNC:
            err = ArbCompare(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_PRINT_SIZE_FUNC:
            err = ArbPrintSize(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_PRINT_FUNC:
            err = ArbPrint(in_data, out_data, arbExtra);
            break;
        case PF_Arbitrary_SCAN_FUNC:
            err = ArbScan(in_data, out_data, arbExtra);
            break;
    }
    return err;
}
```

### The PF_ArbParamsExtra Structure

```c
typedef struct {
    A_short              id;              // Matches the id you set in PF_ArbitraryDef
    A_short              padding;
    PF_FunctionSelector  which_function;  // Which callback to execute
    // Union of per-callback parameter structs:
    union { ... } u;
} PF_ArbParamsExtra;
```

## Implementing Each Callback

### PF_Arbitrary_NEW_FUNC

Allocate and initialize a new instance of your data. Called when AE needs a fresh instance.

> **Historical note**: In post-CS6 AE, `NEW_FUNC` is rarely called directly. Instead, AE calls `COPY_FUNC` with your default arb handle from `PF_ADD_PARAM`. However, you should still implement it for safety and backward compatibility.

```c
PF_Err ArbNew(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    PF_ArbitraryH *arbPH = arbExtra->u.new_func_params.arbPH;

    *arbPH = suites.HandleSuite1()->host_new_handle(sizeof(MyArbData));
    if (*arbPH) {
        MyArbData *dataP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(*arbPH);
        if (dataP) {
            AEFX_CLR_STRUCT(*dataP);
            // Initialize with defaults
            dataP->value1 = 0.5;
            dataP->value2 = 1.0;
            suites.HandleSuite1()->host_unlock_handle(*arbPH);
        } else {
            err = PF_Err_OUT_OF_MEMORY;
        }
    } else {
        err = PF_Err_OUT_OF_MEMORY;
    }
    return err;
}
```

### PF_Arbitrary_DISPOSE_FUNC

Free a previously allocated arb handle.

```c
PF_Err ArbDispose(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    PF_ArbitraryH arbH = arbExtra->u.dispose_func_params.arbH;
    if (arbH) {
        suites.HandleSuite1()->host_dispose_handle(arbH);
    }
    return PF_Err_NONE;
}
```

### PF_Arbitrary_COPY_FUNC

Duplicate arb data from source to destination. The **destination handle is already allocated** by the caller -- you just need to copy the contents.

```c
PF_Err ArbCopy(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    PF_ArbitraryH srcH  = arbExtra->u.copy_func_params.src_arbH;
    PF_ArbitraryH *dstPH = arbExtra->u.copy_func_params.dst_arbPH;

    if (srcH && dstPH) {
        MyArbData *srcP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(srcH);
        if (srcP) {
            PF_ArbitraryH dstH = *dstPH;
            if (dstH) {
                MyArbData *dstP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(dstH);
                if (dstP) {
                    memcpy(dstP, srcP, sizeof(MyArbData));
                    suites.HandleSuite1()->host_unlock_handle(dstH);
                } else {
                    err = PF_Err_OUT_OF_MEMORY;
                }
            }
            suites.HandleSuite1()->host_unlock_handle(srcH);
        }
    }
    return err;
}
```

> **Pitfall**: Do NOT allocate the destination handle yourself in COPY_FUNC. The `*dst_arbPH` already points to a valid handle allocated by the host. Just lock, memcpy, and unlock.

### PF_Arbitrary_FLAT_SIZE_FUNC

Report how many bytes your flattened data will occupy. Used for serialization to disk.

```c
PF_Err ArbFlatSize(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    // If your data has no pointers, flat size == struct size
    *(arbExtra->u.flat_size_func_params.flat_data_sizePLu) = sizeof(MyArbData);
    return PF_Err_NONE;
}
```

If your data contains dynamically-sized elements (like a variable-length array), calculate the actual serialized size here.

### PF_Arbitrary_FLATTEN_FUNC

Serialize your arb data into a flat byte buffer for saving to disk. The buffer is pre-allocated to the size you reported in `FLAT_SIZE_FUNC`.

```c
PF_Err ArbFlatten(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    PF_ArbitraryH arbH = arbExtra->u.flatten_func_params.arbH;
    void *bufP          = arbExtra->u.flatten_func_params.flat_dataPV;
    A_u_long buf_size   = arbExtra->u.flatten_func_params.buf_sizeLu;

    if (arbH && bufP) {
        MyArbData *dataP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(arbH);
        if (dataP) {
            // For POD structs, a simple memcpy works
            memcpy(bufP, dataP, MIN(buf_size, sizeof(MyArbData)));
            suites.HandleSuite1()->host_unlock_handle(arbH);
        }
    }
    return err;
}
```

> **Important**: If your struct layout might change between plugin versions, add a version number at the start of the flattened data so `UNFLATTEN_FUNC` can handle old formats.

### PF_Arbitrary_UNFLATTEN_FUNC

Deserialize from a flat buffer back into a live arb handle. You must allocate the output handle.

```c
PF_Err ArbUnflatten(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    const void *flatP   = arbExtra->u.unflatten_func_params.flat_dataPV;
    A_u_long buf_size   = arbExtra->u.unflatten_func_params.buf_sizeLu;
    PF_ArbitraryH *arbPH = arbExtra->u.unflatten_func_params.arbPH;

    *arbPH = suites.HandleSuite1()->host_new_handle(sizeof(MyArbData));
    if (*arbPH) {
        MyArbData *dataP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(*arbPH);
        if (dataP) {
            AEFX_CLR_STRUCT(*dataP);
            memcpy(dataP, flatP, MIN(buf_size, sizeof(MyArbData)));
            suites.HandleSuite1()->host_unlock_handle(*arbPH);
        } else {
            err = PF_Err_OUT_OF_MEMORY;
        }
    } else {
        err = PF_Err_OUT_OF_MEMORY;
    }
    return err;
}
```

### PF_Arbitrary_INTERP_FUNC

Interpolate between two keyframe values. `tF` ranges from 0.0 (left keyframe) to 1.0 (right keyframe). Velocity curves are already factored into `tF`.

```c
PF_Err ArbInterpolate(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    PF_ArbitraryH leftH    = arbExtra->u.interp_func_params.left_arbH;
    PF_ArbitraryH rightH   = arbExtra->u.interp_func_params.right_arbH;
    PF_FpLong     t         = arbExtra->u.interp_func_params.tF;
    PF_ArbitraryH *resultPH = arbExtra->u.interp_func_params.interpPH;

    MyArbData *leftP  = (MyArbData*)suites.HandleSuite1()->host_lock_handle(leftH);
    MyArbData *rightP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(rightH);
    MyArbData *outP   = (MyArbData*)suites.HandleSuite1()->host_lock_handle(*resultPH);

    if (leftP && rightP && outP) {
        // Linear interpolation
        outP->value1 = leftP->value1 + t * (rightP->value1 - leftP->value1);
        outP->value2 = leftP->value2 + t * (rightP->value2 - leftP->value2);

        // For non-numeric data, you might snap or blend differently
    }

    suites.HandleSuite1()->host_unlock_handle(*resultPH);
    suites.HandleSuite1()->host_unlock_handle(rightH);
    suites.HandleSuite1()->host_unlock_handle(leftH);

    return err;
}
```

> **Tip**: If your arb data is not meaningfully interpolatable, you can copy the left keyframe for `t < 0.5` and the right keyframe for `t >= 0.5` (step interpolation). But set `PF_ParamFlag_CANNOT_INTERP` on the parameter to signal this to the user.

### PF_Arbitrary_COMPARE_FUNC

Compare two arb data instances. Used by AE for caching and undo decisions.

```c
PF_Err ArbCompare(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    PF_ArbitraryH aH = arbExtra->u.compare_func_params.a_arbH;
    PF_ArbitraryH bH = arbExtra->u.compare_func_params.b_arbH;
    PF_ArbCompareResult *resultP = arbExtra->u.compare_func_params.compareP;

    MyArbData *aP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(aH);
    MyArbData *bP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(bH);

    if (aP && bP) {
        if (memcmp(aP, bP, sizeof(MyArbData)) == 0) {
            *resultP = PF_ArbCompare_EQUAL;
        } else {
            // For ordered comparison (used in interpolation):
            // Calculate a scalar "magnitude" for ordering
            PF_FpLong sumA = aP->value1 + aP->value2;
            PF_FpLong sumB = bP->value1 + bP->value2;
            *resultP = (sumA < sumB) ? PF_ArbCompare_LESS : PF_ArbCompare_MORE;
        }
    }

    suites.HandleSuite1()->host_unlock_handle(bH);
    suites.HandleSuite1()->host_unlock_handle(aH);

    return err;
}
```

#### PF_ArbCompareResult Values

| Value | Meaning |
|-------|---------|
| `PF_ArbCompare_EQUAL` | The two values are identical |
| `PF_ArbCompare_LESS` | Value A is "less than" value B |
| `PF_ArbCompare_MORE` | Value A is "greater than" value B |
| `PF_ArbCompare_NOT_EQUAL` | Values differ (no ordering defined) |

> **Pitfall**: If you only ever return `EQUAL` or `NOT_EQUAL`, interpolation may behave unexpectedly. For properly ordered interpolation, implement meaningful `LESS`/`MORE` comparisons.

### PF_Arbitrary_PRINT_SIZE_FUNC / PF_Arbitrary_PRINT_FUNC

These convert arb data to a human-readable text representation. Used when the user copies keyframe data as text.

```c
PF_Err ArbPrintSize(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    // Report the maximum number of bytes needed for the printed string
    *(arbExtra->u.print_size_func_params.print_sizePLu) = 256;
    return PF_Err_NONE;
}

PF_Err ArbPrint(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    PF_ArbitraryH arbH = arbExtra->u.print_func_params.arbH;
    A_char *bufP        = arbExtra->u.print_func_params.print_bufferPC;
    A_u_long buf_size   = arbExtra->u.print_func_params.print_sizeLu;

    MyArbData *dataP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(arbH);
    if (dataP) {
        suites.ANSICallbacksSuite1()->sprintf(bufP, "%.4f,%.4f",
            dataP->value1, dataP->value2);
        suites.HandleSuite1()->host_unlock_handle(arbH);
    }
    return err;
}
```

The `print_flags` field can contain `PF_ArbPrint_ABBREVIATED` (1 << 0) requesting a shorter representation, or `PF_ArbPrint_NONE` (0) for the complete description.

### PF_Arbitrary_SCAN_FUNC

Parse text back into arb data. The inverse of `PRINT_FUNC`. Called when the user pastes keyframe text.

```c
PF_Err ArbScan(
    PF_InData           *in_data,
    PF_OutData          *out_data,
    PF_ArbParamsExtra   *arbExtra)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    const A_char *bufP  = arbExtra->u.scan_func_params.bufPC;
    PF_ArbitraryH *arbPH = arbExtra->u.scan_func_params.arbPH;

    *arbPH = suites.HandleSuite1()->host_new_handle(sizeof(MyArbData));
    if (*arbPH) {
        MyArbData *dataP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(*arbPH);
        if (dataP) {
            AEFX_CLR_STRUCT(*dataP);
            if (sscanf(bufP, "%lf,%lf", &dataP->value1, &dataP->value2) != 2) {
                err = PF_Err_CANNOT_PARSE_KEYFRAME_TEXT;
            }
            suites.HandleSuite1()->host_unlock_handle(*arbPH);
        }
    }
    return err;
}
```

Return `PF_Err_CANNOT_PARSE_KEYFRAME_TEXT` if the text cannot be parsed.

## Combining Arb Data with Custom UI

The power of arb data comes from pairing it with a custom control that visualizes and edits the data. Here is the full pattern:

### Step 1: Set Up the Parameter

```c
// In ParamsSetup
def.param_type    = PF_Param_ARBITRARY_DATA;
def.ui_flags      = PF_PUI_CONTROL;
def.ui_width      = 200;
def.ui_height     = 100;
def.flags         = PF_ParamFlag_SUPERVISE;
def.u.arb_d.id    = 1;
def.u.arb_d.dephault = default_arbH;
def.u.arb_d.value = NULL;
```

### Step 2: Register Custom UI

```c
PF_CustomUIInfo ci;
AEFX_CLR_STRUCT(ci);
ci.events = PF_CustomEFlag_EFFECT;  // ECW events
err = in_data->inter.register_ui(in_data->effect_ref, &ci);
```

### Step 3: Draw the Custom Control

During `PF_Event_DRAW`, check that the event is for your arb parameter's control area:

```c
PF_Err DrawEvent(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_LayerDef     *output,
    PF_EventExtra   *event_extra)
{
    PF_Err err = PF_Err_NONE;

    // Only draw in the ECW control area for our arb parameter
    if ((*event_extra->contextH)->w_type != PF_Window_EFFECT) {
        return err;
    }
    if (event_extra->effect_win.area != PF_EA_CONTROL) {
        return err;
    }
    if (event_extra->effect_win.index != MY_ARB_PARAM_INDEX) {
        return err;
    }

    // Lock the arb data
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    MyArbData *dataP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(
        params[MY_ARB_PARAM_INDEX]->u.arb_d.value);

    if (dataP) {
        // Get drawing reference
        DRAWBOT_DrawRef drawing_ref = NULL;
        ERR(suites.EffectCustomUISuite1()->PF_GetDrawingReference(
            event_extra->contextH, &drawing_ref));

        if (drawing_ref) {
            // Draw your visualization using Drawbot
            DrawMyControl(in_data, out_data, dataP, event_extra, drawing_ref);
        }

        suites.HandleSuite1()->host_unlock_handle(
            params[MY_ARB_PARAM_INDEX]->u.arb_d.value);
    }

    event_extra->evt_out_flags = PF_EO_HANDLED_EVENT;
    return err;
}
```

### Step 4: Handle Clicks to Modify Arb Data

When the user clicks on your custom control, modify the arb data and signal the change:

```c
PF_Err DoClick(
    PF_InData       *in_data,
    PF_OutData      *out_data,
    PF_ParamDef     *params[],
    PF_LayerDef     *output,
    PF_EventExtra   *event_extra)
{
    PF_Err err = PF_Err_NONE;

    if ((*event_extra->contextH)->w_type != PF_Window_EFFECT) {
        return err;
    }
    if (event_extra->effect_win.area != PF_EA_CONTROL) {
        return err;
    }

    PF_Point mouse = event_extra->u.do_click.screen_point;
    PF_UnionableRect frame = event_extra->effect_win.current_frame;

    // Calculate relative position within the control
    float rel_x = (float)(mouse.h - frame.left) / (float)(frame.right - frame.left);
    float rel_y = (float)(mouse.v - frame.top) / (float)(frame.bottom - frame.top);

    // Lock and modify arb data
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    MyArbData *dataP = (MyArbData*)suites.HandleSuite1()->host_lock_handle(
        params[MY_ARB_PARAM_INDEX]->u.arb_d.value);

    if (dataP) {
        dataP->value1 = rel_x;
        dataP->value2 = rel_y;
        suites.HandleSuite1()->host_unlock_handle(
            params[MY_ARB_PARAM_INDEX]->u.arb_d.value);
    }

    // Signal that the parameter value changed
    params[MY_ARB_PARAM_INDEX]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
    event_extra->evt_out_flags = PF_EO_HANDLED_EVENT;

    // Start drag if desired
    event_extra->u.do_click.send_drag = TRUE;

    return err;
}
```

## Updating Custom UI Dimensions at Runtime

You can change the height of your custom control area using `PF_UpdateParamUI`:

```c
// During PF_Cmd_UPDATE_PARAMS_UI or PF_Cmd_USER_CHANGED_PARAM:
PF_ParamDef updated_def;
AEFX_CLR_STRUCT(updated_def);

// Only these fields can be changed:
updated_def.ui_width  = new_width;
updated_def.ui_height = new_height;

suites.ParamUtilsSuite3()->PF_UpdateParamUI(
    in_data->effect_ref,
    MY_ARB_PARAM_INDEX,
    &updated_def);
```

> **Pitfall**: `PF_UpdateParamUI` can only change cosmetic properties (`ui_flags`, `ui_width`, `ui_height`, `name`, and certain `flags`). You cannot change parameter values through this function -- use `PF_ChangeFlag_CHANGED_VALUE` instead.

## PF_Param_NO_DATA: Custom UI Without Data

If you want a custom control that has no associated data stream (no keyframes, no saved state), use `PF_Param_NO_DATA` instead of `PF_Param_ARBITRARY_DATA`:

```c
def.param_type = PF_Param_NO_DATA;
def.ui_flags   = PF_PUI_CONTROL;
def.ui_width   = 200;
def.ui_height  = 50;
```

This gives you a drawable area in the ECW but no arb callback system. Useful for static displays, buttons, or controls that modify other parameters rather than holding their own data.

## Using PF_PUI_STD_CONTROL_ONLY with Arb Data

If your arb data has a complex internal structure but the user only needs to interact with it through standard parameters (sliders, colors), you can add standard controls that modify the arb data indirectly:

```c
// Add a slider that modifies the arb data through USER_CHANGED_PARAM
PF_ParamDef slider_def;
AEFX_CLR_STRUCT(slider_def);
slider_def.ui_flags = PF_PUI_STD_CONTROL_ONLY;
// ... set up as normal float slider ...
PF_ADD_FLOAT_SLIDERX(...);
```

Then in `PF_Cmd_USER_CHANGED_PARAM`:

```c
if (extra->param_index == MY_SLIDER_INDEX) {
    // Read the slider value and write it into the arb data
    MyArbData *dataP = LockMyArb(in_data, params);
    dataP->some_field = params[MY_SLIDER_INDEX]->u.fs_d.value;
    UnlockMyArb(in_data, params);
    params[MY_ARB_PARAM_INDEX]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
}
```

## Versioning Arb Data

When you update your plugin and change the arb data structure, old projects will contain the old format. Handle this in `UNFLATTEN_FUNC`:

```c
typedef struct {
    A_u_long version;  // Always first field
    // ... rest of data
} MyArbData_v2;

PF_Err ArbUnflatten(...)
{
    const A_u_long *versionP = (const A_u_long*)flatP;

    if (*versionP == 1) {
        // Old format: read MyArbData_v1, convert to v2
        MyArbData_v1 old_data;
        memcpy(&old_data, flatP, sizeof(MyArbData_v1));
        // ... create v2 and migrate ...
    } else if (*versionP == 2) {
        // Current format
        memcpy(dataP, flatP, sizeof(MyArbData_v2));
    }
}
```

## Common Pitfalls

1. **Not implementing all callbacks.** AE will call every callback at various points. An unhandled callback causes undefined behavior. At minimum, implement all eleven: NEW, DISPOSE, COPY, FLAT_SIZE, FLATTEN, UNFLATTEN, INTERP, COMPARE, PRINT_SIZE, PRINT, SCAN.

2. **Allocating dst in COPY_FUNC.** The destination handle already exists. Just lock, copy, unlock. Allocating a new handle leaks the original.

3. **Forgetting to set PF_PUI flags.** In AE, `PF_Param_ARBITRARY_DATA` without `PF_PUI_CONTROL`, `PF_PUI_TOPIC`, or `PF_PUI_NO_ECW_UI` will cause problems.

4. **Changing arb data during render.** Arb data accessed through `params[]` during render is read-only. Modify it only during UI selectors (`PF_Cmd_EVENT`, `PF_Cmd_USER_CHANGED_PARAM`).

5. **Not handling version migration.** When your data structure changes, old projects will unflatten old-format data. Always version your flat format.

6. **Returning PF_Err_NONE from SCAN_FUNC on parse failure.** Return `PF_Err_CANNOT_PARSE_KEYFRAME_TEXT` so AE knows the paste failed. Returning success with garbage data corrupts the parameter.

7. **Thread safety with arb data.** When `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` is set, arb data in params is read-only during render. Lock handles, read, unlock. Do not write.

8. **Large arb data causing undo bloat.** Every `PF_ChangeFlag_CHANGED_VALUE` on an arb parameter stores the entire arb data in the undo buffer. If your data is large, consider using sequence_data for non-undoable state and arb data only for the user-facing values.
