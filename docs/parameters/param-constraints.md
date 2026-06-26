# Param Constraints

> 1 Q&A · source: AE plugin dev community Discord

### How do you restrict two float sliders relative to each other (e.g., Start cannot be >= End)?

Modifying param values in UserChangedParam works but causes problems with keyframing: a keyframe with an acceptable value can later become unacceptable, locking the value and preventing keyframe deletion. A better approach is to use the min()/max() pattern: treat the parameters as generic 'endpoints' and in your render code use min(A,B) as Start and max(A,B) as End.

*Tags: `keyframing`, `param-constraints`, `sliders`, `user-changed-param`*

---
