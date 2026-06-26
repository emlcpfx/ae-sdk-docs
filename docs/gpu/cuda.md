# CUDA

> 11 Q&As · source: AE plugin dev community Discord

### How do you pass custom data structs to GPU kernels in AE plugins?

Define the struct in both the kernel code and your C++ header, then use void pointers to transfer the data from AE to the kernel. Pointers work with CUDA. For Metal, the approach may differ. For OpenCL it works with a few caveats.

*Tags: `cuda`, `custom-struct`, `gpu`, `kernel`, `metal`, `opencl`*

---

### Why does device-to-device buffer transfer in Vulkan have the same speed as host-to-device transfer when using external CUDA memory?

The transfer likely implicitly bounces off host memory or the CUDA buffer may actually be on host memory rather than device memory, even though the transfer is performed device-to-device. This can happen when Vulkan cannot reliably determine how to handle the caching of imported CUDA memory handles.

*Tags: `compute-cache`, `cuda`, `gpu`, `memory`, `vulkan`*

---

### What GPU APIs are recommended for After Effects plugins?

For GPU work in After Effects plugins, several options are viable: Vulkan is recommended for maximum control over performance and memory transfers, with MoltenVK providing Metal support on macOS and KosmicKrisp coming as a successor. WebGPU is easier to use and cross-platform but may not allow platform-specific optimizations. OpenGL is mature but deprecated and implemented via Metal/DirectX/Vulkan under the hood. A practical approach is to implement a CPU plugin that does GPU rendering underneath. Vulkan interoperability features allow importing textures from CUDA, OpenCL, DirectX, Metal, or OpenGL.

*Tags: `cross-platform`, `cuda`, `gpu`, `memory`, `metal`, `opengl`, `vulkan`*

---

### How can After Effects GPU features be utilized with Vulkan?

Vulkan has interoperability features that allow you to import textures from native After Effects GPU features (which may use CUDA, OpenCL, DirectX, or Metal) into Vulkan for processing, providing a way to leverage both native AE GPU capabilities and Vulkan's explicit control.

*Tags: `cuda`, `gpu`, `interop`, `metal`, `vulkan`*

---

### What are the risks of encoding data into dead pixels and retrieving them with sampleImage?

The solution is dependent on render format which can be tricky. CPU and GPU have different byte orders in After Effects (ARGB vs RGBA), and GPU formats like CUDA may use BGRA. Any swizzeling or color format conversion could mangle the data. Additionally, after threading changes in AE, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering yet.

*Tags: `cuda`, `gpu`, `memory`, `render-loop`, `threading`*

---

### Why is device-to-device buffer transfer between Vulkan and CUDA no faster than host-to-device transfer?

When importing a CUDA handle into Vulkan for external memory, the transfer may implicitly bounce through host memory if Vulkan cannot reliably determine how to handle the memory's caching. Additionally, the AE_cuda buffer may be allocated on the host rather than the device, which would explain why device-to-device transfer speeds match host-to-device speeds. Check whether the buffer is actually allocated on the device or if it's a host-side allocation.

*Tags: `cross-platform`, `cuda`, `gpu`, `memory`, `vulkan`*

---

### What should you assume about the data pointer in pF_effectworld when using GPU mode in After Effects plugins?

According to the SDK documentation, when using GPU mode (CUDA, Metal, or OpenCL), you must assume that the data pointer is NULL in the pF_effectworld structure. This means the pixel data is not available in system memory and you need to work with GPU-resident data instead.

*Tags: `cuda`, `gpu`, `metal`, `mfr`, `opencl`, `reference`*

---

### What are alternative GPU APIs for communicating with After Effects to access GPU data?

Vulkan, CUDA, and Metal are viable options for GPU communication with After Effects. Developers have successfully implemented CUDA and Metal to get GPU data from AE in GPU mode. Vulkan is recommended as a cross-platform alternative to DirectX, though Adobe has not officially supported Vulkan exposure to plugins at this time.

*Tags: `cross-platform`, `cuda`, `gpu`, `metal`, `vulkan`*

---

### Are AE SDK compositing suites (transfer_rect, FillMatteSuite) available in GPU render paths?

No. The `PF_WorldTransformSuite1` functions (`composite_rect`, `transfer_rect`, `blend`) and `PF_FillMatteSuite2` (`fill`, `premultiply`) operate on `PF_EffectWorld` CPU buffers only. In GPU SmartRender, pixel data lives on the device — the `data` pointer in `PF_EffectWorld` is NULL. GPU plugins must implement Porter-Duff compositing math in their own CUDA/Metal/OpenCL kernels. There is no GPU equivalent of `transfer_rect`.

*Tags: `compositing`, `cuda`, `gpu`, `metal`, `opencl`, `smart-render`, `transfer-rect`*

---

### What is the correct formula for Porter-Duff Over compositing in GPU kernels?

For premultiplied float4 RGBA inputs, the standard formula is:

```
out.rgb = top.rgb + bottom.rgb * (1 - top.a)
out.a   = top.a + bottom.a * (1 - top.a)
```

However, many GPU pipelines operate in **straight (unpremultiplied) space** internally because blur kernels produce alpha-weighted intermediates. In that case, use the straight-space formula:

```
out_a = top.a + bottom.a * (1 - top.a)
out.rgb = (top.rgb * top.a + bottom.rgb * bottom.a * (1 - top.a)) / out_a
out.a = out_a
```

The straight-space formula is equivalent to unpremultiplying the result of the premultiplied formula. Choose based on your pipeline's internal data convention. If your blur kernels output straight RGB, use the straight-space composite. If your pipeline maintains premultiplied throughout, use the premultiplied formula.

*Tags: `compositing`, `cuda`, `gpu`, `metal`, `opencl`, `porter-duff`, `premultiplied`, `straight-alpha`*

---

### Why do GPU blur+composite pipelines commonly use straight (unpremultiplied) space internally?

Three reasons: (1) Blurring in straight space prevents dark halos at semi-transparent edges — when premultiplied RGB is averaged, transparent pixels contribute black, darkening edges. Straight-space blur averages the actual colors weighted by alpha, producing clean color extension. (2) Alpha expansion (increasing alpha to extend edges) can simply set a new alpha value without scaling RGB — in premultiplied space, you'd need to scale RGB proportionally to maintain the invariant `RGB ≤ alpha`. (3) The straight-space compositing formula naturally handles the alpha-to-RGB relationship via the `/ out_a` normalization. The typical GPU pipeline pattern is: `Input (premult) → Unpremultiply → [blur, expand, composite in straight] → Premultiply → Output (premult)`.

*Tags: `blur`, `compositing`, `cuda`, `gpu`, `halo`, `premultiplied`, `straight-alpha`*

---
