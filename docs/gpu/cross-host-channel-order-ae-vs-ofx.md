# Cross-Host Channel Order: AE is BGRA, OFX is RGBA -- Branch, Don't Hardcode

> If one kernel source ships to both After Effects and an OFX host, there is **no
> single correct channel order**. AE hands GPU kernels BGRA lanes; OFX hands them
> RGBA. A hardcoded order is correct on exactly one host and swaps red and blue on
> the other -- which is why this bug oscillates instead of getting fixed.

## Symptom

A per-channel GPU effect (grain, per-channel gain, tint, chromatic aberration)
comes out with red and blue swapped -- but only on one host. Fix it, and some
time later the *other* host is swapped. Fix that, and the first one breaks again.

Each fix looks correct in isolation, passes review, and ships.

## Why one constant cannot work

| | Lane `.x` | Lane `.z` | Source |
|---|---|---|---|
| **After Effects** native GPU world | Blue | Red | `PF_PixelFormat_GPU_BGRA128`, read via `GetGPUWorldData` |
| **OFX** (Resolve, Nuke, Flame) | Red | Blue | RGBA by spec |

Both are correct, for their host. So the question "is the buffer RGBA or BGRA?"
has no answer for a cross-host plugin -- only "which host am I rendering for right
now?" does.

If your adapter/framework hands kernels the host's device pointers **without
swizzling** (the common case, since swizzling a 4K frame twice per render is pure
waste), the lane order reaching the kernel is the host's, and the kernel must
adapt.

## The oscillation trap

This is the failure pattern to recognize, taken from a real plugin that shipped
the same swap three times:

| | Change | AE | OFX |
|---|---|---|---|
| Fix #1 | hardcode BGRA | correct | **swapped** |
| Fix #2 | hardcode RGBA | **swapped** | correct |
| Fix #3 | branch on a host flag | correct | correct |

Fix #2 even shipped a written rule -- *"treat GPU float4 as RGBA on every host,
do not special-case AE"* -- which turned the broken constant into documented
policy, so the next developer had a citation telling them to reintroduce the bug.

**A channel-order fix that changes a constant rather than introducing a
conditional is not a fix.** It is the same bug, moved to the other host.

## The fix

Carry the host's lane order as a flag into the kernel and branch on it -- on the
**read and the write**, using the same lanes for both:

```cpp
// Host side: set from whichever host you are rendering for.
// AE -> true. OFX -> false.
params.bgra = host_is_after_effects ? 1 : 0;

// Kernel: read
float pR = bgra ? px.z : px.x;
float pG = px.y;
float pB = bgra ? px.x : px.z;
float pA = px.w;

// ... per-channel work: grain, gain, tint ...

// Kernel: write back into the lanes we read from
out.x = bgra ? outB : outR;
out.y = outG;
out.z = bgra ? outR : outB;
out.w = pA;
```

Note the effect is *not* a color-flipped image: the kernel reads a lane, modifies
it, and writes back to the same lane, so pixels pass through in place. What gets
transposed is which **parameters** land on which channel -- the red grain
amount/size/seed driving the blue channel, and the luma coefficients (`0.2126` R /
`0.0722` B) landing on the wrong channels. Debug it by looking at the effect's
per-channel behavior, not at the frame's overall color.

## Catch it at build time, not in the host

The reason this ships repeatedly is that the usual CPU/GPU parity test runs the
GPU path in **one** lane order -- so it proves one host and is silent about the
other. Both hardcoded kernels pass it.

Make the parity test run **twice**:

1. **Pass 1** -- RGBA lanes, input as-is. (The OFX model.)
2. **Pass 2** -- BGRA lanes: swap R and B in the input buffer, tell the kernel
   it is BGRA, then swap R and B back in the output. (The AE model.)

Both passes must match the same CPU reference. Then:

- a kernel hardcoding RGBA fails pass 2,
- a kernel hardcoding BGRA fails pass 1,
- only a kernel that honors the flag passes both.

This needs no GPU host and no AE round-trip -- it is a pure host-side buffer
transform around your existing kernel launch, so it runs in CI.

## How to avoid it

- **Never hardcode a lane order in a kernel that ships to more than one host.**
  Branch on a host-supplied flag.
- **Treat a constant-only channel-order fix as a red flag.** If the diff has no
  conditional in it, it is moving the bug, not removing it. Check the git history
  first: a *third* channel-order commit on the same kernel means the approach is
  wrong, not the polarity.
- **Per-channel effects are the canary.** Symmetric operations (blur, erode,
  luma, composite) cannot see a lane swap and will happily pass every test.
  Validate any new GPU path with an effect that treats channels differently.
- **A green parity run only proves the lane order it ran.** Say which one that
  was before calling the path verified.

## See also

- [Verify GPU Channel Order Empirically](pixel-layout-rgba.md) -- probe the lane
  order of *your* pipeline instead of inferring it from the format enum.
- [CPU/GPU Render Parity](cpu-gpu-parity.md) -- AE's BGRA float4 layout beside the
  CPU ARGB layout, and the other common CPU/GPU drift sources.

*Tags: `gpu`, `pixel-format`, `channel-order`, `rgba`, `bgra`, `ofx`, `cross-host`, `regression`*
