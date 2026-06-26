# GPU Rendering

GPU-accelerated rendering in AE plugins — CUDA, OpenCL, Metal, DirectX, Vulkan, and WebGPU.

## AE's Native GPU API

AE provides a GPU rendering path through `PF_GPUDeviceSuite`:

1. Set `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` in your PiPL
2. Handle `PF_Cmd_GPU_DEVICE_SETUP` — accept/reject GPU frameworks
3. Implement `PF_Cmd_SMART_RENDER_GPU` — process using GPU buffers

**Framework priority on Windows:**
- NVIDIA GPUs: AE offers **CUDA** first
- AMD/Intel: **OpenCL** or **DirectX**
- AE 2025+: May require `PF_OutFlag2_SUPPORTS_DIRECTX_RENDERING`

## Approaches

| Approach | Pros | Cons |
|----------|------|------|
| **AE GPU API** (CUDA/OpenCL) | Native integration, no data transfer overhead | Limited to AE's offered frameworks |
| **Vulkan Hybrid** | Works with masks, any GPU | CPU↔GPU transfer overhead |
| **WebGPU** (wgpu-native) | Cross-platform, modern API | Newer, less battle-tested |

## Key Guides

- [CPU/GPU Render Parity](cpu-gpu-parity.md) — Pixel format differences, algorithm parity checklist, Metal gotchas, SIMD guards
- [GPU Framework Notes](../guides/gpu-framework-notes.md) — CUDA vs OpenCL, framework selection, buffer management, alpha handling
- [Vulkan Architecture](../guides/vulkan-architecture.md) — Hybrid CPU/GPU rendering with Vulkan compute
- [WebGPU Integration](../guides/webgpu-ae-integration.md) — Using wgpu-native in AE plugins

---

*10 documents in this section. Use **Search** (top of page) to find what you need.*
