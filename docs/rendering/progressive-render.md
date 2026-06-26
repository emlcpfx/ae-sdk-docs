# Progressive / Accumulating Renderers in AE: Render the Whole Budget in One Call

> After Effects drives the render loop; a plugin cannot ask AE to re-invoke it for
> a frame it already rendered. To port a progressive accumulator (path tracer,
> sample-refining renderer), do the full sample budget in a single render call and
> hash every visible input into AE's cache key.

## Symptom

You port a progressive sample-accumulating renderer expecting the real-time
model: render one sample, bump a tick that invalidates AE's cache for this frame,
let AE re-call you, accumulate another sample, repeat until converged.

In practice the loop never advances. The sample count ticks to 1, AE caches the
noisy single-sample output, and the only way to coax more samples is to toggle a
parameter (which AE *does* invalidate for) -- and even then you get exactly one
more call before AE caches again. Every render call enters with one sample done,
resets, renders one sample, and exits. AE never re-requests the frame on its own.

## Root cause

AE's "varies without parameters" mechanism
(`PF_OutFlag_NON_PARAM_VARY` plus the `PF_OutFlag2_I_MIX_GUID_DEPENDENCIES`
cache-key mix-in path via `extra->cb->GuidMixInPtr`) lets a plugin force a
re-render by changing a value it hashes into the per-frame cache key. Bumping that
value invalidates the **next** cache lookup -- but **AE does not trigger that
lookup on its own.** Something external has to ask AE for the frame again: the
comp viewer redrawing, the user scrubbing the timeline, RAM preview advancing, a
parameter edit. A plugin cannot make AE re-request a frame it already has.

In short: `NON_PARAM_VARY` *invalidates* the cache; it does not *drive a redraw
loop*. Real-time-style progressive renderers that "keep refining on screen" do it
with a parallel render thread that pushes refined frames back through AEGP suite
calls and explicit redraw notifications -- substantially more host plumbing.

Two related caching traps show up on the way:

1. **Without the current frame time in the mix-in,** a static-camera frame at
   time T+1 hashes identically to time T (camera matrices and scalar params are
   unchanged), so AE returns T's cached pixels for T+1 -- it looks like the
   accumulator never reset for the new frame.

2. **Visibility-affecting toggles** (a denoise checkbox, a backend choice) must
   be hashed into the plugin's mix-in too. With `NON_PARAM_VARY` set, AE's normal
   param-driven invalidation is relaxed, so a toggle that isn't in your mix-in
   produces stale output.

## The fix

Render the full sample budget in **one** render call. Loop internally until the
budget is reached, with a cancel check between samples so scrubbing and camera
moves stay responsive. AE gets a converged frame in one shot, caches it, and only
re-invokes you when something the user can see changes -- at which point your
mix-in changes the key and AE asks for a fresh render.

```cpp
const int samples_this_call = std::max(1, budget - samples_done);
for (int s = 0; s < samples_this_call; ++s) {
    if (render_was_cancelled()) break;        // cooperative bail (PF_ABORT)
    render_one_sample_into(accum, frame_index++);
    blend_running_average(accum, /* sample index */ s, W, H);
    report_progress(double(s + 1) / samples_this_call);
}
```

No per-call tick bump is needed; AE is no longer in the inner loop. The cache-key
mix-in (fed to `GuidMixInPtr` in PreRender) must include everything that can
change the displayed image:

```cpp
mix(current_frame_time);    // each frame is fresh -- avoids stale static-camera cache
mix(view_inverse_matrix);   // camera move invalidates
mix(proj_inverse_matrix);
mix(scalar_params...);
mix(denoise_toggle);        // every visibility-affecting toggle
mix(backend_choice);
```

The plugin's own internal accumulation signature only controls *what you do* on a
given call; *whether AE calls you* is driven solely by the GuidMixIn. Anything not
in the mix-in is invisible to AE's invalidation.

## How to avoid it

When porting a progressive renderer to AE:

1. **Don't try to loop AE.** AE drives; the plugin reacts. Per render request, do
   as much work as you'll make the user wait for (with a cancel check between
   samples), then return a finished frame.
2. **Hash everything that affects the image into the GuidMixIn** -- time, camera,
   scalars, and *every* visibility-affecting toggle. A missing one leaves stale
   output in the viewer cache until the user perturbs the key some other way.
3. **With `NON_PARAM_VARY` set, the plugin owns invalidation.** Anything not in
   your mix-in is functionally invisible to AE.
4. **Cap the per-call budget.** A 1024-spp call at 4K will lock the UI for many
   seconds; expose a max-samples slider so the user can dial it down for
   interactive scrubbing and up for stills.
5. **Bound the accumulator memory.** Once the cap is reached, stop. The viewer's
   frame cache grows with cap x image size x number of distinct (time, params)
   keys.

## See also

- [Multi-Frame Rendering](../memory-threading/mfr.md) -- `GuidMixInPtr` usage and
  the cache-invalidation patterns it shares with this one.

*Tags: `render`, `cache-key`, `guid-mixin`, `non-param-vary`, `progressive`,
`path-tracer`*
