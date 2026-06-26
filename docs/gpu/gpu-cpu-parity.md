# GPU Kernels Must Replicate Output-Changing CPU Optimizations

> When a GPU kernel mirrors a CPU render path, every CPU shortcut that changes
> the *result* (not just the speed) has to be replicated in the kernel, or the
> two paths diverge visibly.

## Symptom

The GPU render of an effect looks subtly different from the CPU render of the
same frame with the same parameters. A common form: per-channel character drifts
(for example the color balance of procedural grain or noise differs), even
though the geometry and overall look are close.

## Root cause

CPU render code accumulates *correctness-affecting* optimizations over time, and
they are easy to forget when porting the kernel. A representative case: a CPU
noise generator auto-drops from two octaves to one when the feature scale falls
below ~2 pixels, because the second octave would be sub-pixel and wasted. The GPU
kernel always ran two octaves. At small scales (which include the default), the
CPU used one octave everywhere while the GPU used two -- different texture,
different per-channel balance.

The distinction that matters: there are two kinds of CPU optimization.

- **Output-changing** -- octave/LOD dropping, fast paths that skip work and
  thereby change values, clamping branches, response-curve breakpoints. These
  alter the pixels and **must** be mirrored in the kernel.
- **Output-preserving** -- a LUT instead of per-pixel math, SIMD vs scalar,
  cache-tiling. These produce (within float tolerance) identical values and are
  free to differ between paths.

## The fix

Port the output-changing branch into the kernel so both paths take the same
decision. For the octave example:

```cpp
// CPU and GPU must agree on this:
int octaves = (scale < 2.0f) ? 1 : 2;
```

Add the same threshold logic to each GPU backend (CUDA, Metal, OpenCL).

## How to avoid it

When writing a GPU kernel that parallels a CPU render path:

1. Audit the CPU code for every branch that changes *output values* and
   replicate each one in the kernel.
2. Auto-LOD, octave skipping, early-outs, and fast paths that produce different
   values all count -- not just the obviously math-heavy parts.
3. Test CPU vs GPU output across the parameter range, with special attention to
   defaults and edge cases (zero strength, sub-pixel scales, single-channel
   input). Per-channel effects expose drift that channel-symmetric effects hide.
4. Pure performance optimizations that yield identical values are fine to differ
   -- don't waste effort forcing the kernel to mimic a LUT bit-for-bit.

## See also

- [Channel Order Is Not Implied by the Format Enum](pixel-layout-rgba.md) -- the
  other classic CPU/GPU divergence: getting the channel order wrong in the
  kernel.

*Tags: `gpu`, `cpu`, `parity`, `noise`, `lod`, `rendering`*
