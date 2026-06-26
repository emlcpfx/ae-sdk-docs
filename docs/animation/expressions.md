# Expression Integration for Plugin Developers

This document covers how expressions interact with effect plugin parameters, the evaluation order, API access to expressions via AEGP, and the practical implications for plugin development.

---

## Table of Contents

- [Overview](#overview)
- [How Expressions Affect Plugin Parameters](#how-expressions-affect-plugin-parameters)
- [Evaluation Order](#evaluation-order)
- [PF_ParamFlag_CANNOT_TIME_VARY](#pf_paramflag_cannot_time_vary)
- [When Expressions Override Plugin Values](#when-expressions-override-plugin-values)
- [Reading Pre-Expression vs Post-Expression Values](#reading-pre-expression-vs-post-expression-values)
- [AEGP Expression API](#aegp-expression-api)
- [Expression Errors and Their Effect on Plugins](#expression-errors-and-their-effect-on-plugins)
- [PF_Cmd_USER_CHANGED_PARAM and Expressions](#pf_cmd_user_changed_param-and-expressions)
- [Dependent Parameters and Expressions](#dependent-parameters-and-expressions)
- [Practical Patterns](#practical-patterns)
- [Pitfalls and Warnings](#pitfalls-and-warnings)

---

## Overview

After Effects allows users to write expressions (JavaScript-based scripts) on any time-varying parameter. From a plugin developer's perspective, expressions are an invisible layer between your parameter definitions and the values your render code receives. Understanding this interaction is essential for building robust plugins.

Key principle: **Your plugin's `params[]` array during render always contains the final, post-expression values.** If an expression is applied to one of your parameters, the value in `params[]` is the expression's output, not the raw keyframe value.

---

## How Expressions Affect Plugin Parameters

When a user applies an expression to one of your plugin's parameters:

1. AE evaluates the keyframe-interpolated value of the parameter at the current time
2. That value is fed to the expression engine as `value` (the implicit input)
3. The expression runs and produces a new output value
4. **Your plugin receives the expression's output in `params[]`**

This is transparent to your plugin. You do not need to do anything special to support expressions on your parameters -- they work automatically for all standard parameter types.

```
Keyframes --> Interpolation --> Expression Engine --> params[] --> Your Plugin Render
```

---

## Evaluation Order

The evaluation sequence for a single frame is:

1. **Keyframe interpolation**: AE interpolates keyframe values at the requested time
2. **Expression evaluation**: If an expression exists on the parameter, it runs with the interpolated value available as `value`
3. **Parameter delivery**: The final value is placed in `params[]` for your render call
4. **Effect render**: Your `PF_Cmd_RENDER` or `PF_Cmd_SMART_RENDER` is called

For multiple effects on the same layer, effects are evaluated top to bottom. An expression on Effect B's parameter can reference Effect A's output (since A renders first), but it cannot reference Effect B's own output (circular dependency).

### Important Timing Detail

Expressions can request values at times other than the current frame. For example, `value.wiggle()` or `thisComp.layer("Control").effect("Slider")("Slider").valueAtTime(time - 1)`. When this happens, AE may need to evaluate other properties or even render other frames. This is why `PF_CHECKOUT_PARAM` can sometimes trigger cascading evaluations.

---

## PF_ParamFlag_CANNOT_TIME_VARY

```cpp
PF_ParamFlag_CANNOT_TIME_VARY = 1 << 1   // defined in AE_Effect.h
```

This flag, set during `PF_Cmd_PARAMS_SETUP`, prevents a parameter from being keyframed **and** prevents expressions from being applied to it.

### What It Does

- The stopwatch icon is not displayed in the timeline
- Users cannot add keyframes to the parameter
- Users cannot apply expressions to the parameter
- The parameter has a single static value for the entire duration of the effect

### When to Use It

- Configuration parameters that should not change over time (e.g., algorithm selection, quality mode)
- Parameters where time-varying behavior would be meaningless or dangerous
- Parameters that control plugin structure (e.g., number of iterations that affects memory allocation)

### Example

```cpp
// In PF_Cmd_PARAMS_SETUP
PF_ParamDef def;
AEFX_CLR_STRUCT(def);

def.flags = PF_ParamFlag_CANNOT_TIME_VARY;
PF_ADD_POPUP(
    "Mode",
    3,              // num choices
    1,              // default
    "Fast|Medium|Best",
    MODE_DISK_ID);
```

### Related Flag: PF_ParamFlag_CANNOT_INTERP

```cpp
PF_ParamFlag_CANNOT_INTERP = 1 << 2
```

This flag allows keyframes but prevents interpolation between them. Values jump discretely from one keyframe to the next (equivalent to forcing hold interpolation). Expressions can still be applied when this flag is set.

---

## When Expressions Override Plugin Values

Expressions can completely replace your parameter values. Here are the scenarios:

### Full Override

```javascript
// Expression on a slider parameter
100 * Math.sin(time * Math.PI)
```

This ignores the keyframed/static value entirely. Your plugin receives the sine wave output.

### Partial Override

```javascript
// Expression references the existing value
value + wiggle(5, 10)
```

This adds noise to whatever the keyframed value is. The `value` keyword in expressions refers to the pre-expression (keyframe-interpolated) value.

### Cross-Property References

```javascript
// Expression on Effect B's slider references Effect A's slider
thisComp.layer("Layer 1").effect("My Effect A")("Amount")
```

This creates a dependency between two effect parameters. After Effects handles the dependency graph automatically.

### Impact on Your Plugin

Your render code does not need to handle these cases differently. The value in `params[]` is always the final post-expression value. However, there are implications for:

- **`PF_Cmd_USER_CHANGED_PARAM`**: This callback fires only for user-initiated changes in the UI. It does NOT fire when an expression changes a parameter value during timeline scrubbing.
- **`PF_Cmd_UPDATE_PARAMS_UI`**: Similarly, this is not called for every expression evaluation during playback.

---

## Reading Pre-Expression vs Post-Expression Values

From AEGP code, you can explicitly choose whether to read the pre-expression or post-expression value using `AEGP_StreamSuite6`:

```cpp
AEGP_StreamValue2 value;
A_Time t = {0, 1};

// Post-expression value (what the user sees, what your render gets)
ERR(stream_suite->AEGP_GetNewStreamValue(
    plugin_id,
    streamH,
    AEGP_LTimeMode_CompTime,
    &t,
    FALSE,          // pre_expressionB = FALSE --> evaluate expression
    &value));

// Pre-expression value (raw keyframe interpolation)
ERR(stream_suite->AEGP_GetNewStreamValue(
    plugin_id,
    streamH,
    AEGP_LTimeMode_CompTime,
    &t,
    TRUE,           // pre_expressionB = TRUE --> ignore expression
    &value));
```

The same `pre_expressionB` parameter exists on `AEGP_GetLayerStreamValue`:

```cpp
AEGP_StreamVal2 val;
ERR(stream_suite->AEGP_GetLayerStreamValue(
    layerH,
    AEGP_LayerStream_OPACITY,
    AEGP_LTimeMode_CompTime,
    &t,
    TRUE,           // pre_expressionB
    &val,
    NULL));
```

---

## AEGP Expression API

The `AEGP_StreamSuite6` provides functions to read, write, enable, and disable expressions programmatically.

### Check Expression State

```cpp
A_Boolean enabled;
ERR(stream_suite->AEGP_GetExpressionState(plugin_id, streamH, &enabled));
```

### Enable/Disable Expression

```cpp
// Disable the expression on a stream
ERR(stream_suite->AEGP_SetExpressionState(plugin_id, streamH, FALSE));
```

### Read Expression Text

```cpp
AEGP_MemHandle expr_handleH = NULL;
ERR(stream_suite->AEGP_GetExpression(plugin_id, streamH, &expr_handleH));

if (expr_handleH) {
    A_UTF16Char *expr_textP = NULL;
    ERR(mem_suite->AEGP_LockMemHandle(expr_handleH, (void**)&expr_textP));
    // expr_textP is a null-terminated UTF-16 string
    ERR(mem_suite->AEGP_UnlockMemHandle(expr_handleH));
    ERR(mem_suite->AEGP_FreeMemHandle(expr_handleH));
}
```

### Write Expression Text

```cpp
// Set a wiggle expression on the stream
const A_UTF16Char expr[] = u"wiggle(5, 50)";
ERR(stream_suite->AEGP_SetExpression(plugin_id, streamH, expr));
```

### Check if Stream Can Vary Over Time

```cpp
A_Boolean can_vary;
ERR(stream_suite->AEGP_CanVaryOverTime(streamH, &can_vary));
// Returns FALSE if PF_ParamFlag_CANNOT_TIME_VARY is set
```

### Check if Stream IS Time-Varying

```cpp
A_Boolean is_timevarying;
ERR(stream_suite->AEGP_IsStreamTimevarying(streamH, &is_timevarying));
// Takes expressions into account -- returns TRUE if expression exists and is enabled,
// even if there are no keyframes
```

---

## Expression Errors and Their Effect on Plugins

When an expression on one of your parameters has an error (syntax error, runtime error, reference to a deleted property, etc.):

### What Happens

1. AE displays the expression error banner on the layer in the timeline
2. **The parameter reverts to its pre-expression (keyframe-interpolated) value**
3. Your plugin's render is called with the pre-expression value
4. The expression is automatically disabled by the expression parser

### Impact on Plugin Rendering

Your plugin does not receive any notification that an expression error occurred. You simply get the pre-expression value. This is usually harmless, but if the user was relying on the expression to keep values in a valid range, you may receive unexpected values.

### Expression Error State

From AEGP, you can detect that an expression was disabled due to an error by checking `AEGP_GetExpressionState`. A disabled expression that the user did not explicitly disable was likely disabled by an error.

### Defensive Coding

Always validate parameter values in your render code regardless of expression state:

```cpp
// Even if you set min/max on the slider, an expression could (before error)
// have been producing out-of-range values. Always clamp.
A_FpLong amount = params[PARAM_AMOUNT]->u.fs_d.value;
amount = A_MAX(0.0, A_MIN(100.0, amount));
```

---

## PF_Cmd_USER_CHANGED_PARAM and Expressions

This is one of the most common sources of confusion for plugin developers.

### The Problem

`PF_Cmd_USER_CHANGED_PARAM` (triggered by the `PF_ParamFlag_SUPERVISE` flag) fires **only when the user directly modifies a parameter value in the UI**. It does NOT fire when:

- An expression changes the value during playback
- A keyframe causes the value to change as the timeline scrubs
- Another effect modifies the value via AEGP

### Consequences

If you use `PF_Cmd_USER_CHANGED_PARAM` to enforce relationships between parameters (e.g., "slider B must always be >= slider A"), those constraints will NOT be enforced during playback when values come from keyframes or expressions.

### Example of the Problem

```cpp
// BAD PATTERN: This only works for direct user edits
case PF_Cmd_USER_CHANGED_PARAM:
    if (extra->param_index == PARAM_START) {
        if (params[PARAM_START]->u.fs_d.value >= params[PARAM_END]->u.fs_d.value) {
            params[PARAM_END]->u.fs_d.value = params[PARAM_START]->u.fs_d.value + 1;
            params[PARAM_END]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
        }
    }
    break;
```

During playback with keyframes or expressions, `PARAM_START` could exceed `PARAM_END` because `USER_CHANGED_PARAM` is not called.

### The Solution

Handle constraints in your render code instead:

```cpp
// GOOD PATTERN: Enforce constraints at render time
A_FpLong start_val = params[PARAM_START]->u.fs_d.value;
A_FpLong end_val = params[PARAM_END]->u.fs_d.value;
A_FpLong actual_start = A_MIN(start_val, end_val);
A_FpLong actual_end = A_MAX(start_val, end_val);
```

---

## Dependent Parameters and Expressions

### The Challenge

Sometimes you want parameter B to follow parameter A. With keyframes, this is impossible to enforce reliably at the parameter level.

### Pattern: Expression-Based Dependencies

One approach is to programmatically set an expression on the dependent parameter during sequence setup:

```cpp
case PF_Cmd_SEQUENCE_SETUP:
{
    // Use AEGP to set an expression on PARAM_B that references PARAM_A
    // This makes B always track A through the expression system
    AEGP_StreamRefH streamH = NULL;
    // ... acquire stream for PARAM_B ...
    const A_UTF16Char expr[] = u"effect(\"My Effect\")(\"Param A\")";
    ERR(stream_suite->AEGP_SetExpression(plugin_id, streamH, expr));
    ERR(stream_suite->AEGP_SetExpressionState(plugin_id, streamH, TRUE));
    // ... dispose stream ...
}
break;
```

### Pattern: PUI_ONLY Display Parameters

If the dependent parameter is display-only (does not affect rendering), use `PF_PUI_CONTROL` and update it in `PF_Cmd_UPDATE_PARAMS_UI`:

```cpp
// Parameter definition
def.ui_flags = PF_PUI_CONTROL;  // display only, does not affect rendering

// In PF_Cmd_UPDATE_PARAMS_UI
PF_ParamDef param_copy;
ERR(PF_CHECKOUT_PARAM(in_data, PARAM_DISPLAY, ...));
param_copy.u.fs_d.value = computed_display_value;
ERR(suites.ParamUtilsSuite()->PF_UpdateParamUI(in_data->effect_ref, PARAM_DISPLAY, &param_copy));
ERR(PF_CHECKIN_PARAM(in_data, &param_copy));
```

---

## Practical Patterns

### Pattern: Checking if a Parameter Has an Expression

From within an effect plugin (using AEGP suites during a command handler):

```cpp
A_Boolean HasExpression(PF_InData *in_data, PF_ParamIndex param_idx)
{
    A_Err err = A_Err_NONE;
    A_Boolean has_expr = FALSE;

    AEGP_EffectRefH effectH = NULL;
    AEGP_StreamRefH streamH = NULL;

    // Get effect ref
    ERR(suites.PFInterfaceSuite()->AEGP_GetNewEffectForEffect(
        plugin_id, in_data->effect_ref, &effectH));

    if (!err) {
        ERR(suites.StreamSuite()->AEGP_GetNewEffectStreamByIndex(
            plugin_id, effectH, param_idx, &streamH));
    }

    if (!err && streamH) {
        ERR(suites.StreamSuite()->AEGP_GetExpressionState(
            plugin_id, streamH, &has_expr));
        ERR(suites.StreamSuite()->AEGP_DisposeStream(streamH));
    }

    if (effectH) {
        ERR(suites.EffectSuite()->AEGP_DisposeEffect(effectH));
    }

    return has_expr;
}
```

### Pattern: Attaching Expressions at Apply Time

When your effect is first applied, set up linking expressions between parameters:

```cpp
case PF_Cmd_SEQUENCE_SETUP:
{
    // Only set expressions on first application, not on project load
    if (!in_data->sequence_data) {
        // Set up expressions via AEGP...
    }
}
break;
```

> **Warning**: Be cautious about setting expressions during `SEQUENCE_RESETUP` (project load). The expressions may already exist in the saved project. Check `AEGP_GetExpressionState` first.

---

## Pitfalls and Warnings

1. **Never assume `USER_CHANGED_PARAM` fires for all value changes.** It only fires for direct user interaction. Keyframe playback and expression evaluation do not trigger it.

2. **Expressions can produce values outside your parameter's defined min/max range.** While the UI clamps values, expressions bypass these limits. Always validate in render.

3. **Expression evaluation can be expensive.** Complex expressions that reference many properties can slow down renders. Your plugin has no control over this, but be aware that `PF_CHECKOUT_PARAM` on an expression-driven parameter may take longer than expected.

4. **`PF_ParamFlag_CANNOT_TIME_VARY` prevents both keyframes AND expressions.** If you want to allow expressions but not keyframes (or vice versa), this flag is too coarse. There is no flag that disables only keyframes while allowing expressions.

5. **Circular expression references crash AE.** If your plugin programmatically sets expressions, ensure they do not create circular dependencies (A depends on B depends on A).

6. **Expression language versions matter.** AE supports both the legacy ExtendScript expression engine and the newer JavaScript engine. Expression syntax differences can cause errors when projects move between systems.

7. **`PF_ParamFlag_CANNOT_INTERP` does NOT prevent expressions.** It only forces hold interpolation between keyframes. Expressions can still be applied and will produce smoothly varying values.

8. **`AEGP_IsStreamTimevarying` accounts for expressions.** This returns `TRUE` if the stream has an active expression, even with zero keyframes. Use this for correct time-varying detection in AEGP code.

9. **Do not modify `params[]` during render based on expression state.** The render call receives final values. Attempting to "correct" for expressions during render can cause unpredictable behavior and cache invalidation issues.

10. **Expressions can reference your effect by match name.** Users write expressions like `effect("ADBE MyEffect")("My Slider")`. Your effect's match name (from the PiPL) and parameter names (from `PF_Cmd_PARAMS_SETUP`) must be stable across versions, or user expressions will break.
