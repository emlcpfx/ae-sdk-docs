# GPU Render Replaces CPU Render -- There Is No Automatic Per-Frame Fallback

> Once a plugin advertises GPU support and AE selects a GPU device, AE routes
> *all* frames to the GPU render path. The CPU render is not called as a
> fallback, and the GPU render entry point does not receive the rich host-scene
> context the CPU path gets. Commit to GPU only when the GPU path can serve
> 100% of frames.

## Symptom

A multi-mode plugin where only one mode (or only some cases) is GPU-wired
regresses the moment GPU support is turned on. With the GPU device selected, only
the GPU-wired cases render; the others show the **input footage unchanged**
(passthrough). Selecting "CPU" as the render device breaks *every* mode (also
passthrough), because the host still routes everything to the GPU entry point.

## Root cause

The render dispatch in After Effects is an either/or decided by whether the
plugin advertises GPU support, not by a per-frame negotiation:

- If GPU is advertised and a GPU device is active, AE sends **every** frame to
  `PF_Cmd_SMART_RENDER_GPU`. The CPU `PF_Cmd_SMART_RENDER` is **never called**.
- The GPU render command receives a minimal context: device pixel buffers and
  scalar parameters. It does **not** get the full smart-render context the CPU
  path has -- no live host camera, no arbitrary sibling layers, none of the
  host-scene plumbing the CPU render relied on.

So any mode that needs host-scene data (the live camera, a source layer it pulls
from, projector layers) simply cannot run in the GPU command, and there is **no
automatic CPU fallback** to catch it. Whatever the GPU path can't handle becomes
passthrough or garbage.

## The fix

Do **not** advertise GPU support unless the GPU render can fully serve *every*
frame the host will route to it -- all modes, and the "CPU device" case too.

For a plugin where only some modes are GPU-able, you have two options:

- **Keep GPU off (interim).** Leave the GPU capability flag unset. The GPU code
  can stay compiled but dormant; all modes use the correct CPU render. No
  regression.

- **Add a real per-frame opt-out and broaden the GPU context (proper).** If you
  want partial GPU, you have to build the escape hatch yourself, because AE
  won't:
  - Decide per frame whether this frame is GPU-serviceable; when it is not,
    take the CPU path for that frame. (In raw SDK terms this means *not* setting
    `PF_RenderOutputFlag_GPU_RENDER_POSSIBLE` in SmartPreRender for frames you
    intend to render on the CPU -- see
    [AE Native GPU Dispatch](ae-native-gpu-dispatch.md).)
  - For GPU frames that need sibling-layer data, check those layers out in
    SmartPreRender and pass the pixels into the GPU render yourself. The live
    host camera and projector-style dependencies generally remain out of reach
    in the GPU command -- modes that need them must stay on the CPU.

## How to avoid it

Before turning on GPU support, confirm the GPU render handles 100% of frames, or
that you have built an explicit per-frame fallback. The GPU render context is
intentionally minimal (device pixels + params); modes that need the live camera
or arbitrary sibling layers cannot live there. Partial-GPU plugins need a
per-frame decision, not a static "supports GPU" capability flag.

## See also

- [AE Native GPU Dispatch](ae-native-gpu-dispatch.md) -- the per-frame
  `GPU_RENDER_POSSIBLE` flag that is the lever for routing a frame to GPU or CPU.

*Tags: `gpu`, `render`, `smart-render`, `pre-render`, `porting`*
