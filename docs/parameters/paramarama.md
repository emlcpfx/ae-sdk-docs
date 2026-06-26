# Parameter UI Showcase: The Paramarama Example

The `Paramarama` SDK example is a living catalog of After Effects parameter types. Located at `Examples/Effect/Paramarama/`, its purpose is not to implement a useful image effect but to exercise every major parameter control type available to effect plugins, along with demonstrating parameter supervision and host compatibility patterns.

The actual render is a sharpening kernel (borrowed from AE's built-in Sharpen), but the value of this example lies entirely in its `ParamsSetup` function and its patterns for handling diverse parameter types.

---

## Parameter Inventory

Paramarama registers the following parameters, in order:

| Index | Enum | Type | Macro | Description |
|---|---|---|---|---|
| 0 | `PARAMARAMA_INPUT` | Layer | (automatic) | Default input layer -- always index 0 |
| 1 | `PARAMARAMA_AMOUNT` | Integer Slider | `PF_ADD_SLIDER` | "An obsolete slider" |
| 2 | `PARAMARAMA_COLOR` | Color Picker | `PF_ADD_COLOR` | "Color to mix" |
| 3 | `PARAMARAMA_FLOAT_VAL` | Float Slider | `PF_ADD_FLOAT_SLIDER` | "Some float value" |
| 4 | `PARAMARAMA_ANGLE` | Angle | `PF_ADD_ANGLE` | "An angle control" |
| 5 | `PARAMARAMA_POPUP` | Popup Menu | `PF_ADD_POPUP` | "Pop-up param" |
| 6 | `PARAMARAMA_DOWNSAMPLE` | Checkbox | `PF_ADD_CHECKBOX` | "Some checkbox" |
| 7 | `PARAMARAMA_3D_POINT` | 3D Point / Arbitrary | `PF_ADD_POINT_3D` | "3D Point" (AE only) |
| 8 | `PARAMARAMA_BUTTON` | Button | `PF_ADD_BUTTON` | "Button" (supervised) |

> **Important**: The order in the enum does not match the order of `PF_ADD_*` calls. The index order in the enum is `AMOUNT, COLOR, FLOAT_VAL, ANGLE, POPUP, DOWNSAMPLE, 3D_POINT, BUTTON`, but the actual `PF_ADD_*` calls place the checkbox (index 6 `DOWNSAMPLE`) between the popup and the 3D point. Always verify order by checking the actual `PF_ADD_*` call sequence, not the enum order.

---

## Disk IDs

Each parameter has a separate "disk ID" used for serialization. These must remain stable across versions:

```cpp
enum {
    AMOUNT_DISK_ID = 1,
    COLOR_DISK_ID,
    FLOAT_DISK_ID,
    DOWNSAMPLE_DISK_ID,
    ANGLE_DISK_ID,
    NULL_DISK_ID,
    POPUP_DISK_ID,
    THREE_D_POINT_DISK_ID,
    BUTTON_DISK_ID
};
```

> **Pitfall**: Note that `NULL_DISK_ID` (6) exists in the disk ID enum even though there is no null parameter currently added. This is a historical artifact. Disk IDs must never be reused or reordered, even if parameters are removed. Doing so corrupts saved projects.

---

## Detailed Parameter Breakdown

### Integer Slider (PF_ADD_SLIDER)

```cpp
PF_ADD_SLIDER(STR(StrID_Slider_Param_Name),    // Name
              PARAMARAMA_AMOUNT_MIN,             // Valid min (0)
              PARAMARAMA_AMOUNT_MAX,             // Valid max (100)
              PARAMARAMA_AMOUNT_MIN,             // Slider min (0)
              PARAMARAMA_AMOUNT_MAX,             // Slider max (100)
              PARAMARAMA_AMOUNT_DFLT,            // Default (93)
              AMOUNT_DISK_ID);                   // Disk ID
```

The integer slider provides a whole-number control. The value is accessed as `params[PARAMARAMA_AMOUNT]->u.sd.value` (a `PF_ParamDefUnion` `sd` field for standard def / slider def).

The slider has two ranges:
- **Valid range**: The absolute min/max the user can type
- **Slider range**: The visual range of the slider widget (can be narrower)

### Color Picker (PF_ADD_COLOR)

```cpp
PF_ADD_COLOR(STR(StrID_Color_Param_Name),    // Name
             DEFAULT_RED,                     // Red default (111)
             DEFAULT_BLUE,                    // Blue default (33)
             DEFAULT_GREEN,                   // Green default (222)
             COLOR_DISK_ID);                  // Disk ID
```

> **Pitfall**: The parameter order in `PF_ADD_COLOR` is **Red, Blue, Green** -- not Red, Green, Blue. This is a known SDK quirk. The Paramarama header defines `DEFAULT_RED=111, DEFAULT_GREEN=222, DEFAULT_BLUE=33`, but passes them as `(111, 33, 222)` to match the macro's (Red, Blue, Green) order.

The color value is accessed as `params[PARAMARAMA_COLOR]->u.cd.value`, which is a `PF_Pixel` struct.

### Float Slider (PF_ADD_FLOAT_SLIDER)

```cpp
PF_ADD_FLOAT_SLIDER(STR(StrID_Float_Param_Name),       // Name
                    static_cast<PF_FpShort>(FLOAT_MIN), // Valid min (2.387)
                    FLOAT_MAX,                           // Valid max (987.653)
                    FLOAT_MIN,                           // Slider min
                    FLOAT_MAX,                           // Slider max
                    AEFX_AUDIO_DEFAULT_CURVE_TOLERANCE,  // Curve tolerance
                    DEFAULT_FLOAT_VAL,                   // Default (27.62)
                    2,                                   // Precision (decimal places)
                    0,                                   // Display flags
                    1,                                   // Wants phase
                    FLOAT_DISK_ID);                      // Disk ID
```

This is the most parameter-rich control. Key fields:

| Parameter | Value | Meaning |
|---|---|---|
| Precision | `2` | Show 2 decimal places in the UI |
| Display flags | `0` | No special display behavior |
| Wants phase | `1` | Hint that this parameter has phase behavior (for audio) |
| Curve tolerance | `AEFX_AUDIO_DEFAULT_CURVE_TOLERANCE` | Keyframe interpolation tolerance |

The float value is accessed as `params[PARAMARAMA_FLOAT_VAL]->u.fs_d.value`.

### Checkbox (PF_ADD_CHECKBOX)

```cpp
PF_ADD_CHECKBOX(STR(StrID_Checkbox_Param_Name),     // Name
                STR(StrID_Checkbox_Description),     // Comment "(with comment!)"
                FALSE,                               // Default value
                0,                                   // Flags
                DOWNSAMPLE_DISK_ID);                 // Disk ID
```

Checkboxes have two text strings: the parameter name (shown on the left in the ECW) and a description/comment (shown next to the checkbox). Access the value as `params[PARAMARAMA_DOWNSAMPLE]->u.bd.value`.

### Angle Control (PF_ADD_ANGLE)

```cpp
PF_ADD_ANGLE(STR(StrID_Angle_Param_Name),    // Name
             0,                               // Default angle (degrees)
             ANGLE_DISK_ID);                  // Disk ID
```

The angle control shows a dial widget in the Effect Controls. The value is a `PF_Fixed` in degrees, accessed via `params[PARAMARAMA_ANGLE]->u.ad.value`. One full rotation is 360 degrees; the control supports multiple rotations (values beyond 360).

### Popup Menu (PF_ADD_POPUP)

```cpp
PF_ADD_POPUP(STR(StrID_Popup_Param_Name),    // Name
             5,                               // Number of choices
             1,                               // Default choice (1-based)
             STR(StrID_Popup_Choices),        // Choice string
             POPUP_DISK_ID);                  // Disk ID
```

The choice string uses pipe `|` as the separator:

```
"Make Slower|Make Jaggy|(-|Plan A|Plan B"
```

Special syntax in popup strings:

| Syntax | Effect |
|---|---|
| `\|` | Separator between choices |
| `(-` | Inserts a separator line (non-selectable divider) |

The popup value is 1-based: `params[PARAMARAMA_POPUP]->u.pd.value` returns 1 through 5. Note that the separator counts as a choice in the `num_choices` count.

### 3D Point (PF_ADD_POINT_3D)

```cpp
PF_ADD_POINT_3D(STR(StrID_3D_Point_Param_Name),
                DEFAULT_POINT_VALS,              // X default (50.0)
                DEFAULT_POINT_VALS,              // Y default (50.0)
                DEFAULT_POINT_VALS,              // Z default (50.0)
                THREE_D_POINT_DISK_ID);          // Disk ID
```

The 3D point is only available in After Effects (identified by `in_data->appl_id == 'FXTC'`). The example includes host-detection logic:

```cpp
if (in_data->version.major >= PF_AE105_PLUG_IN_VERSION &&
    in_data->version.minor >= PF_AE105_PLUG_IN_SUBVERS) {

    if (in_data->appl_id == 'FXTC') {
        PF_ADD_POINT_3D(...);
    } else {
        // Placeholder for non-AE hosts
        PF_ADD_ARBITRARY2(STR(StrID_3D_Point_Param_Name),
                          1, 1, 0,
                          PF_PUI_NO_ECW_UI,
                          0, THREE_D_POINT_DISK_ID, 0);
    }
}
```

> **Pitfall**: When a parameter type is not supported by the host (e.g., 3D points in Premiere Pro), you must still add a placeholder at the same parameter index to keep the parameter array consistent. Using `PF_ADD_ARBITRARY2` with `PF_PUI_NO_ECW_UI` creates an invisible placeholder.

### Button (PF_ADD_BUTTON)

```cpp
PF_ADD_BUTTON(STR(StrID_Button_Param_Name),     // Name
              STR(StrID_Button_Label_Name),      // Button face text
              0,                                 // Reserved
              PF_ParamFlag_SUPERVISE,            // Flags: SUPERVISE
              BUTTON_DISK_ID);                   // Disk ID
```

Buttons require `PF_ParamFlag_SUPERVISE` to trigger the `PF_Cmd_USER_CHANGED_PARAM` command when clicked. Without this flag, clicking the button does nothing.

---

## Parameter Supervision

The `PF_Cmd_USER_CHANGED_PARAM` command is handled in the `UserChangedParam` function:

```cpp
static PF_Err UserChangedParam(
    PF_InData                       *in_data,
    PF_OutData                      *out_data,
    PF_ParamDef                     *params[],
    const PF_UserChangedParamExtra  *which_hitP)
{
    PF_Err err = PF_Err_NONE;

    if (which_hitP->param_index == PARAMARAMA_BUTTON) {
        if (in_data->appl_id != 'PrMr') {
            PF_STRCPY(out_data->return_msg,
                      STR(StrID_Button_Message));
            out_data->out_flags |= PF_OutFlag_DISPLAY_ERROR_MESSAGE;
        } else {
            // In Premiere Pro, this message appears in the Events panel
            PF_STRCPY(out_data->return_msg,
                      STR(StrID_Button_Message));
        }
    }

    return err;
}
```

### How Supervision Works

1. A parameter is marked with `PF_ParamFlag_SUPERVISE` during `PF_ADD_*`.
2. When the user changes that parameter, AE sends `PF_Cmd_USER_CHANGED_PARAM`.
3. The `extra` pointer is cast to `PF_UserChangedParamExtra`, which contains `param_index` identifying which parameter was changed.
4. The handler can then modify other parameters, display messages, or trigger UI updates.

### Displaying Messages

The example uses two different message display mechanisms:

| Host | Mechanism |
|---|---|
| After Effects (`'FXTC'`) | `out_data->return_msg` + `PF_OutFlag_DISPLAY_ERROR_MESSAGE` shows a modal dialog |
| Premiere Pro (`'PrMr'`) | `out_data->return_msg` alone posts to the Events panel |

---

## Dynamic Parameter Count Based on Host Version

Paramarama adjusts its parameter count based on host capabilities:

```cpp
if (in_data->version.major >= PF_AE105_PLUG_IN_VERSION &&
    in_data->version.minor >= PF_AE105_PLUG_IN_SUBVERS) {
    // Add 3D point and button
    out_data->num_params = PARAMARAMA_NUM_PARAMS;       // 9
} else {
    out_data->num_params = PARAMARAMA_NUM_PARAMS - 2;   // 7
}
```

This ensures the plugin works in older hosts that do not support 3D points or buttons. The effect appears with fewer controls but remains functional.

> **Pitfall**: `out_data->num_params` must exactly match the number of `PF_ADD_*` calls executed. A mismatch causes undefined behavior ranging from missing parameters to crashes.

---

## The String Table Pattern

Paramarama uses the SDK's recommended string table pattern for all user-visible strings:

```cpp
// Paramarama_Strings.h
typedef enum {
    StrID_NONE,
    StrID_Name,
    StrID_Description,
    StrID_Slider_Param_Name,
    // ... more IDs ...
    StrID_NUMTYPES
} StrIDType;

// Paramarama_Strings.cpp
TableString g_strs[StrID_NUMTYPES] = {
    StrID_NONE,                 "",
    StrID_Name,                 "Paramarama",
    StrID_Description,          "Parameter Party!\rExercising all parameter types.\r...",
    StrID_Slider_Param_Name,    "An obsolete slider",
    // ...
};

char *GetStringPtr(int strNum) {
    return g_strs[strNum].str;
}
```

Usage via `String_Utils.h`:

```cpp
#define STR(_foo) GetStringPtr(_foo)
// Then in code:
PF_ADD_SLIDER(STR(StrID_Slider_Param_Name), ...);
```

This pattern centralizes all strings in one file, making localization straightforward. Each plugin provides its own `GetStringPtr` implementation with its own `g_strs` table.

---

## Render: Convolution-Based Sharpening

While not the focus of this example, the render function demonstrates several useful patterns:

### Zero-Value Fast Path

```cpp
if (0 == params[PARAMARAMA_AMOUNT]->u.sd.value) {
    // Copy source to destination unchanged
    if (PF_Quality_HI == in_data->quality && in_data->appl_id != 'PrMr') {
        ERR(suites.WorldTransformSuite1()->copy_hq(...));
    } else if (in_data->appl_id != 'PrMr') {
        ERR(suites.WorldTransformSuite1()->copy(...));
    } else {
        ERR(PF_COPY(&params[0]->u.ld, output, NULL, NULL));
    }
}
```

Three code paths for the simple copy operation:
1. **AE, high quality**: `WorldTransformSuite1->copy_hq()` (handles sub-pixel positioning)
2. **AE, draft quality**: `WorldTransformSuite1->copy()` (fast copy)
3. **Premiere Pro**: `PF_COPY()` macro (Premiere does not support `WorldTransformSuite1`)

### Convolution Kernel

```cpp
A_long convKer[9] = {0};
PF_FpLong sharpen = PF_CEIL(params[PARAMARAMA_AMOUNT]->u.sd.value / 16);
PF_FpLong kernelSum = 256 * 9;

convKer[4]  = (long)(sharpen * kernelSum);
kernelSum   = (256 * 9 - convKer[4]) / 4;
convKer[1]  = convKer[3] = convKer[5] = convKer[7] = (long)kernelSum;
```

This builds a 3x3 sharpening kernel where the center pixel is boosted and the 4-connected neighbors are reduced, producing the classic unsharp mask effect.

---

## GlobalSetup Flags

```cpp
out_data->out_flags  = PF_OutFlag_DEEP_COLOR_AWARE;
out_data->out_flags2 = PF_OutFlag2_SUPPORTS_THREADED_RENDERING;
```

| Flag | Meaning |
|---|---|
| `PF_OutFlag_DEEP_COLOR_AWARE` | Plugin handles 16-bit color (AE will send 16-bit worlds) |
| `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` | Safe for Multi-Frame Rendering |

The matching PiPL values:
- `AE_Effect_Global_OutFlags { 0x2000000 }` = `PF_OutFlag_DEEP_COLOR_AWARE`
- `AE_Effect_Global_OutFlags_2 { 0x8000000 }` = `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`

> **Note**: Although `PF_OutFlag_DEEP_COLOR_AWARE` is set, the `Render` function does not include separate paths for 16-bit pixels. The convolution callback in `WorldTransformSuite1` handles depth internally. If you were using `PF_ITERATE` with custom pixel functions, you would need separate callbacks for `PF_Pixel8` and `PF_Pixel16`.

---

## Entry Point and Host Registration

```cpp
extern "C" DllExport
PF_Err PluginDataEntryFunction2(
    PF_PluginDataPtr inPtr,
    PF_PluginDataCB2 inPluginDataCallBackPtr,
    SPBasicSuite* inSPBasicSuitePtr,
    const char* inHostName,
    const char* inHostVersion)
{
    PF_Err result = PF_Err_INVALID_CALLBACK;

    result = PF_REGISTER_EFFECT_EXT2(
        inPtr,
        inPluginDataCallBackPtr,
        "Paramarama",            // Display name
        "ADBE Paramarama",       // Match name (must be unique, never change)
        "Sample Plug-ins",       // Category
        AE_RESERVED_INFO,        // Reserved
        "EffectMain",            // Entry point function name
        "https://www.adobe.com"); // Support URL

    return result;
}
```

The `PluginDataEntryFunction2` is the modern entry point (replacing `PluginDataEntryFunction`). The `EffectMain` function receives all commands and dispatches to handlers:

```cpp
PF_Err EffectMain(
    PF_Cmd       cmd,
    PF_InData    *in_data,
    PF_OutData   *out_data,
    PF_ParamDef  *params[],
    PF_LayerDef  *output,
    void         *extra)       // Note: 6th parameter for supervised effects
```

> **Important**: Effects that handle `PF_Cmd_USER_CHANGED_PARAM` must declare their `EffectMain` with the 6th `void *extra` parameter. The `extra` pointer carries the `PF_UserChangedParamExtra` struct for supervision events and `PF_EventExtra` for custom UI events. The 5-parameter version of `EffectMain` cannot receive these.

---

## Exception Handling Pattern

The main entry point wraps all dispatching in a try/catch:

```cpp
try {
    switch (cmd) {
        case PF_Cmd_ABOUT:     err = About(...);     break;
        case PF_Cmd_RENDER:    err = Render(...);     break;
        // ...
    }
} catch(PF_Err &thrown_err) {
    err = thrown_err;
}
```

This catches any `PF_Err` thrown by suite calls (the `AEGP_SuiteHandler` throws on error) and converts them to return values. This pattern prevents C++ exceptions from propagating across the AE/plugin boundary, which would crash AE.

---

## Source Files

| File | Purpose |
|---|---|
| `Paramarama.cpp` | Effect logic: parameter setup, render, supervision |
| `Paramarama.h` | Enums, defines, includes, entry point declaration |
| `Paramarama_Strings.h` | String ID enum |
| `Paramarama_Strings.cpp` | String table implementation |
| `ParamaramaPiPL.r` | PiPL resource |

---

## Quick Reference: Parameter Access Patterns

| Parameter Type | Add Macro | Union Field | Value Type |
|---|---|---|---|
| Integer Slider | `PF_ADD_SLIDER` | `u.sd.value` | `A_long` |
| Fixed Slider | `PF_ADD_FIXED` | `u.fd.value` | `PF_Fixed` (16.16) |
| Float Slider | `PF_ADD_FLOAT_SLIDER` | `u.fs_d.value` | `PF_FpLong` (double) |
| Angle | `PF_ADD_ANGLE` | `u.ad.value` | `PF_Fixed` (degrees) |
| Checkbox | `PF_ADD_CHECKBOX` | `u.bd.value` | `PF_Boolean` |
| Color | `PF_ADD_COLOR` | `u.cd.value` | `PF_Pixel` |
| Popup | `PF_ADD_POPUP` | `u.pd.value` | `A_long` (1-based) |
| Point | `PF_ADD_POINT` | `u.td.x_value`, `u.td.y_value` | `PF_Fixed` |
| 3D Point | `PF_ADD_POINT_3D` | `u.point3d_d.x_value`, etc. | `PF_FpLong` |
| Button | `PF_ADD_BUTTON` | N/A | No value; triggers supervision |
| Arbitrary | `PF_ADD_ARBITRARY2` | `u.arb_d.value` | `PF_ArbitraryH` |

---

## See Also

- `PF_ParamDef` structure in `AE_Effect.h`
- `Param_Utils.h` for the `PF_ADD_*` macro definitions
- The **Checkout** example for layer parameter (`PF_ADD_LAYER`) usage
- The **CCU** (Custom Comp UI) example for custom parameter UI
