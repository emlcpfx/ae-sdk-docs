# Wgpu

> 1 Q&A · source: AE plugin dev community Discord

### What are the performance characteristics and trade-offs of WebGPU libraries like WGPU versus alternatives?

WGPU library has good compatibility but not great performance, with unavoidable bounds checking in shaders (unless using SPIRV shaders directly). Google's Dawn offers much better performance but is difficult to build. In practice, WebGPU-based plugins are usable but not super fast - performance depends on the effect logic, such as iterating over multiple frames.

*Tags: `gpu`, `performance`, `vulkan`, `webgpu`, `wgpu`*

---
