# Parameter Flags: PF_ParamFlag Reference

## Overview

When adding a parameter to an After Effects effect plugin via `PF_ADD_PARAM` (or one of its convenience wrappers like `PF_ADD_SLIDER`, `PF_ADD_CHECKBOX`, etc.), you can specify behavioral flags through the `PF_ParamDef.flags` field. These flags are defined as the `PF_ParamFlags` type (an `A_long`) in `AE_Effect.h` and control keyframing, UI presentation, supervision, and backward compatibility.

Flags are combined using bitwise OR and passed to the parameter definition before calling `PF_ADD_PARAM`.

## Complete Flag Enumeration

```c
enum {
    PF_ParamFlag_RESERVED1                        = 1 << 0,  // 0x01 - Reserved, do not use
    PF_ParamFlag_CANNOT_TIME_VARY                 = 1 << 1,  // 0x02
    PF_ParamFlag_CANNOT_INTERP                    = 1 << 2,  // 0x04
    PF_ParamFlag_RESERVED2                        = 1 << 3,  // 0x08 - Reserved (old WANTS_UPDATE)
    PF_ParamFlag_RESERVED3                        = 1 << 4,  // 0x10 - Reserved (old SEPARATE)
    PF_ParamFlag_COLLAPSE_TWIRLY                  = 1 << 5,  // 0x20
    PF_ParamFlag_SUPERVISE                        = 1 << 6,  // 0x40
    PF_ParamFlag_START_COLLAPSED                  = PF_ParamFlag_COLLAPSE_TWIRLY,  // 0x20 (alias)
    PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS       = 1 << 7,  // 0x80
    PF_ParamFlag_LAYER_PARAM_IS_TRACKMATTE        = 1 << 7,  // 0x80 (shared bit, layer params only)
    PF_ParamFlag_EXCLUDE_FROM_HAVE_INPUTS_CHANGED = 1 << 8,  // 0x100
    PF_ParamFlag_SKIP_REVEAL_WHEN_UNHIDDEN        = 1 << 9   // 0x200
};
typedef A_long PF_ParamFlags;
```

## Summary Table

| Flag | Bit | Value | Purpose |
|------|-----|-------|---------|
| `CANNOT_TIME_VARY` | 1 | 0x02 | Disable keyframing entirely |
| `CANNOT_INTERP` | 2 | 0x04 | Disable smooth interpolation between keyframes |
| `COLLAPSE_TWIRLY` / `START_COLLAPSED` | 5 | 0x20 | Start parameter collapsed; dynamic twirl control |
| `SUPERVISE` | 6 | 0x40 | Receive `PF_Cmd_USER_CHANGED_PARAM` notifications |
| `USE_VALUE_FOR_OLD_PROJECTS` | 7 | 0x80 | Use `value` (not `dephault`) for missing params in old projects |
| `LAYER_PARAM_IS_TRACKMATTE` | 7 | 0x80 | Mark layer param as track matte (Premiere only, same bit) |
| `EXCLUDE_FROM_HAVE_INPUTS_CHANGED` | 8 | 0x100 | Skip this param in `PF_HaveInputsChangedOverTimeSpan` |
| `SKIP_REVEAL_WHEN_UNHIDDEN` | 9 | 0x200 | Do not auto-reveal when un-hidden |

## Detailed Flag Reference

---

### PF_ParamFlag_CANNOT_TIME_VARY (0x02)

Prevents the parameter from being keyframed. When set, no keyframe stopwatch icon appears in the timeline for this parameter. The value remains constant for the entire duration of the layer.

**When to use:** For parameters that configure the effect's behavior globally and should not change over time, such as quality settings, algorithm selection popups, or one-time configuration values.

```c
PF_ADD_POPUP(
    "Mode",
    3,                               // num_choices
    1,                               // default
    "Normal|Fast|Quality",
    PF_ParamFlag_CANNOT_TIME_VARY,   // no keyframing
    MY_PARAM_MODE);
```

> **Note:** This flag is permanent once set during `PF_Cmd_PARAMS_SETUP`. It cannot be changed dynamically later.

> **Crash warning — never set this on `PF_Param_ARBITRARY_DATA`.** An ARB param is an animatable *stream*; `CANNOT_TIME_VARY` declares "no stream" and contradicts the arb-data-stream AE builds in `FLT_MakeStreamFromParamDef`. AE throws while constructing it and double-throws during the unwind → `std::terminate` → `abort`, i.e. AE crashes (`FATAL_APP_EXIT`, mis-attributed to `sentry.dll`) the instant the effect is applied. Register ARB params with `flags = 0`. See `parameters/param-registration-crashes-on-apply.md`.

---

### PF_ParamFlag_CANNOT_INTERP (0x04)

Disables smooth interpolation between keyframes. The parameter can still be keyframed, but values jump discretely from one keyframe to the next (hold keyframe behavior). The user can still apply "no interpolation" or "discontinuous" interpolation modes.

**When to use:** For parameters where intermediate interpolated values are meaningless, such as a popup menu index or a mode selector that you want to be keyframeable but not smoothly animated.

```c
PF_ADD_POPUP(
    "Shape",
    4,
    1,
    "Circle|Square|Triangle|Hexagon",
    PF_ParamFlag_CANNOT_INTERP,
    MY_PARAM_SHAPE);
```

> **Note:** `CANNOT_INTERP` does not prevent keyframing. It only prevents smooth interpolation between keyframes. If you want to prevent keyframes altogether, use `CANNOT_TIME_VARY`.

---

### PF_ParamFlag_COLLAPSE_TWIRLY / PF_ParamFlag_START_COLLAPSED (0x20)

These are the same flag value. `PF_ParamFlag_START_COLLAPSED` is an alias for `PF_ParamFlag_COLLAPSE_TWIRLY`.

When set during `PF_Cmd_PARAMS_SETUP`, the parameter's twirly arrow in the Effect Controls Window (ECW) starts in the collapsed (twirled-up) state when the effect is first applied.

Starting in AE 4.0, this flag can also be **dynamically toggled** during `PF_Cmd_UPDATE_PARAMS_UI` and `PF_Cmd_USER_CHANGED_PARAM` to programmatically expand or collapse parameter groups based on user input.

**When to use:** For parameter groups that contain advanced or rarely-used options. Collapsing them by default keeps the Effect Controls panel clean.

```c
// For a parameter group -- start collapsed
def.flags = PF_ParamFlag_START_COLLAPSED;
PF_ADD_TOPIC("Advanced Options", MY_GROUP_START);
```

> **Important:** For this flag to work on parameter groups (topics), you must also set the appropriate bit in `PF_ParamDef.ui_flags`. The SDK header states: *"If you want a parameter group to honor the PF_ParamFlag_COLLAPSE_TWIRLY or PF_ParamFlag_START_COLLAPSED flag, set this bit. Otherwise, all parameter groups will default to being twirled open."*

---

### PF_ParamFlag_SUPERVISE (0x40)

Enables `PF_Cmd_USER_CHANGED_PARAM` notifications for this parameter. When the user modifies a supervised parameter, After Effects sends `PF_Cmd_USER_CHANGED_PARAM` to your plugin with the index of the changed parameter available in the extra data.

**This flag is mandatory for `PF_Param_BUTTON` parameters.** Buttons have no stored value -- their entire purpose is to trigger a notification callback.

**When to use:**
- Implementing parameter dependencies (e.g., changing one popup shows/hides other parameters)
- Validating parameter combinations and clamping values
- Updating UI state in response to user input
- Required for all button parameters

```c
PF_ADD_POPUP(
    "Algorithm",
    3,
    1,
    "Standard|Advanced|Custom",
    PF_ParamFlag_SUPERVISE | PF_ParamFlag_CANNOT_TIME_VARY,
    MY_PARAM_ALGORITHM);
```

Handling the notification:

```c
case PF_Cmd_USER_CHANGED_PARAM:
{
    PF_UserChangedParamExtra *extra =
        (PF_UserChangedParamExtra *)in_data->extra;

    if (extra->param_index == MY_PARAM_ALGORITHM) {
        // React to the algorithm change
        err = UpdateParameterVisibility(in_data, out_data, params);
    }
    break;
}
```

> **Warning:** Every supervised parameter causes an extra round-trip to your plugin when the user changes it. Do not set `PF_ParamFlag_SUPERVISE` on parameters you do not need to respond to.

---

### PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS (0x80)

This is one of the most critical flags for maintaining backward compatibility when adding new parameters to a shipping plugin.

#### The Problem

When you release version 2 of your plugin with a new parameter that did not exist in version 1, users who open projects saved with version 1 will be missing that parameter. By default, After Effects initializes the missing parameter using the `dephault` field from `PF_ParamDef`. But sometimes you need a different value for old projects to preserve the original rendering behavior.

#### How It Works

When this flag is set on a parameter:
- **New applications of the effect** (or Reset to Defaults): Uses the `dephault` field as the initial value (normal behavior).
- **Loading an old project** where this parameter did not exist: Uses the `value` field instead of `dephault`.

This lets you set the `dephault` to the ideal starting value for new users while setting `value` to whatever preserves the old rendering behavior.

**Valid for:** All `PF_Param` types **except `PF_Param_LAYER`**.

#### Example Scenario

You originally shipped a blur plugin without a "Quality" option. Version 2 adds a "Quality" popup with Low/Standard/High. For new users, "High" is the best default. But old projects were rendering at the equivalent of "Standard," so loading them with "High" would change their appearance.

```c
PF_ParamDef def;
AEFX_CLR_STRUCT(def);

def.flags = PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS;

// New users get "High" quality (index 3)
def.u.pd.dephault = 3;

// Old projects get "Standard" quality (index 2) to preserve their look
def.u.pd.value = 2;

PF_ADD_POPUP(
    "Quality",
    3,
    1,
    "Low|Standard|High",
    def.flags,
    MY_PARAM_QUALITY);
```

> **Critical Warning:** If you add parameters to a shipping plugin and do not use this flag where appropriate, old projects may render differently after the update. This is the most common cause of "my plugin update broke old projects" reports. Always ask: what value should this parameter have to make old projects render identically to the previous version?

---

### PF_ParamFlag_LAYER_PARAM_IS_TRACKMATTE (0x80)

Shares the same bit value (0x80) as `PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS`. This is intentional and not a conflict because:

- `USE_VALUE_FOR_OLD_PROJECTS` is explicitly **not valid** for `PF_Param_LAYER` parameters
- `LAYER_PARAM_IS_TRACKMATTE` is **only valid** for `PF_Param_LAYER` parameters

When set on a layer parameter, it indicates the layer should be presented as a track matte with applied filters. This flag is used by **Premiere Pro** and is **ignored by After Effects**.

---

### PF_ParamFlag_EXCLUDE_FROM_HAVE_INPUTS_CHANGED (0x100)

When set, this parameter is excluded from the check performed by `PF_HaveInputsChangedOverTimeSpan()`. This SmartFX function lets you ask After Effects whether any of your inputs have changed over a given time range, which is useful for caching optimizations.

**When to use:** For parameters that do not affect rendering output, such as UI-only controls or preview quality settings.

```c
PF_ADD_CHECKBOX(
    "Show Overlay",
    "",
    FALSE,
    PF_ParamFlag_EXCLUDE_FROM_HAVE_INPUTS_CHANGED,
    MY_PARAM_SHOW_OVERLAY);
```

---

### PF_ParamFlag_SKIP_REVEAL_WHEN_UNHIDDEN (0x200)

When a hidden parameter is later unhidden (via `AEGP_SetDynamicStreamFlag` with `AEGP_DynStreamFlag_HIDDEN`), After Effects normally "reveals" it by twirling open parent groups and scrolling the Effect Controls panel to make the parameter visible. Setting this flag suppresses that automatic reveal behavior.

**When to use:** For parameters that are frequently hidden and shown as part of normal UI workflow, where the automatic scrolling and twirling would be jarring. Use only during param setup.

```c
def.flags = PF_ParamFlag_SKIP_REVEAL_WHEN_UNHIDDEN;
```

---

## Reserved Flags

Three flags are reserved and must not be used:

| Flag | Bit | History |
|------|-----|---------|
| `PF_ParamFlag_RESERVED1` | 0 (0x01) | Unknown original purpose |
| `PF_ParamFlag_RESERVED2` | 3 (0x08) | Was the old `PF_ParamFlag_WANTS_UPDATE`, never used in practice |
| `PF_ParamFlag_RESERVED3` | 4 (0x10) | Was the old `PF_ParamFlag_SEPARATE`, replaced by `PF_PUI_ECW_SEPARATOR` |

Setting these reserved bits may cause undefined behavior in current or future AE versions.

## Common Flag Combinations

### Supervised Non-Keyframeable Popup

The most common pattern for mode/algorithm selectors:

```c
PF_ADD_POPUP(
    "Blend Mode",
    5,
    1,
    "Normal|Add|Multiply|Screen|Overlay",
    PF_ParamFlag_SUPERVISE | PF_ParamFlag_CANNOT_TIME_VARY,
    MY_PARAM_BLEND_MODE);
```

### Button (Must Be Supervised)

```c
PF_ADD_BUTTON(
    "Actions",
    "Reset to Defaults",
    0,
    PF_ParamFlag_SUPERVISE,   // Required for all buttons
    MY_PARAM_RESET_BUTTON);
```

### Backward-Compatible New Parameter with Supervision

```c
PF_ParamDef def;
AEFX_CLR_STRUCT(def);

def.flags = PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS | PF_ParamFlag_SUPERVISE;

def.u.sd.value    = 0;     // Old projects: 0 (preserves old rendering)
def.u.sd.dephault = 50;    // New applications: 50 (better default)
def.u.sd.valid_min = 0;
def.u.sd.slider_min = 0;
def.u.sd.valid_max = 100;
def.u.sd.slider_max = 100;

PF_ADD_SLIDER("Intensity", 0, 100, 0, 100, 50, MY_PARAM_INTENSITY);
```

### Cache-Friendly UI-Only Parameter

A parameter that triggers UI updates but should not cause re-renders:

```c
def.flags = PF_ParamFlag_SUPERVISE
          | PF_ParamFlag_CANNOT_TIME_VARY
          | PF_ParamFlag_EXCLUDE_FROM_HAVE_INPUTS_CHANGED;

PF_ADD_CHECKBOX(
    "Show Preview Overlay",
    "Enabled",
    FALSE,
    def.flags,
    MY_PARAM_SHOW_OVERLAY);
```

### Collapsed Advanced Group

```c
// Start the advanced options group, collapsed by default
{
    PF_ParamDef def;
    AEFX_CLR_STRUCT(def);
    def.flags = PF_ParamFlag_START_COLLAPSED;
    PF_ADD_TOPIC("Advanced", MY_GROUP_ADV_START);
}

// Children of the group...
PF_ADD_FLOAT_SLIDER("Threshold", 0.0, 1.0, 0.0, 1.0, 0.0, 0.5,
                    PF_Precision_HUNDREDTHS, 0, 0, MY_PARAM_THRESHOLD);

PF_END_TOPIC(MY_GROUP_ADV_END);
```

## Dynamic Twirl State Control

During `PF_Cmd_UPDATE_PARAMS_UI` or `PF_Cmd_USER_CHANGED_PARAM`, you can toggle the collapsed state of parameter groups:

```c
case PF_Cmd_UPDATE_PARAMS_UI:
{
    PF_ParamDef param_copy;
    AEFX_CLR_STRUCT(param_copy);

    ERR(PF_CHECKOUT_PARAM(in_data, MY_GROUP_ADV_START,
                          in_data->current_time,
                          in_data->time_step,
                          in_data->time_scale,
                          &param_copy));

    PF_Boolean should_collapse = /* your logic here */;

    if (should_collapse) {
        param_copy.flags |= PF_ParamFlag_COLLAPSE_TWIRLY;
    } else {
        param_copy.flags &= ~PF_ParamFlag_COLLAPSE_TWIRLY;
    }

    param_copy.uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;

    ERR(PF_CHECKIN_PARAM(in_data, &param_copy));
    break;
}
```

## Pitfalls and Warnings

> **Pitfall: Param flags vs. UI flags.** `PF_ParamFlags` and `PF_ParamUIFlags` (also called `ui_flags`) are separate fields on `PF_ParamDef`. A common mistake, especially with `PF_ADD_ARBITRARY2`, is putting param flags into the `ui_flags` argument or vice versa. `PF_ParamFlag_CANNOT_TIME_VARY` and `PF_ParamFlag_SUPERVISE` are param flags. `PF_PUI_CONTROL` and `PF_PUI_ECW_SEPARATOR` are UI flags. Putting them in the wrong field will silently fail.

> **Pitfall: Forgetting SUPERVISE on buttons.** `PF_Param_BUTTON` requires `PF_ParamFlag_SUPERVISE`. Without it, clicking the button does nothing because your plugin never receives `PF_Cmd_USER_CHANGED_PARAM`.

> **Pitfall: USE_VALUE_FOR_OLD_PROJECTS and layer params.** `PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS` (0x80) shares its bit with `PF_ParamFlag_LAYER_PARAM_IS_TRACKMATTE` (0x80). Do not set 0x80 on a layer parameter expecting backward-compatibility behavior.

> **Pitfall: Adding parameters without USE_VALUE_FOR_OLD_PROJECTS.** When you add a new parameter in a plugin update, old projects will initialize it to the `dephault` value. If the default does not match the behavior of the old version, old projects will render differently. Always use this flag when adding parameters to a plugin that has already shipped.

> **Pitfall: RESERVED bits.** Do not use bits 0, 3, or 4. They are marked reserved in the SDK header and setting them may cause problems.

## See Also

- `AE_Effect.h` -- Flag definitions and extensive inline documentation
- `PF_Cmd_USER_CHANGED_PARAM` -- Notification sent for supervised parameters
- `PF_Cmd_UPDATE_PARAMS_UI` -- Where you can dynamically toggle `COLLAPSE_TWIRLY`
- `AEGP_SetDynamicStreamFlag` -- For hiding/unhiding parameters (relates to `SKIP_REVEAL_WHEN_UNHIDDEN`)
- `PF_HaveInputsChangedOverTimeSpan` -- Relates to `EXCLUDE_FROM_HAVE_INPUTS_CHANGED`
