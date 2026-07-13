# PreRender's GPU Gate Must Match SmartRender's -- A Mismatched Decline Silently Kills GPU

> `PF_RenderOutputFlag_GPU_RENDER_POSSIBLE` is set in PreRender, but the
> conditions that *use* the GPU path live in SmartRender. If PreRender declines
> GPU on a condition SmartRender ignores, the effect renders on the CPU forever,
> with no error, no warning, and no visible sign that anything is wrong. It just
> runs slow.

## Symptom

The plugin advertises GPU support correctly. `PF_Cmd_GPU_DEVICE_SETUP` is called
and succeeds. AE offers a framework you accept. And yet
`PF_Cmd_SMART_RENDER_GPU` is *never* invoked -- every frame goes to the CPU
`PF_Cmd_SMART_RENDER`.

Switching the project between "Mercury Software Only" and "Mercury GPU
Acceleration" changes nothing, which is the tell that made this so confusing to
diagnose: it looks exactly like the host ignoring your GPU support, when in fact
*you* declined it.

## Root cause

GPU eligibility is decided per-frame in PreRender:

```cpp
if (some_condition_that_forces_cpu) {
    // decline: don't set the flag
} else {
    extra->output->flags |= PF_RenderOutputFlag_GPU_RENDER_POSSIBLE;
}
```

Those "force CPU" conditions are usually features that only the CPU path
implements -- a debug visualisation, a CPU-only filter, a legacy mode. The trap
is that the *same* condition is re-derived independently in SmartRender, and the
two derivations drift.

Real example (an AE keyer with a CPU-only Edge Control debug visualisation):

```cpp
// PreRender  -- decides whether GPU is even offered
edge_debug_active = (edge_debug != EDGE_DEBUG_OFF);

// SmartRender -- decides whether the debug visualisation actually runs
edge_debug_active = (edge_control_enabled && edge_debug != EDGE_DEBUG_OFF);
```

PreRender was missing the `edge_control_enabled` conjunct. A user who picked a
debug mode, then switched Edge Control *off*, left the dropdown on a non-Off
value with the feature disabled. From then on:

- **PreRender** saw `edge_debug != OFF` -> declined GPU on every frame.
- **SmartRender** saw `!edge_control_enabled` -> skipped the debug visualisation
  entirely and rendered normally.

So the plugin permanently lost GPU acceleration *to protect a feature it wasn't
even running*. Nothing in the UI indicated the effect had silently dropped to
CPU. It was found only by logging the decision.

## Fix

Derive the condition **once** and share it, or -- if the two commands genuinely
cannot share code -- treat the pair as a unit and assert they match. The rule:

> Any predicate that gates `GPU_RENDER_POSSIBLE` in PreRender must be *exactly*
> the predicate SmartRender uses to run the CPU-only feature. If PreRender is
> stricter, you lose GPU for nothing. If it is looser, SmartRender's CPU-only
> feature gets invoked on the GPU path, which is worse.

## Diagnostic: log the decision *and the reason*

This class of bug is invisible without instrumentation, because the failure mode
is "everything works, just slowly". Log the branch you took, the framework AE
offered, and the bit depth:

```cpp
const char* gpu_decision = "";
if (cpu_only_feature_active) {
    gpu_decision = "DECLINED (cpu-only feature active)";
} else {
    extra->output->flags |= PF_RenderOutputFlag_GPU_RENDER_POSSIBLE;
    gpu_decision = "OFFERED (GPU_RENDER_POSSIBLE set)";
}

std::stringstream ss;
ss << "PreRender: " << gpu_decision
   << " | what_gpu=" << (int)extra->input->what_gpu
   << " (0=NONE 1=OPENCL 2=METAL 3=CUDA 4=DIRECTX)"
   << " | bitdepth=" << (int)extra->input->bitdepth;
Log(ss.str());
```

Two things make this actually usable:

1. **Log the reason string, not just a boolean.** "DECLINED" alone sends you
   hunting through every gate; "DECLINED (cpu-only feature active)" points at
   the line.
2. **Write it to a file, not just `OutputDebugStringA`.** `OutputDebugString`
   output is invisible unless DebugView happens to be attached to a live AE,
   which means you cannot inspect the GPU handshake after the fact. Append to a
   file under `%TEMP%` and the whole negotiation is there to read.

Correlating the two logs is what cracks it -- when PreRender says
`DECLINED (cpu-only feature active)` on the same frame that SmartRender reports
that feature as inactive, the contradiction is the bug.

## Related: `PF_OutFlag2_SUPPORTS_DIRECTX_RENDERING` is a capability
declaration, not a requirement

While chasing the above, the DirectX flag is a tempting suspect, and the folklore
comment "required for AE 2025+ GPU rendering" has been copy-pasted widely. Two
observations from AE 2025 on Windows with an NVIDIA GPU:

- A plugin that **sets** the flag while implementing only CUDA/OpenCL still gets
  offered `PF_GPU_Framework_CUDA` and renders on GPU normally.
- A plugin that **omits** it is also offered CUDA.

So on NVIDIA/Windows it is neither required for CUDA nor fatal to it. It is an
*opt-in capability declaration*: AE queries `PF_Cmd_GPU_DEVICE_SETUP` once per
framework, and you accept one by setting `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32`
in `out_flags2` for that query. Adobe's own `SDK_Invert_ProcAmp` sets the DirectX
flag *and* ships a real DirectX backend (`Util/DirectXUtils.{h,cpp}`).

Guidance: **do not set the flag unless you actually implement a DirectX
backend.** Declaring a framework you cannot service is at best meaningless and at
worst invites the host to hand you a device you must refuse. (Behaviour on
AMD/Intel, where DirectX is a more likely offer, is untested here.)

## GPU worlds are premultiplied

Related, and worth stating loudly because the opposite was written down in these
docs for a long time: **AE GPU world buffers are PREMULTIPLIED, exactly like CPU
world buffers.**

Measured 2026-07-13 (RTX 3090, AE 2025, CUDA) by writing raw values straight into
a GPU output world and reading the composite back, in an isolated comp over black
with no other layers and no other effects:

| band | wrote | premult predicts | straight predicts | CPU read | GPU read |
|------|-------|------------------|-------------------|----------|----------|
| A | `rgb=0.5, a=0.5`  | 0.50 | 0.25  | 0.50 | **0.50** |
| B | `rgb=0.25, a=0.5` | 0.25 | 0.125 | 0.25 | **0.25** |
| C | `rgb=0, a=0`      | 0    | 0     | 0    | 0 |

The GPU renders byte-identically to the premultiplied CPU world. Straight would
have been half as bright.

Your kernel chain may still be straight *internally* — that is often the natural
way to write it (unpremultiply once on input, work in straight space, composite
freely). **Just premultiply on the way out.** Shipping straight RGB into a buffer
AE reads as premultiplied makes every semi-transparent pixel too bright: a hot
fringe on exactly the soft edges (hair, motion blur), which reads as a *keying*
artifact rather than an alpha bug, and can survive for months.

### How to measure it yourself, without fooling yourself

This is easy to get wrong, and I got it wrong twice before getting it right.

1. **Isolated comp.** One layer, your effect, nothing else. No solids, no
   adjustment layers, no other effects, black comp background, transparency grid
   off. A grey solid behind the layer makes the straight and premultiplied models
   *both* fit the numbers, and you will confidently pick the wrong one.
2. **Include a fully transparent band** (`rgb=0, a=0`). It *measures* the
   background instead of letting you assume it. Assuming the background is black
   is exactly the mistake that produces a false "straight" reading.
3. **Use only VALID premultiplied values** (`rgb <= alpha`) in the probe. A pixel
   like `rgb=1.0, a=0.5` implies a straight colour of 2.0; AE's handling of that
   is undefined and it will hand you a meaningless number.
4. **Make the CPU and GPU probes visually distinct** — e.g. CPU paints horizontal
   bands, GPU paints vertical. AE's frame cache does **not** reliably invalidate
   when you switch render engines, so "switching to Software Only" can silently
   hand you back the cached GPU frame. Orientation makes that impossible to miss;
   numbers alone will not.

*Tags: `gpu`, `prerender`, `smart-render`, `cuda`, `directx`, `premultiplied`, `debugging`*
