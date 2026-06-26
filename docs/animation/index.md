# Animation & Time

Keyframes, time management, and time-varying behavior in AE plugins.

## Time in AE

AE uses a rational time system:
- `current_time` / `time_scale` = time in seconds
- `time_step` = duration of one frame

## Keyframes

Effect parameters can be keyframed by the user. Your plugin reads the interpolated value at render time — you don't need to handle keyframe interpolation yourself.

To **read keyframes programmatically** (AEGP only): use `AEGP_KeyframeSuite`.

## Time-Varying Output

If your effect's output depends on something **other than** the current parameter values and time (e.g., random noise, external data), do **not** set `PF_OutFlag_NON_PARAM_VARY`. This flag tells AE it can cache aggressively.

---

*9 documents in this section. Use **Search** (top of page) to find what you need.*
