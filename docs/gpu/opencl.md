# Opencl

> 2 Q&As · source: AE plugin dev community Discord

### How do you pass custom data structs to GPU kernels in AE plugins?

Define the struct in both the kernel code and your C++ header, then use void pointers to transfer the data from AE to the kernel. Pointers work with CUDA. For Metal, the approach may differ. For OpenCL it works with a few caveats.

*Tags: `cuda`, `custom-struct`, `gpu`, `kernel`, `metal`, `opencl`*

---

### What should you assume about the data pointer in pF_effectworld when using GPU mode in After Effects plugins?

According to the SDK documentation, when using GPU mode (CUDA, Metal, or OpenCL), you must assume that the data pointer is NULL in the pF_effectworld structure. This means the pixel data is not available in system memory and you need to work with GPU-resident data instead.

*Tags: `cuda`, `gpu`, `metal`, `mfr`, `opencl`, `reference`*

---
