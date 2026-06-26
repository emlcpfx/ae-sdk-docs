# After Effects Plugin Parameter Greying Out (Disabling) Guide

This guide explains how to dynamically disable (grey out) parameters in After Effects plugins based on the state of other parameters.

## Table of Contents
1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Implementation Steps](#implementation-steps)
4. [Complete Code Example](#complete-code-example)
5. [Common Patterns](#common-patterns)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)
8. [SDK References](#sdk-references)

## Overview

Parameter disabling/greying out allows plugins to create dynamic UIs where certain controls become unavailable based on the state of other parameters. This improves user experience by clearly indicating which parameters are relevant in the current configuration.

### Key Use Cases
- Disabling advanced settings when in "simple" mode
- Greying out dependent parameters when their parent feature is disabled
- Hiding irrelevant controls based on workflow choices
- Preventing conflicting parameter combinations

## Core Concepts

### Three Essential Components

1. **PF_OutFlag_SEND_UPDATE_PARAMS_UI** - Global flag to receive UI update messages
2. **PF_Cmd_UPDATE_PARAMS_UI** - Command sent when UI needs updating
3. **PF_PUI_DISABLED** - UI flag that greys out the parameter

### SDK Functions and Flags

#### Global Setup Flag
```cpp
PF_OutFlag_SEND_UPDATE_PARAMS_UI = 1L << 25  // 0x02000000
```
This flag must be set in `GlobalSetup` to receive `PF_Cmd_UPDATE_PARAMS_UI` messages.

#### UI Flags for Parameters
```cpp
PF_PUI_DISABLED  = 1L << 2  // Makes parameter appear greyed out
PF_PUI_INVISIBLE = 1L << 4  // Hides parameter completely (limited support)
```

#### Key Suite Function
```cpp
PF_UpdateParamUI(effect_ref, param_index, param_def_ptr)
```
Part of `PF_ParamUtilSuite3`, this function updates the UI state of a parameter.

## Implementation Steps

### Step 1: Enable UI Update Messages

In your `GlobalSetup` function, add the `PF_OutFlag_SEND_UPDATE_PARAMS_UI` flag:

```cpp
static PF_Err GlobalSetup(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_ParamDef* params[],
    PF_LayerDef* output)
{
    out_data->my_version = PF_VERSION(1, 0, 0, 0, 0);

    // Add this flag to receive UPDATE_PARAMS_UI messages
    out_data->out_flags = PF_OutFlag_DEEP_COLOR_AWARE |
                         PF_OutFlag_PIX_INDEPENDENT |
                         PF_OutFlag_SEND_UPDATE_PARAMS_UI;  // <-- Critical!

    out_data->out_flags2 = PF_OutFlag2_SUPPORTS_SMART_RENDER |
                          PF_OutFlag2_FLOAT_COLOR_AWARE |
                          PF_OutFlag2_SUPPORTS_THREADED_RENDERING;

    return PF_Err_NONE;
}
```

### Step 2: Handle UPDATE_PARAMS_UI Command

Add a handler for the `PF_Cmd_UPDATE_PARAMS_UI` command in your main dispatcher:

```cpp
DllExport PF_Err EffectMain(
    PF_Cmd cmd,
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_ParamDef* params[],
    PF_LayerDef* output,
    void* extra)
{
    PF_Err err = PF_Err_NONE;

    switch(cmd) {
        // ... other cases ...

        case PF_Cmd_UPDATE_PARAMS_UI:
            err = UpdateParamsUI(in_data, out_data, params, output);
            break;

        // ... other cases ...
    }

    return err;
}
```

### Step 3: Implement UpdateParamsUI Function

Create the function that checks parameter states and updates UI accordingly:

> **Note:** Use `in_data->current_time` for the parameter checkout time in most cases. Some older references suggest using time 0 to avoid "no key values" errors, but `current_time` is the standard approach and ensures you get the parameter value at the current timeline position.

```cpp
static PF_Err UpdateParamsUI(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_ParamDef* params[],
    PF_LayerDef* output)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    // Check the controlling parameter's state
    PF_ParamDef control_param;
    AEFX_CLR_STRUCT(control_param);

    ERR(PF_CHECKOUT_PARAM(in_data,
                         PARAM_CONTROL,  // The parameter that controls others
                         in_data->current_time,
                         in_data->time_step,
                         in_data->time_scale,
                         &control_param));

    if (!err) {
        // Checkout the dependent parameter first
        PF_ParamDef dependent_param;
        AEFX_CLR_STRUCT(dependent_param);

        ERR(PF_CHECKOUT_PARAM(in_data,
                             PARAM_DEPENDENT,
                             in_data->current_time,
                             in_data->time_step,
                             in_data->time_scale,
                             &dependent_param));

        // Update UI flags based on control parameter state
        if (control_param.u.bd.value) {  // If checkbox is checked
            dependent_param.ui_flags |= PF_PUI_DISABLED;  // Disable
        } else {
            dependent_param.ui_flags &= ~PF_PUI_DISABLED; // Enable
        }

        // Mark as changed
        dependent_param.uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;

        // Apply the UI update - CRITICAL: Must call this!
        ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(
            in_data->effect_ref,
            PARAM_DEPENDENT,
            &dependent_param));

        // Check in both parameters
        ERR(PF_CHECKIN_PARAM(in_data, &dependent_param));
        ERR(PF_CHECKIN_PARAM(in_data, &control_param));
    }

    // Request UI refresh
    out_data->out_flags |= PF_OutFlag_REFRESH_UI;

    return err;
}
```

## Complete Code Example

Here's an implementation that greys out a "Blur" parameter when "Keep Original Alpha" is enabled:

```cpp
// Parameter indices
enum {
    PARAM_INPUT = 0,
    PARAM_EROSION,
    PARAM_EXTEND,
    PARAM_BLUR,
    PARAM_KEEP_ORIGINAL_ALPHA,
    PARAM_TOTAL
};

// UPDATE_PARAMS_UI handler
static PF_Err UpdateParamsUI(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_ParamDef* params[],
    PF_LayerDef* output)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    // Check the Keep Original Alpha checkbox state
    PF_ParamDef keep_alpha_param;
    AEFX_CLR_STRUCT(keep_alpha_param);

    ERR(PF_CHECKOUT_PARAM(in_data,
                         PARAM_KEEP_ORIGINAL_ALPHA,
                         in_data->current_time,
                         in_data->time_step,
                         in_data->time_scale,
                         &keep_alpha_param));

    if (!err) {
        // Create a copy of the Blur parameter for updating
        PF_ParamDef blur_param = *params[PARAM_BLUR];

        // If Keep Original Alpha is checked, disable blur
        if (keep_alpha_param.u.bd.value) {
            blur_param.ui_flags |= PF_PUI_DISABLED;
        } else {
            blur_param.ui_flags &= ~PF_PUI_DISABLED;
        }

        // Update the blur parameter UI
        ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(
            in_data->effect_ref,
            PARAM_BLUR,
            &blur_param));

        // Check in the parameter we checked out
        ERR(PF_CHECKIN_PARAM(in_data, &keep_alpha_param));
    }

    return err;
}
```

## Common Patterns

### Pattern 1: Mode-Based Parameter Groups

Disable groups of parameters based on a mode selection:

```cpp
static PF_Err UpdateParamsUI(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_ParamDef* params[],
    PF_LayerDef* output)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    // Check mode parameter
    PF_ParamDef mode_param;
    AEFX_CLR_STRUCT(mode_param);
    ERR(PF_CHECKOUT_PARAM(in_data, PARAM_MODE,
                         in_data->current_time,
                         in_data->time_step,
                         in_data->time_scale,
                         &mode_param));

    if (!err) {
        bool is_advanced = (mode_param.u.pd.value == MODE_ADVANCED);

        // Update multiple parameters based on mode
        int advanced_params[] = {
            PARAM_ADVANCED_1,
            PARAM_ADVANCED_2,
            PARAM_ADVANCED_3
        };

        for (int i = 0; i < 3; i++) {
            PF_ParamDef param_copy = *params[advanced_params[i]];

            if (is_advanced) {
                param_copy.ui_flags &= ~PF_PUI_DISABLED;  // Enable
            } else {
                param_copy.ui_flags |= PF_PUI_DISABLED;   // Disable
            }

            ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(
                in_data->effect_ref,
                advanced_params[i],
                &param_copy));
        }

        ERR(PF_CHECKIN_PARAM(in_data, &mode_param));
    }

    return err;
}
```

### Pattern 2: Cascading Dependencies

Disable parameters that depend on other disabled parameters:

```cpp
// If Feature A is off, disable its settings
// If Feature A Setting 1 is off, disable subsettings
if (!feature_a_enabled) {
    setting_1.ui_flags |= PF_PUI_DISABLED;
    subsetting_1_1.ui_flags |= PF_PUI_DISABLED;
    subsetting_1_2.ui_flags |= PF_PUI_DISABLED;
} else {
    setting_1.ui_flags &= ~PF_PUI_DISABLED;

    // Check secondary dependency
    if (!setting_1_enabled) {
        subsetting_1_1.ui_flags |= PF_PUI_DISABLED;
        subsetting_1_2.ui_flags |= PF_PUI_DISABLED;
    } else {
        subsetting_1_1.ui_flags &= ~PF_PUI_DISABLED;
        subsetting_1_2.ui_flags &= ~PF_PUI_DISABLED;
    }
}
```

### Pattern 3: Hiding vs Disabling

Use `PF_PUI_INVISIBLE` to completely hide parameters (note: limited support):

```cpp
// Hide parameter completely (may not work in all hosts)
if (should_hide) {
    param_copy.ui_flags |= PF_PUI_INVISIBLE;
} else {
    param_copy.ui_flags &= ~PF_PUI_INVISIBLE;
}
```

## Best Practices

### 1. Always Use Parameter Copies
Never modify the original parameter definitions passed to your function:
```cpp
// WRONG - Don't modify original
params[PARAM_BLUR]->ui_flags |= PF_PUI_DISABLED;

// CORRECT - Make a copy first
PF_ParamDef blur_param = *params[PARAM_BLUR];
blur_param.ui_flags |= PF_PUI_DISABLED;
```

### 2. Check Out Parameters Properly
Always use proper checkout/checkin for reading parameter values:
```cpp
PF_ParamDef param;
AEFX_CLR_STRUCT(param);
ERR(PF_CHECKOUT_PARAM(in_data, PARAM_INDEX,
                     in_data->current_time,
                     in_data->time_step,
                     in_data->time_scale,
                     &param));
// Use param...
ERR(PF_CHECKIN_PARAM(in_data, &param));
```

### 3. Handle Errors Gracefully
Always check for errors and handle them appropriately:
```cpp
if (!err) {
    // Perform operations
}
```

### 4. Initialize Suite Handler
Always initialize the suite handler when using suite functions:
```cpp
AEGP_SuiteHandler suites(in_data->pica_basicP);
```

### 5. Consider Performance
UPDATE_PARAMS_UI is called frequently, so keep it efficient:
- Cache calculations when possible
- Avoid unnecessary parameter checkouts
- Batch UI updates when updating multiple parameters

### 6. Thread Safety Considerations
Note that UPDATE_PARAMS_UI may affect MFR (Multi-Frame Rendering) compatibility:
```cpp
/*
 This plugin is not marked as Thread-Safe if it writes to
 sequence data during PF_Cmd_UPDATE_PARAMS_UI.
 */
```

## Critical Implementation Details

### MUST Call PF_UpdateParamUI
The most common mistake is forgetting to call `PF_UpdateParamUI` after modifying the ui_flags. Simply changing the flags is NOT enough:

```cpp
// WRONG - This won't work!
PF_ParamDef param;
PF_CHECKOUT_PARAM(...);
param.ui_flags |= PF_PUI_DISABLED;
PF_CHECKIN_PARAM(...);  // UI won't update!

// CORRECT - Must call PF_UpdateParamUI
PF_ParamDef param;
PF_CHECKOUT_PARAM(...);
param.ui_flags |= PF_PUI_DISABLED;
param.uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
suites.ParamUtilsSuite3()->PF_UpdateParamUI(
    in_data->effect_ref, param_index, &param);
PF_CHECKIN_PARAM(...);  // Now UI will update!
```

### Must Checkout Parameters to Modify
You cannot modify parameters directly from the params array. Always checkout first:

```cpp
// WRONG - Don't modify params array directly
params[PARAM_BLUR]->ui_flags |= PF_PUI_DISABLED;

// CORRECT - Checkout, modify, update, checkin
PF_ParamDef param;
AEFX_CLR_STRUCT(param);
PF_CHECKOUT_PARAM(in_data, PARAM_BLUR, in_data->current_time, ...);
param.ui_flags |= PF_PUI_DISABLED;
param.uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
suites.ParamUtilsSuite3()->PF_UpdateParamUI(...);
PF_CHECKIN_PARAM(in_data, &param);
```

### Request UI Refresh
Always request a UI refresh at the end of UPDATE_PARAMS_UI:

```cpp
out_data->out_flags |= PF_OutFlag_REFRESH_UI;
```

## Troubleshooting

### Common Issues and Solutions

#### Issue: Parameters Not Greying Out
**Cause**: Missing `PF_OutFlag_SEND_UPDATE_PARAMS_UI` flag
**Solution**: Add the flag to GlobalSetup:
```cpp
out_data->out_flags |= PF_OutFlag_SEND_UPDATE_PARAMS_UI;
```

#### Issue: UPDATE_PARAMS_UI Not Being Called
**Cause**: Flag not set or command not handled
**Solution**:
1. Verify flag is set in GlobalSetup
2. Add case for `PF_Cmd_UPDATE_PARAMS_UI` in main dispatcher
3. Ensure PiPL resource matches code flags

#### Issue: UI Updates Not Visible
**Cause**: Not calling PF_UpdateParamUI or using wrong effect_ref
**Solution**: Ensure you're calling:
```cpp
suites.ParamUtilsSuite3()->PF_UpdateParamUI(
    in_data->effect_ref,  // Must use in_data->effect_ref
    param_index,
    &param_copy);
```

#### Issue: Crashes When Updating UI
**Cause**: Modifying original parameter or memory issues
**Solution**: Always work with parameter copies, not originals

#### Issue: Parameters Re-enable Unexpectedly
**Cause**: Not handling all UI update scenarios
**Solution**: Ensure UPDATE_PARAMS_UI handler covers all cases:
- Initial load
- Time changes
- Parameter changes
- Effect Controls Window opening

### Debugging Tips

1. **Add Logging**: Use DebugLog or OutputDebugString to trace execution:
```cpp
DebugLog("UpdateParamsUI called - checkbox state: %d",
         keep_alpha_param.u.bd.value);
```

2. **Check Return Values**: Always check error codes:
```cpp
PF_Err err = suites.ParamUtilsSuite3()->PF_UpdateParamUI(...);
if (err) {
    DebugLog("PF_UpdateParamUI failed with error: %d", err);
}
```

3. **Verify Suite Availability**: Ensure suites are properly initialized:
```cpp
if (!suites.ParamUtilsSuite3()) {
    return PF_Err_INTERNAL_STRUCT_DAMAGED;
}
```

## SDK References

### Official Documentation
- **Parameter Supervision**: [ae-plugins.docsforadobe.dev/effect-details/parameter-supervision.html](https://ae-plugins.docsforadobe.dev/effect-details/parameter-supervision.html)
- **PF_ParamDef Structure**: [ae-plugins.docsforadobe.dev/effect-basics/PF_ParamDef.html](https://ae-plugins.docsforadobe.dev/effect-basics/PF_ParamDef.html)
- **Parameters Guide**: [ae-plugins.docsforadobe.dev/effect-basics/parameters.html](https://ae-plugins.docsforadobe.dev/effect-basics/parameters.html)

### SDK Examples
- **Supervisor Sample**: `AfterEffectsSDK/Examples/UI/Supervisor/`
  - Demonstrates comprehensive parameter UI management
  - Shows mode-based parameter hiding/showing
  - Includes advanced UI update patterns

### Key Header Files
- `AE_Effect.h` - Contains flag definitions and command enums
- `AE_EffectCB.h` - Parameter checkout/checkin macros
- `AE_EffectSuites.h` - Suite function definitions
- `AEFX_SuiteHelper.h` - Suite handler utilities

### Forum Discussions
- [Adobe Community: How to grey out part of plugin UI](https://community.adobe.com/t5/after-effects-discussions/ae-sdk-how-to-grey-out-part-of-plugin-ui/)
- [Adobe Community: How can I hide an effect parameter?](https://community.adobe.com/t5/after-effects/how-can-i-hide-an-effect-parameter/)

## Summary

Greying out parameters in After Effects plugins requires three key components:
1. Setting `PF_OutFlag_SEND_UPDATE_PARAMS_UI` in GlobalSetup
2. Handling `PF_Cmd_UPDATE_PARAMS_UI` in your main dispatcher
3. Using `PF_UpdateParamUI()` with `PF_PUI_DISABLED` flag

The pattern is well-established in the SDK and widely used in professional plugins. Following the examples and best practices in this guide will ensure reliable parameter UI management in your After Effects plugins.

Remember to always test your implementation across different scenarios:
- Loading saved projects
- Changing time positions
- Opening/closing Effect Controls
- Undoing/redoing parameter changes
- Copying effects between layers

This ensures a professional, polished user experience that follows After Effects UI conventions.
