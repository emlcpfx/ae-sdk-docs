# Enabling AE Native GPU Dispatch: The Three-Flag Handshake

> Why a GPU-capable plugin can silently keep rendering on the CPU, and the three things that all have to be true for After Effects to route a frame to your GPU render path.

## Symptom

Your plugin declares GPU support, After Effects accepts the GPU device during
`PF_Cmd_GPU_DEVICE_SETUP`, but every frame still routes to `PF_Cmd_SMART_RENDER`
(the CPU path) instead of `PF_Cmd_SMART_RENDER_GPU`. There is no error and no
warning -- the effect simply renders on the CPU. Diagnostic logging in the GPU
render entry point never fires.

## Root cause

AE's GPU rendering is a **three-stage handshake**, and missing any one stage
falls back to CPU silently:

1. **Global setup / PiPL** -- `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` must be set
   in `PF_Cmd_GLOBAL_SETUP` (`out_data->out_flags2`) and declared in the PiPL.
   This advertises that the plugin *can* render on the GPU at all.

2. **GPU device setup** -- the `PF_Cmd_GPU_DEVICE_SETUP` handler must accept the
   offered framework (CUDA / Metal / OpenCL) and set
   `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` on its `out_flags2`. This says the
   plugin accepted *this device*.

3. **SmartPreRender (per frame)** -- in `PF_Cmd_SMART_PRE_RENDER` the plugin
   must set `extra->output->flags |= PF_RenderOutputFlag_GPU_RENDER_POSSIBLE`.
   This tells AE that **this specific frame** can render on the GPU.

The trap is stage 3. It is easy to set `extra->output->flags` only along some
code path -- for example only when border expansion is active (to add
`PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS`) -- and leave the field at `0` for
the common case. With no `GPU_RENDER_POSSIBLE` bit, AE drops to the CPU smart
render for that frame even though the device was accepted.

## The fix

In `PF_Cmd_SMART_PRE_RENDER`, after computing the result and max rects, set the
flag on **every** PreRender call:

```cpp
// inside the SMART_PRE_RENDER handler
extra->output->flags |= PF_RenderOutputFlag_GPU_RENDER_POSSIBLE;
```

AE checks this per frame, so it cannot be set once and cached. If you only want
GPU on certain frames (mixed CPU/GPU plugin), set it conditionally -- but for an
all-GPU effect it must be unconditional.

## Checklist

For a frame to actually reach your GPU render code, all three must hold at once:

- [ ] `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` set in Global Setup and the PiPL.
- [ ] `PF_Cmd_GPU_DEVICE_SETUP` accepts a framework and sets the flag.
- [ ] `PF_RenderOutputFlag_GPU_RENDER_POSSIBLE` set in *every* SmartPreRender.

Missing any one produces a silent CPU fallback with no error. When debugging GPU
activation, add a log line at each of the three checkpoints so you can see which
stage is dropping the handshake.

## See also

- [GPU Render Model](gpu-render-model.md) -- enabling GPU render *replaces* the
  CPU path; there is no automatic per-frame fallback once you commit to it.
- [Metal Compute Shaders](metal-compute.md) -- the device-setup and dispatch
  patterns on macOS.

*Tags: `gpu`, `smart-render`, `pre-render`, `flags`, `silent-fallback`*
