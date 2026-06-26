# Progress Banner / Re-Render Technique in After Effects Plugins

## Summary

Research into how to display a "Computing: X%" progress banner that updates while an effect is rendering, based on how certain plugins and Adobe's 3D Camera Tracker implement this.

## Core Technique: Hidden Parameter Re-Render Trick

The fundamental approach is: **change a hidden parameter to force AE to re-render the frame**, allowing you to draw updated progress banners each time.

### How It Works

1. Add a **hidden parameter** (not visible to user) to the effect
2. When the user triggers a long operation (e.g. presses a button), change the hidden parameter inside `PF_Cmd_USER_CHANGED_PARAM` -- this invalidates the cache and triggers a re-render
3. In `PF_Cmd_SMART_RENDER`, check the current state/progress and draw the appropriate banner (e.g. "0%", "43%", "done")
4. To trigger **additional** re-renders (for progress updates), you need a mechanism to keep changing the hidden parameter

### The Re-Render Trigger Problem

Parameters can **only** be changed during:
- `PF_Cmd_USER_CHANGED_PARAM`
- `PF_Cmd_EVENT`

You **cannot** change parameters inside `PF_Cmd_SMART_RENDER`. So you need another mechanism to keep triggering re-renders.

## Two Approaches Found

### Approach 1: Hidden Parameter + Idle Hook

- User presses button -> `PF_Cmd_USER_CHANGED_PARAM` fires -> change hidden param -> triggers re-render
- Background work communicates progress via shared memory (e.g. sequence data)
- An idle hook (`AEGP_RegisterIdleHook`) or similar mechanism checks progress and updates the hidden parameter to trigger more re-renders
- **Caveat**: Changing a parameter invalidates cached frames, so previous frames lose their cache

#### Potential Optimization
- Use AEGP to set a **keyframe** on the hidden parameter affecting only the current frame, so other frames' caches are preserved

### Approach 2: PF_Cmd_RESERVED3 + FORCE_RERENDER (3D Camera Tracker style)

- `PF_Cmd_RESERVED3` is an undocumented "idle" command selector that AE spam-calls on effects (similar to `AEGP_RegisterIdleHook` but effect-level)
- Inside `PF_Cmd_RESERVED3`, check if background work has progressed
- If yes, set `PF_OutFlag_FORCE_RERENDER` in `out_data->out_flags` -- this triggers AE to call `PF_Cmd_SMART_RENDER` again
- In `PF_Cmd_SMART_RENDER`, read current state and draw the appropriate banner
- **Advantage**: No need to change parameters, so possibly less cache invalidation
- **Disadvantage**: `PF_Cmd_RESERVED3` is undocumented/internal, may not be stable across versions

### Key Difference Between Approaches

| | Hidden Param | PF_Cmd_RESERVED3 |
|---|---|---|
| Public API | Yes (common practice) | No (internal/undocumented) |
| Cache invalidation | Yes (invalidates frames) | Possibly less |
| Access to PF_OutData | N/A | Yes (can set FORCE_RERENDER) |
| Idle hook alternative | AEGP_RegisterIdleHook (no PF_OutData access) | Built-in idle loop |
| Premiere Pro support | Yes (Adobe-recommended) | Different internal selectors in PPro |

## Architecture for Background Processing

Both approaches share the same pattern:
1. **Launch background work** (separate thread / analysis server) when user triggers the action
2. **Communicate state** between background thread and render thread via shared memory (sequence data, global state)
3. **Periodic check** (via idle hook or RESERVED3) reads the shared state
4. **Trigger re-render** (via param change or FORCE_RERENDER)
5. **SmartRender draws** the appropriate banner or final result based on current state

## Practical Notes

- Hidden parameters are a **very common practice** used in countless plugins, both internal and external (confirmed by community developers)
- Adobe themselves suggested the hidden parameter trick for Premiere Pro `FORCE_RERENDER` issues
- `PF_OutFlag_FORCE_RERENDER` does not work reliably in Premiere Pro -- use hidden parameter approach instead

## Sources

- Community developer discussions (March 2026) on AE plugin development
- [AE SDK docs: Parameter Supervision](https://ae-plugins.docsforadobe.dev/effect-details/parameter-supervision/#updating-parameter-values)
- [Adobe Community: Premiere Pro force re-render](https://community.adobe.com/questions-529/premiere-pro-force-re-render-31840)
