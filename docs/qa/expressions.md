# Q&A: expressions

**6 entries** tagged with `expressions`.

---

## How can I synchronize two parameters so that changing one updates the other (e.g., radius and area)?

Full sync is not entirely possible due to AE's architecture. Problems include: (1) keyframable params may have contradictory interpolation settings, (2) 'current time' is ambiguous with nested comps and multi-frame rendering, (3) since AE 2015, you cannot change param values from the render thread. Workarounds: (1) Use an expression on one param, set programmatically via AEGP_SetExpression during UPDATE_PARAMS_UI. (2) Use PF_PUI_STD_CONTROL_ONLY to make a param non-keyframable. (3) Use PF_PUI_DISABLED to prevent manual changes while still allowing expressions. To get param values without downsample scaling, use AEGP_GetNewStreamValue instead of regular checkout.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-06-01 · Tags: `parameter-sync`, `expressions`, `update-params-ui`, `downsample`, `keyframes`*

---

## How can internally computed positions from a plugin be exposed to users in After Effects?

Internally computed positions can be exposed to users by outputting them through a point control parameter. This allows users to index the values in expressions or access them programmatically. One practical example is exporting bounding boxes as nulls, which is a technique used in projects like Vision (https://aescripts.com/vision).

*Tags: `params`, `ui`, `scripting`, `expressions`, `arb-data`*

---

## What is a method to pass text data from a render into an expression in After Effects?

One semi-reliable method is to encode and render dead pixels in your render output, then use a sampleImage expression to retrieve those pixels, which can then set the source text. This technique works with both regular layers and Null objects, though it has a few caveats.

*Tags: `scripting`, `expressions`, `render-loop`, `workaround`*

---

## How are colors represented in After Effects expressions?

In After Effects expressions, colors use a four-number array format: [red, green, blue, alpha]. All four values must be provided in expressions, or you can use color space conversions to generate the values. While the color data type supports four channels internally, not all color parameters expose the alpha channel for user editing.

```cpp
[red, green, blue, alpha]
```

*Tags: `params`, `expressions`, `scripting`*

---

## Can invisible effect parameters be accessed from expressions in After Effects?

Invisible parameters marked with PF_PUI_INVISIBLE cannot be accessed by name or match name in expressions, but they can be accessed by their index number. This was confirmed through experimentation where accessing by index worked successfully, while name and match name references returned property missing errors.

*Tags: `expressions`, `params`, `scripting`, `aegp`*

---

## How can you create dependent parameters that work correctly with keyframing and expressions in After Effects plugins?

When a parameter is keyframed or uses expressions, the PF_Cmd_USER_CHANGED_PARAM callback is not called during timeline scrubbing, making it difficult to synchronize dependent parameters. The PF_ParamFlag_SUPERVISE flag works for user-initiated changes but not for time-varying data. If the dependent parameter doesn't affect rendering (PUI_ONLY flag), you can use it. Otherwise, one practical solution is to use Motion Script expressions to express dependencies between sliders and attach these expressions during sequence setup, allowing the effect to use the computed parameters during render.

```cpp
params[SLIDER2]->u.fs_d.value = params[SLIDER1]->u.fs_d.value;
params[SLIDER2]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `keyframing`, `expressions`, `scripting`, `aegp`*

---
