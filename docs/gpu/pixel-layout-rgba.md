# Verify GPU Channel Order Empirically -- Don't Infer It From the Format Enum

> Channel-order bugs in GPU kernels (red and blue swapped) are common and easy to
> "fix" in the wrong direction. The enum name a host hands you is a format *tag*,
> not a guarantee of the lane order your kernel reads. Prove the order with a
> pixel test before you change a kernel.

## Symptom

A GPU render path shows red and blue channels swapped relative to the CPU path.
It only appears for effects that treat channels *differently* -- per-channel
grain, per-channel gain, tint, chromatic aberration. Channel-symmetric
operations (blur, erode, luma, monochrome math) look fine, which is exactly why
the bug ships: the symmetric tests pass.

The dangerous part is the "fix": someone flips the kernel from one channel order
to the other based on a guess, it appears to fix one effect, and now a second
swap is layered on top of the first. The next per-channel effect breaks again.

## Root cause

Two failure modes feed this:

1. **Treating the SDK format enum as proof of memory order.** A host-side pixel
   format constant (such as a `..._GPU_BGRA128` tag) describes how the host
   *labels* the world, but the precise `float4` lane order a given kernel
   receives -- after any adapter swizzle, texture view, or framework conversion
   in between -- is what actually matters. The label and the lanes can differ
   depending on the path the data took to reach your kernel.

2. **Reading an SDK sample as an assertion.** A sample that applies the BT.601
   red coefficient (`0.299`) to `pixel.z` *looks* like it's asserting "`.z` is
   red." It is only correct for the exact buffer layout that sample receives.
   Copying that assumption into a different pipeline (one with its own
   adapter/swizzle) is how the wrong order gets baked in.

## The fix

Determine the lane order with an **empirical pixel test**, then write the kernel
to that proven order and stop guessing:

- Feed pure red (`R=1, G=0, B=0, A=1`) through the actual GPU path.
- Read back the first lane. If it is `1.0`, the kernel sees red in `.x`
  (RGBA-style); if it is `0.0` and the third lane is `1.0`, the kernel sees red
  in `.z` (BGRA-style).
- Lock the kernel's reads and writes to whatever the probe proves, end to end:

```cpp
// Read in the proven order (here: lanes confirmed R,G,B,A by the probe)
float pR = px.x, pG = px.y, pB = px.z, pA = px.w;
// ... per-channel work ...
out.x = pR; out.y = pG; out.z = pB; out.w = pA;
```

> Note: After Effects' native GPU worlds are tagged `PF_PixelFormat_GPU_BGRA128`
> and, read directly from `GetGPUWorldData`, the lanes are **BGRA**
> (`.x=Blue, .y=Green, .z=Red, .w=Alpha`). See
> [CPU/GPU Render Parity](cpu-gpu-parity.md) for that detail. The lesson here is
> not "always RGBA" -- it is that the *only* authority for the lane order in
> *your* kernel is a pixel test of *your* pipeline, because adapters, texture
> views, and framework layers can re-order lanes before your kernel runs.

## How to avoid it

- **Probe, don't infer.** A pure-red input test is the authority; the enum name
  and the SDK samples are not.
- **Before "fixing" a suspected channel swap, check git history for prior
  channel-order changes on the same kernel.** If the order worked before a
  recent swap, the swap *is* the bug -- revert it instead of stacking another.
- **Use a per-channel effect as the canary.** Test every new GPU path with an
  effect that touches each channel differently (grain, per-channel gain, tint)
  before declaring it correct. Symmetric effects mask the bug.

## See also

- [CPU/GPU Render Parity](cpu-gpu-parity.md) -- AE's BGRA float4 layout and the
  CPU ARGB layout side by side.
- [Metal Compute Shaders](metal-compute.md) -- why Metal's `.r/.g/.b/.a`
  swizzles are just aliases for `.x/.y/.z/.w` and don't track real channels.

*Tags: `gpu`, `pixel-format`, `channel-order`, `rgba`, `bgra`, `regression`*
