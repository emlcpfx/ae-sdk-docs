# Kernel

> 1 Q&A · source: AE plugin dev community Discord

### How do you pass custom data structs to GPU kernels in AE plugins?

Define the struct in both the kernel code and your C++ header, then use void pointers to transfer the data from AE to the kernel. Pointers work with CUDA. For Metal, the approach may differ. For OpenCL it works with a few caveats.

*Tags: `cuda`, `custom-struct`, `gpu`, `kernel`, `metal`, `opencl`*

---
