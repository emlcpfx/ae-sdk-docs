# Time

> 2 Q&As · source: AE plugin dev community Discord

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
