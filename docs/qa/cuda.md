# Q&A: cuda

**8 entries** tagged with `cuda`.

---

## How do you pass custom data structs to GPU kernels in AE plugins?

Define the struct in both the kernel code and your C++ header, then use void pointers to transfer the data from AE to the kernel. Pointers work with CUDA. For Metal, the approach may differ. For OpenCL it works with a few caveats.

*Contributors: [**tlafo**](../contributors/tlafo/), [**James Whiffin**](../contributors/james-whiffin/) · Source: adobe-plugin-devs · 2023-07-29 · Tags: `gpu`, `cuda`, `metal`, `opencl`, `kernel`, `custom-struct`*

---

## Why does device-to-device buffer transfer in Vulkan have the same speed as host-to-device transfer when using external CUDA memory?

The transfer likely implicitly bounces off host memory or the CUDA buffer may actually be on host memory rather than device memory, even though the transfer is performed device-to-device. This can happen when Vulkan cannot reliably determine how to handle the caching of imported CUDA memory handles.

*Tags: `vulkan`, `cuda`, `gpu`, `memory`, `compute-cache`*

---

## What GPU APIs are recommended for After Effects plugins?

For GPU work in After Effects plugins, several options are viable: Vulkan is recommended for maximum control over performance and memory transfers, with MoltenVK providing Metal support on macOS and KosmicKrisp coming as a successor. WebGPU is easier to use and cross-platform but may not allow platform-specific optimizations. OpenGL is mature but deprecated and implemented via Metal/DirectX/Vulkan under the hood. A practical approach is to implement a CPU plugin that does GPU rendering underneath. Vulkan interoperability features allow importing textures from CUDA, OpenCL, DirectX, Metal, or OpenGL.

*Tags: `gpu`, `vulkan`, `metal`, `opengl`, `cuda`, `cross-platform`, `memory`*

---

## How can After Effects GPU features be utilized with Vulkan?

Vulkan has interoperability features that allow you to import textures from native After Effects GPU features (which may use CUDA, OpenCL, DirectX, or Metal) into Vulkan for processing, providing a way to leverage both native AE GPU capabilities and Vulkan's explicit control.

*Tags: `gpu`, `vulkan`, `metal`, `cuda`, `interop`*

---

## What are the risks of encoding data into dead pixels and retrieving them with sampleImage?

The solution is dependent on render format which can be tricky. CPU and GPU have different byte orders in After Effects (ARGB vs RGBA), and GPU formats like CUDA may use BGRA. Any swizzeling or color format conversion could mangle the data. Additionally, after threading changes in AE, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering yet.

*Tags: `gpu`, `memory`, `render-loop`, `threading`, `cuda`*

---

## Why is device-to-device buffer transfer between Vulkan and CUDA no faster than host-to-device transfer?

When importing a CUDA handle into Vulkan for external memory, the transfer may implicitly bounce through host memory if Vulkan cannot reliably determine how to handle the memory's caching. Additionally, the AE_cuda buffer may be allocated on the host rather than the device, which would explain why device-to-device transfer speeds match host-to-device speeds. Check whether the buffer is actually allocated on the device or if it's a host-side allocation.

*Tags: `vulkan`, `cuda`, `gpu`, `memory`, `cross-platform`*

---

## What should you assume about the data pointer in pF_effectworld when using GPU mode in After Effects plugins?

According to the SDK documentation, when using GPU mode (CUDA, Metal, or OpenCL), you must assume that the data pointer is NULL in the pF_effectworld structure. This means the pixel data is not available in system memory and you need to work with GPU-resident data instead.

*Tags: `gpu`, `cuda`, `metal`, `opencl`, `mfr`, `reference`*

---

## What are alternative GPU APIs for communicating with After Effects to access GPU data?

Vulkan, CUDA, and Metal are viable options for GPU communication with After Effects. Developers have successfully implemented CUDA and Metal to get GPU data from AE in GPU mode. Vulkan is recommended as a cross-platform alternative to DirectX, though Adobe has not officially supported Vulkan exposure to plugins at this time.

*Tags: `gpu`, `vulkan`, `cuda`, `metal`, `cross-platform`*

---
