# Velocity

> 1 Q&A · source: AE plugin dev community Discord

### How can I set keyframe velocity and influence values using the AEGP API?

The AEGP_SetKeyframeTemporalEase function is broken and does not properly set keyframe velocity and influence values—these settings simply do not stick. You can set keyframe interpolation types (linear, hold, etc.) but not velocity. A workaround is to use AEGP_ExecuteScript() to execute JavaScript that sets the interpolations instead, though this approach is slower when dealing with many properties and layers.

*Tags: `aegp`, `api`, `keyframe`, `velocity`, `workaround`*

---
