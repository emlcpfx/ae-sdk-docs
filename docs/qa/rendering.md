# Q&A: rendering

**8 entries** tagged with `rendering`.

---

## Why does transform_world give bad results when dragging a slider, and what is the correct first argument?

The SDK documentation is wrong about putting in_data as the first argument to transform_world. The first argument is actually the effect_ref (in_data->effect_ref), but passing NULL instead works better. When using effect_ref, transform_world appears to check for interrupts, which causes it to halt the transformation during slider dragging for smoother interactive experience, but this results in incorrect output. Passing NULL avoids this interrupt check. Many suite callbacks accept NULL instead of effect_ref.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `transform-world`, `documentation-bug`, `effect-ref`, `interrupt`, `rendering`*

---

## How can I chain multiple transform_world calls, using the output of one as input for the next?

Never use the same buffer as both input and output of transform_world, as it reads and overwrites simultaneously, causing corrupted output. Never overwrite the input buffer (AE caches it). For repeated transforms: (1) Create a temp buffer, (2) transform input to temp, (3) transform temp to output, (4) transform output to temp, (5) repeat as needed, (6) copy to output if last result is in temp. However, matrix multiplication is extremely cheap (20-something operations for 3x3), while each transform_world render costs millions of operations per image. Always prefer multiplying matrices and doing a single render.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `transform-world`, `matrix-multiplication`, `buffer-management`, `performance`, `rendering`*

---

## Is there a way to show a progress bar in Premiere Pro for video filter / AE effect rendering?

In the AE SDK, there is a function for reporting render progress back to the host, which results in the standard AE progress bar being displayed/updated. It's unclear whether Premiere supports this. There is also an unofficial way used by some of Adobe's native effects to display custom progress, but the details may not be publicly disclosed.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2026-02-09 · Tags: `progress-bar`, `rendering`, `premiere-pro`, `ui-feedback`*

---

## Is there an open-source example of volumetric fractal noise implementation for After Effects?

Yes, there is a GitHub project by mes51 that implements volumetric fractal noise: https://github.com/mes51/VolumetricFractalNoise. This project serves as a reference implementation and workaround for volumetric fractal noise effects in After Effects plugins.

*Tags: `open-source`, `reference`, `gpu`, `rendering`*

---

## Why does a semitransparent layer appear differently in gamma 1.0 versus gamma 2.2 when composited over a black background?

In gamma 2.2, semitransparent layers appear the same whether composited over a black background or over nothing because gamma 2.2 rendering applies consistent color space conversions that make the visual difference negligible. In gamma 1.0 (linear color space), the same semitransparent layer looks completely different when placed over a black background versus transparency because linear blending treats black (0,0,0) and transparent areas fundamentally differently in the blend calculation. This is due to how alpha compositing and color space gamma correction interact—gamma 2.2 compresses the value range in a way that reduces the perceived difference, while linear gamma 1.0 preserves the full mathematical difference in how semitransparent pixels blend with the background.

*Tags: `rendering`, `color-space`, `compositing`, `alpha-blending`*

---

## What approach works well for implementing GPU acceleration in After Effects plugins?

A practical approach is to present your plugin as a CPU plugin to After Effects while implementing GPU rendering (Vulkan, OpenGL, or WebGPU) under the hood. This avoids compatibility issues while still leveraging GPU acceleration. Ensure that critical operations are actually executed on the GPU—falling back to CPU paths defeats the purpose.

*Tags: `gpu`, `vulkan`, `webgpu`, `optimization`, `rendering`*

---

## Is it feasible to build a 2D global illumination plugin using material property metadata effects on individual layers with a separate renderer effect?

Yes, this architecture is theoretically viable with the AE SDK. The pattern involves creating dummy effects on individual layers that only expose material parameters (emissive, diffuse, glass properties, etc.) without rendering output, while a separate renderer effect on an adjustment layer or solid discovers and interprets these metadata effects at render time. The key challenge is properly communicating cross-layer dependencies to AE's caching system so changes to materials trigger re-renders of the expensive global illumination computation while still benefiting from disk caching when unrelated elements change.

*Tags: `aegp`, `params`, `layer-checkout`, `caching`, `smart-render`, `rendering`*

---

## Why is motion blur incorrect when using PF_TransformWorld with fast rotation?

When using PF_TransformWorld with only two matrices (previous and current), motion blur artifacts can occur during fast rotation, causing particles to appear size-limited. The solution is to use multiple intermediate matrices for in-between values rather than just the start and end positions. Additionally, ensure that the PF_Rect *dest_rect is expanded to encompass the full area of all rotated samples to capture the complete motion blur region.

```cpp
PF_Err transform_world (
  PF_InData *in_data,
  PF_Quality quality,
  PF_ModeFlags m_flags,
  PF_Field field,
  const PF_EffectWorld *src_world,
  const PF_CompositeMode *comp_mode,
  const PF_MaskWorld *mask_world0,
  const PF_FloatMatrix *matrices,
  A_long num_matrices,
  Boolean src2dst_matrix,
  const PF_Rect *dest_rect,
  PF_EffectWorld *dst_world);
```

*Tags: `mfr`, `transform`, `motion-blur`, `rendering`, `output-rect`*

---
