# Keyframe

> 8 Q&As · source: AE plugin dev community Discord

### How can I set keyframe velocity and influence values using the AEGP API?

The AEGP_SetKeyframeTemporalEase function is broken and does not properly set keyframe velocity and influence values—these settings simply do not stick. You can set keyframe interpolation types (linear, hold, etc.) but not velocity. A workaround is to use AEGP_ExecuteScript() to execute JavaScript that sets the interpolations instead, though this approach is slower when dealing with many properties and layers.

*Tags: `aegp`, `api`, `keyframe`, `velocity`, `workaround`*

---

### What does the dimensionL parameter represent in AEGP_GetKeyframeTemporalEase, and how should it be used?

The dimensionL parameter refers to the separate X, Y, and Z axes for properties where each axis can be controlled individually with different ease settings. For example, position and scale properties can have different ease settings per axis. The dimensionL ranges from 0 to (temporal_dimensionality - 1), where 0 typically represents a single dimension for properties like sliders, colors, or points that animate as one uniform property.

*Tags: `aegp`, `keyframe`, `params`*

---

### How does the A_Time structure work and how do you calculate the value and scale components?

A_Time is represented as a ratio where value/scale equals time in seconds. You can use any whole numbers that express the desired time as a numerator/denominator ratio. For example, {1500, 500} equals 1.5 seconds, and {94208, 25600} equals approximately 3.68 seconds. To set A_Time from a double value like 1.875 seconds, you need to find appropriate whole numbers whose ratio equals that decimal value.

*Tags: `aegp`, `keyframe`, `params`, `time`*

---

### How do you insert keyframes at specific times using AEGP_InsertKeyframe with the correct A_Time values?

Use the AEGP_InsertKeyframe function from the KeyframeSuite3, passing a properly formatted A_Time structure where the ratio of value to scale equals your desired time in seconds. For example, to insert a keyframe at 1.2 seconds, you could use A_Time {12, 10} or {60, 50}. The function signature is: AEGP_InsertKeyframe(position_streamH, AEGP_LTimeMode_LayerTime, &timePT, &new_indexL) where timePT is your A_Time structure.

```cpp
ERR(suites.KeyframeSuite3()->AEGP_InsertKeyframe(position_streamH,
  AEGP_LTimeMode_LayerTime,
  &timePT,
  &new_indexL));
```

*Tags: `aegp`, `keyframe`, `params`, `time`*

---

### How can I set time-variant stream values for every frame in an AEGP plugin without creating keyframes at each frame?

According to shachar carmi, expressions are the only way to modify parameter values between keyframes without creating keyframes at each frame. Using AEGP_SetLayerStreamValue() or the Keyframe Suite will automatically create a keyframe at the current time if the stream is already keyframed. On AE 13.5+, only the UI thread can safely change parameter values. The SetStreamValue() function sets the value at the current item time, but if keyframes already exist on that stream, it will create or modify a keyframe at that time.

*Tags: `aegp`, `keyframe`, `params`, `threading`*

---

### What is the recommended approach for animating parameters in AEGP plugins?

Use the Keyframe Suite to animate parameters. This is the proper API for managing keyframe-based animation. Direct SetStreamValue() calls will create keyframes on already-keyframed streams, so the Keyframe Suite should be used for intentional parameter animation.

*Tags: `aegp`, `keyframe`, `params`*

---

### How can I toggle a stopwatch or create keyframes for slider parameters programmatically in an Effects plugin?

You can create keyframes for any parameter using AEGP_KEYFRAMESUITE3. Most AEGP suites work for both AEGP plugins and Effects plugins, including the keyframe suite. The "CheesyCheese" sample project demonstrates how to implement this functionality.

*Tags: `aegp`, `effects`, `keyframe`, `params`*

---

### Is there a sample project demonstrating how to use AEGP_KEYFRAMESUITE3 for keyframe creation?

The "CheesyCheese" sample project is included in the Adobe After Effects SDK and demonstrates how to implement keyframe creation using AEGP_KEYFRAMESUITE3. This example shows practical usage of the keyframe suite for both AEGP and Effects plugins.

*Tags: `aegp`, `keyframe`, `reference`, `sample-code`*

---
