# GPU Memory Management in AE Plugins

> CPU-to-GPU data transfer patterns, the `PF_GPUDeviceSuite1` API, rowbytes handling during transfers, host (pinned) memory, and understanding when AE gives you GPU pointers vs CPU pointers.

## Overview

After Effects manages GPU memory allocation for plugins through `PF_GPUDeviceSuite1` (defined in `AE_EffectGPUSuites.h`). This suite handles device memory allocation, host (pinned) memory allocation, GPU effect world creation, and buffer pointer retrieval. All GPU memory used by plugins should be allocated through this suite so AE can track memory budgets and purge caches when the GPU runs low.

---

## PF_GPUDeviceSuite1 Complete Reference

Defined in `AE_EffectGPUSuites.h`, acquired via:

```cpp
AEFX_SuiteScoper<PF_GPUDeviceSuite1> gpuSuite =
    AEFX_SuiteScoper<PF_GPUDeviceSuite1>(in_dataP,
        kPFGPUDeviceSuite, kPFGPUDeviceSuiteVersion1, out_dataP);
```

### Device Enumeration

| Function | Description |
|----------|-------------|
| `GetDeviceCount(effect_ref, &count)` | Number of GPU devices AE supports |
| `GetDeviceInfo(effect_ref, device_index, &info)` | Fills `PF_GPUDeviceInfo` with device, context, and queue handles |

### Exclusive Device Access

| Function | Description |
|----------|-------------|
| `AcquireExclusiveDeviceAccess(effect_ref, device_index)` | Lock device for exclusive use. For CUDA: calls `cuCtxPushCurrent`. |
| `ReleaseExclusiveDeviceAccess(effect_ref, device_index)` | Release lock. For CUDA: calls `cuCtxPopCurrent`. |

> **Note**: Full GPU plugins (those using `PF_Cmd_SMART_RENDER_GPU`) always hold exclusive access. These calls are only needed if you do GPU work outside the GPU render entry point.

### Device Memory

| Function | Description |
|----------|-------------|
| `AllocateDeviceMemory(effect_ref, device_index, size_bytes, &ptr)` | Allocates GPU-resident memory. Underlying call: `cuMemAlloc` (CUDA), `clCreateBuffer` (OpenCL), `CreateCommittedResource` (DX12). |
| `FreeDeviceMemory(effect_ref, device_index, ptr)` | Frees previously allocated device memory |
| `PurgeDeviceMemory(effect_ref, device_index, size_bytes, &bytes_purged)` | Emergency purge of AE's GPU cache. Use only when you cannot allocate enough memory. |

### Host (Pinned) Memory

| Function | Description |
|----------|-------------|
| `AllocateHostMemory(effect_ref, device_index, size_bytes, &ptr)` | Allocates page-locked (pinned) host memory. Underlying call: `cuMemHostAlloc` (CUDA), `malloc` (OpenCL), `CreateCommittedResource` on upload heap (DX12). |
| `FreeHostMemory(effect_ref, device_index, ptr)` | Frees previously allocated host memory |
| `PurgeHostMemory(effect_ref, device_index, size_bytes, &bytes_purged)` | Emergency purge of host-side GPU memory |

### GPU Effect Worlds

| Function | Description |
|----------|-------------|
| `CreateGPUWorld(effect_ref, device_index, width, height, par, field, pixel_format, clear, &worldP)` | Creates a GPU-resident effect world. `pixel_format` must be a GPU format (`PF_PixelFormat_GPU_BGRA128`). |
| `DisposeGPUWorld(effect_ref, worldP)` | Frees a plugin-created GPU world. You may only dispose worlds your plugin created. |
| `GetGPUWorldData(effect_ref, worldP, &pixPP)` | Returns the GPU buffer address for the world |
| `GetGPUWorldSize(effect_ref, worldP, &size_bytes)` | Returns total data size in bytes |
| `GetGPUWorldDeviceIndex(effect_ref, worldP, &device_index)` | Returns which GPU device owns this world |

---

## PF_GPUDeviceInfo Structure

```cpp
typedef struct {
    PF_GPU_Framework device_framework;
    PF_Boolean       compatibleB;

    void* platformPV;                  // cl_platform_id (OpenCL only)
    void* devicePV;                    // CUdevice / cl_device_id / MTLDevice / ID3D12Device
    void* contextPV;                   // CUcontext / cl_context
    void* command_queuePV;             // CUstream / cl_command_queue / MTLCommandQueue / ID3D12CommandQueue
    void* offscreen_opengl_contextPV;  // CGLContextObj / HGLRC (primary device only)
    void* offscreen_opengl_devicePV;   // HDC (primary device only)
} PF_GPUDeviceInfo;
```

### Casting by Framework

| Framework | `devicePV` cast | `contextPV` cast | `command_queuePV` cast |
|-----------|----------------|-------------------|----------------------|
| CUDA | `CUdevice` | `CUcontext` | `CUstream` |
| OpenCL | `cl_device_id` | `cl_context` | `cl_command_queue` |
| Metal | `id<MTLDevice>` | (not used) | `id<MTLCommandQueue>` |
| DirectX 12 | `ID3D12Device*` | (not used) | `ID3D12CommandQueue*` |

---

## When PF_EffectWorld.data is NULL

In GPU render mode (`PF_Cmd_SMART_RENDER_GPU`), the `data` pointer in `PF_EffectWorld` is **NULL**. The pixel data resides entirely on the GPU device.

```cpp
// This will crash in GPU mode:
PF_Pixel *pixels = (PF_Pixel*)output_worldP->data;  // NULL!

// Correct approach -- get the GPU pointer:
void *gpu_ptr = nullptr;
gpuSuite->GetGPUWorldData(in_dataP->effect_ref, output_worldP, &gpu_ptr);
```

The `PF_EffectWorld` structure still has valid metadata in GPU mode:

| Field | Valid in GPU mode? | Description |
|-------|-------------------|-------------|
| `data` | **NO** -- NULL | CPU pixel pointer, not available |
| `width` | Yes | Image width in pixels |
| `height` | Yes | Image height in pixels |
| `rowbytes` | Yes | Row stride in bytes (including padding) |
| `pix_aspect_ratio` | Yes | Pixel aspect ratio |

### What GetGPUWorldData Returns

The pointer returned by `GetGPUWorldData` depends on the framework:

| Framework | Pointer type | Description |
|-----------|-------------|-------------|
| CUDA | `CUdeviceptr` (cast to `void*`) | Device memory from `cuMemAlloc`. Cast to `float*` or `float4*`. |
| OpenCL | `cl_mem` (cast to `void*`) | Buffer object. Cast to `cl_mem` for kernel arguments. |
| Metal | `id<MTLBuffer>` (cast to `void*`) | Metal buffer. Cast to `id<MTLBuffer>`. |
| DirectX 12 | `ID3D12Resource*` (cast to `void*`) | Committed resource. Cast to `ID3D12Resource*`. |

---

## CPU-to-GPU Data Transfer Patterns

### When You Need Transfers

Most GPU effects do not need explicit CPU-GPU transfers. AE provides GPU-resident input buffers and expects GPU-resident output. However, you need transfers for:

- **Lookup tables (LUTs)** -- CPU-side data that the kernel needs
- **Parameter buffers** -- Struct data passed to compute shaders
- **CPU fallback** -- Downloading GPU results for CPU post-processing
- **Custom data** -- Precomputed tables, noise seeds, gradient maps

### CUDA: cuMemcpyHtoD / cuMemcpyDtoH

```cpp
// Upload a LUT to GPU
float lut[256];
// ... fill lut ...

void *gpu_lut = nullptr;
gpuSuite->AllocateDeviceMemory(in_dataP->effect_ref,
    device_index, sizeof(lut), &gpu_lut);

cudaMemcpy(gpu_lut, lut, sizeof(lut), cudaMemcpyHostToDevice);

// Use gpu_lut in kernel...

// Download results (rare -- usually not needed)
float4 *cpu_buffer = (float4*)malloc(width * height * sizeof(float4));
cudaMemcpy(cpu_buffer, gpu_output, width * height * sizeof(float4),
           cudaMemcpyDeviceToHost);

// Cleanup
gpuSuite->FreeDeviceMemory(in_dataP->effect_ref, device_index, gpu_lut);
```

### Using Pinned (Host) Memory for Faster Transfers

Pinned memory enables asynchronous DMA transfers and avoids an extra copy through a staging buffer:

```cpp
// Allocate pinned host memory
void *pinned_ptr = nullptr;
gpuSuite->AllocateHostMemory(in_dataP->effect_ref,
    device_index, data_size, &pinned_ptr);

// Fill the pinned buffer on CPU
memcpy(pinned_ptr, source_data, data_size);

// Transfer to device (faster than from pageable memory)
void *device_ptr = nullptr;
gpuSuite->AllocateDeviceMemory(in_dataP->effect_ref,
    device_index, data_size, &device_ptr);

cudaMemcpy(device_ptr, pinned_ptr, data_size, cudaMemcpyHostToDevice);

// Cleanup
gpuSuite->FreeHostMemory(in_dataP->effect_ref, device_index, pinned_ptr);
// ... use device_ptr, then FreeDeviceMemory when done
```

### OpenCL: clEnqueueWriteBuffer / clEnqueueReadBuffer

```cpp
cl_command_queue queue = (cl_command_queue)device_info.command_queuePV;

// Upload
float lut[256];
cl_mem gpu_lut = clCreateBuffer(
    (cl_context)device_info.contextPV,
    CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR,
    sizeof(lut), lut, &cl_err);

// Or write explicitly:
clEnqueueWriteBuffer(queue, gpu_lut, CL_TRUE,
    0, sizeof(lut), lut,
    0, NULL, NULL);

// Download
clEnqueueReadBuffer(queue, gpu_output, CL_TRUE,
    0, output_size, cpu_buffer,
    0, NULL, NULL);
```

### Metal: newBufferWithBytes

```objc
id<MTLDevice> device = (id<MTLDevice>)device_info.devicePV;

// Upload parameter data
id<MTLBuffer> paramBuf = [device newBufferWithBytes:&params
                                   length:sizeof(params)
                                   options:MTLResourceStorageModeManaged];

// For larger data (LUTs, textures):
id<MTLBuffer> lutBuf = [device newBufferWithBytes:lutData
                                 length:lutSize
                                 options:MTLResourceStorageModeShared];
// StorageModeShared: accessible by both CPU and GPU (Apple Silicon)
// StorageModeManaged: requires explicit synchronization (Intel Macs)
```

---

## Rowbytes Handling During Transfer

When copying image data between CPU and GPU, you must account for row stride (rowbytes) differences. GPU buffers and CPU buffers may have different padding.

### Pitched Copy with CUDA

```cpp
// 2D copy respecting different source and destination pitches
cudaMemcpy2D(
    dst_ptr,                    // destination pointer
    dst_rowbytes,               // destination pitch in bytes
    src_ptr,                    // source pointer
    src_rowbytes,               // source pitch in bytes
    width * sizeof(float4),     // width of data to copy in bytes
    height,                     // number of rows
    cudaMemcpyHostToDevice);    // direction
```

### Row-by-Row Copy (Portable)

If the source and destination have different row strides:

```cpp
// Upload CPU image to GPU, row by row
char *cpu_row = (char*)cpu_data;
for (int y = 0; y < height; y++) {
    size_t offset = y * gpu_rowbytes;  // offset into GPU buffer
    cudaMemcpy(
        (char*)gpu_ptr + offset,
        cpu_row,
        width * sizeof(float4),         // only copy valid pixels, not padding
        cudaMemcpyHostToDevice);
    cpu_row += cpu_rowbytes;
}
```

> **Pitfall**: Copying `width * sizeof(float4)` bytes per row is correct. Copying `rowbytes` per row would also transfer padding bytes, which is safe but wastes bandwidth. Never copy `width * sizeof(float4)` bytes assuming it covers the full stride -- you might underrun or overrun the source.

### The Rowbytes Invariant

```
rowbytes >= width * bytes_per_pixel

For PF_PixelFormat_GPU_BGRA128:
    bytes_per_pixel = 16  (4 floats * 4 bytes)
    rowbytes >= width * 16
```

Always use `rowbytes` for row advancement, `width * bytes_per_pixel` for the valid data portion within each row.

---

## HOST_VISIBLE vs DEVICE_LOCAL Memory

These Vulkan terms have direct analogs in AE's memory model:

| Concept | AE Suite Function | CUDA Equivalent | Characteristics |
|---------|------------------|-----------------|-----------------|
| DEVICE_LOCAL | `AllocateDeviceMemory` | `cuMemAlloc` | Fast GPU access, no CPU access |
| HOST_VISIBLE (pinned) | `AllocateHostMemory` | `cuMemHostAlloc` | CPU-accessible, DMA-capable for fast transfers |
| HOST_VISIBLE + HOST_COHERENT | (not directly exposed) | `cuMemAllocManaged` | Unified memory, not used by AE |

### When to Use Each

- **Device memory** (`AllocateDeviceMemory`): For intermediate GPU buffers, LUTs uploaded once, temporary render targets. This is the fast path for GPU compute.

- **Host/pinned memory** (`AllocateHostMemory`): For staging buffers when you need to transfer data from CPU to GPU. Pinned memory avoids an extra copy that pageable malloc would require.

- **GPU worlds** (`CreateGPUWorld`): For intermediate image buffers with proper width/height/rowbytes metadata. Preferred over raw `AllocateDeviceMemory` for image data because AE tracks the pixel format and dimensions.

### Memory Budget and Purging

AE tracks GPU memory usage across all plugins. When allocation fails, try purging first:

```cpp
void *mem = nullptr;
PF_Err err = gpuSuite->AllocateDeviceMemory(
    in_dataP->effect_ref, device_index, needed_bytes, &mem);

if (err != PF_Err_NONE || !mem) {
    // Try to free some AE cache memory
    size_t purged = 0;
    gpuSuite->PurgeDeviceMemory(
        in_dataP->effect_ref, device_index, needed_bytes, &purged);

    // Retry allocation
    err = gpuSuite->AllocateDeviceMemory(
        in_dataP->effect_ref, device_index, needed_bytes, &mem);

    if (err != PF_Err_NONE || !mem) {
        return PF_Err_OUT_OF_MEMORY;
    }
}
```

> **Warning**: `PurgeDeviceMemory` should only be used in emergency situations. It forces AE to discard cached GPU data, which can severely impact performance if other effects need to re-render.

---

## GPU Data Flow Diagram

```
PF_Cmd_SMART_PRE_RENDER
    |
    +-- Set PF_RenderOutputFlag_GPU_RENDER_POSSIBLE
    +-- Pass pre_render_data (CPU-side params)
    |
    v
PF_Cmd_SMART_RENDER_GPU
    |
    +-- checkout_layer_pixels -> input PF_EffectWorld
    |     data = NULL (GPU mode)
    |     width, height, rowbytes = valid
    |
    +-- GetGPUWorldData(input_world) -> void* src_gpu
    |     CUDA: CUdeviceptr  |  OpenCL: cl_mem  |  Metal: id<MTLBuffer>
    |
    +-- checkout_output -> output PF_EffectWorld
    +-- GetGPUWorldData(output_world) -> void* dst_gpu
    |
    +-- (optional) CreateGPUWorld -> intermediate PF_EffectWorld
    +-- GetGPUWorldData(intermediate) -> void* tmp_gpu
    |
    +-- Dispatch kernels: src_gpu -> [process] -> dst_gpu
    |
    +-- (optional) DisposeGPUWorld(intermediate)
    |
    +-- Return PF_Err_NONE
```

---

## Framework-Specific Memory Details

### CUDA Memory Model

```
+-------------------+      cudaMemcpyHtoD      +-------------------+
|   Host (CPU)      | -----------------------> |   Device (GPU)    |
|   Pageable RAM    |                          |   VRAM            |
+-------------------+                          +-------------------+
        |                                              ^
        v                                              |
+-------------------+      DMA Transfer         +-------------------+
|   Pinned Memory   | -----------------------> |   Device Memory   |
|   (Host Alloc)    |      (faster)            |   (Device Alloc)  |
+-------------------+                          +-------------------+
```

- `AllocateDeviceMemory` -> `cuMemAlloc` -> VRAM allocation
- `AllocateHostMemory` -> `cuMemHostAlloc` -> Page-locked system RAM
- AE's CUDA stream (`command_queuePV`) should be used for async operations

### OpenCL Memory Model

OpenCL buffers (`cl_mem`) can be created with various flags:

| Flag | Behavior |
|------|----------|
| `CL_MEM_READ_ONLY` | Kernel can only read; driver may optimize placement |
| `CL_MEM_WRITE_ONLY` | Kernel can only write |
| `CL_MEM_READ_WRITE` | Default; kernel can read and write |
| `CL_MEM_COPY_HOST_PTR` | Initialize buffer contents from host pointer at creation |
| `CL_MEM_USE_HOST_PTR` | Use the provided host pointer directly (zero-copy on some hardware) |

AE's `AllocateDeviceMemory` for OpenCL returns the result of `clCreateBuffer`.

### Metal Memory Model

| Storage Mode | CPU Access | GPU Access | Use Case |
|-------------|-----------|-----------|----------|
| `MTLStorageModeShared` | Yes | Yes | Apple Silicon unified memory; parameter buffers |
| `MTLStorageModeManaged` | Yes (synchronized) | Yes | Intel Macs; requires explicit sync |
| `MTLStorageModePrivate` | No | Yes | GPU-only data; fastest GPU access |

AE creates GPU worlds with Private storage. Parameter buffers should use Shared or Managed.

---

## Common Pitfalls

1. **Using `cudaMalloc` directly** -- Bypasses AE's memory tracking. Use `AllocateDeviceMemory` instead. AE cannot purge memory it does not know about, leading to allocation failures.

2. **Forgetting to dispose GPU worlds** -- `CreateGPUWorld` allocations persist until explicitly freed with `DisposeGPUWorld`. Forgetting this leaks GPU memory across render calls.

3. **Assuming data pointer is valid in GPU mode** -- `PF_EffectWorld.data` is NULL when rendering on GPU. Always use `GetGPUWorldData`.

4. **Ignoring rowbytes during transfers** -- Source and destination may have different row strides. Use pitched copies (`cudaMemcpy2D`) or row-by-row loops.

5. **Mixing allocation sources** -- Memory allocated with `AllocateDeviceMemory` must be freed with `FreeDeviceMemory`, not `cudaFree`. Memory from `AllocateHostMemory` must be freed with `FreeHostMemory`, not `free`.

6. **Purging too aggressively** -- `PurgeDeviceMemory` and `PurgeHostMemory` are emergency functions. Calling them routinely degrades AE's caching performance.

7. **Not accounting for bytes_per_pixel** -- GPU format `PF_PixelFormat_GPU_BGRA128` is 16 bytes per pixel (4 channels * 4 bytes). Pitch in elements = `rowbytes / 16`. Getting this wrong causes diagonal shearing.

8. **Transferring more data than exists** -- When downloading from GPU, copy `width * sizeof(float4)` per row, not `rowbytes`. The padding region may contain uninitialized data.

---

## See Also

- [CUDA Kernel Patterns](cuda-kernels.md) -- Kernel dispatch, pitch handling, error checking
- [Metal Compute Shaders](metal-compute.md) -- Metal buffer management and dispatch
- [OpenCL Setup](opencl-setup.md) -- OpenCL buffer creation and kernel arguments
- [CPU/GPU Render Parity](cpu-gpu-parity.md) -- Pixel format differences between CPU and GPU paths
