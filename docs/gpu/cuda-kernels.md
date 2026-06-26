# CUDA Kernel Patterns for AE GPU Effects

> How After Effects dispatches GPU work to CUDA kernels, including kernel launch patterns, thread indexing, memory layout, pitch handling, and error checking.

## Overview

When After Effects renders on an NVIDIA GPU, it offers the `PF_GPU_Framework_CUDA` framework to plugins. AE manages the CUDA context and stream; your plugin receives device pointers to GPU buffers and dispatches CUDA kernels against them. The pixel format on the GPU side is always `PF_PixelFormat_GPU_BGRA128` -- 32-bit float per channel, **BGRA** order.

This document covers practical CUDA kernel patterns derived from the official SDK sample (`SDK_Invert_ProcAmp`) and production plugin experience.

---

## Lifecycle: From SmartRenderGPU to Kernel Dispatch

The flow for a CUDA-based GPU effect is:

1. **`PF_Cmd_GLOBAL_SETUP`** -- set `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32`
2. **`PF_Cmd_GPU_DEVICE_SETUP`** -- check `extraP->input->what_gpu == PF_GPU_Framework_CUDA`. For CUDA, the kernel is statically linked (compiled by nvcc), so there is no runtime compilation step. Set `out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` to accept.
3. **`PF_Cmd_SMART_PRE_RENDER`** -- set `PF_RenderOutputFlag_GPU_RENDER_POSSIBLE` on `extraP->output->flags`
4. **`PF_Cmd_SMART_RENDER_GPU`** -- retrieve GPU buffer pointers via `PF_GPUDeviceSuite1::GetGPUWorldData`, then call your CUDA dispatch function

```
  EffectMain
     |
     +-- PF_Cmd_GPU_DEVICE_SETUP    --> Accept CUDA framework
     +-- PF_Cmd_SMART_PRE_RENDER    --> Set GPU_RENDER_POSSIBLE flag
     +-- PF_Cmd_SMART_RENDER_GPU    --> Get GPU pointers, launch kernels
     +-- PF_Cmd_GPU_DEVICE_SETDOWN  --> (CUDA: nothing to release)
```

### GPU Device Setup for CUDA

CUDA kernels are compiled at build time by nvcc and statically linked into the plugin binary. There is no runtime compilation. The device setup handler simply signals acceptance:

```cpp
static PF_Err GPUDeviceSetup(
    PF_InData *in_dataP,
    PF_OutData *out_dataP,
    PF_GPUDeviceSetupExtra *extraP)
{
    PF_Err err = PF_Err_NONE;

    if (extraP->input->what_gpu == PF_GPU_Framework_CUDA) {
        // CUDA kernels are statically linked -- nothing to compile
        out_dataP->out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;
    }

    return err;
}
```

For OpenCL or Metal, you would compile shaders here and store pipeline state in `extraP->output->gpu_data`. CUDA does not need this.

---

## Getting GPU Pointers

In `SmartRenderGPU`, use `PF_GPUDeviceSuite1` to obtain raw device pointers:

```cpp
AEFX_SuiteScoper<PF_GPUDeviceSuite1> gpu_suite =
    AEFX_SuiteScoper<PF_GPUDeviceSuite1>(in_dataP,
        kPFGPUDeviceSuite, kPFGPUDeviceSuiteVersion1, out_dataP);

void *src_gpu = nullptr;
void *dst_gpu = nullptr;

ERR(gpu_suite->GetGPUWorldData(in_dataP->effect_ref, input_worldP, &src_gpu));
ERR(gpu_suite->GetGPUWorldData(in_dataP->effect_ref, output_worldP, &dst_gpu));
```

**Critical**: these are `CUdeviceptr` values cast to `void*`. They point to GPU memory allocated by AE via `cuMemAlloc`. You can cast them directly to `float*` or `float4*` for your kernel calls.

### Creating Intermediate GPU Buffers

If your effect requires multi-pass rendering, allocate intermediate buffers through the suite:

```cpp
PF_EffectWorld *temp_buffer = nullptr;
ERR(gpu_suite->CreateGPUWorld(
    in_dataP->effect_ref,
    extraP->input->device_index,
    input_worldP->width,
    input_worldP->height,
    input_worldP->pix_aspect_ratio,
    in_dataP->field,
    PF_PixelFormat_GPU_BGRA128,  // only GPU format available
    false,                        // clear_pixB
    &temp_buffer));

// ... use temp_buffer in kernel dispatches ...

// MUST free when done
ERR(gpu_suite->DisposeGPUWorld(in_dataP->effect_ref, temp_buffer));
```

> **Warning**: Always dispose intermediate GPU worlds before returning from `SmartRenderGPU`. AE does not track plugin-allocated GPU worlds for automatic cleanup.

---

## Kernel Architecture

### Basic Kernel Structure

A CUDA kernel for AE image processing follows a standard 2D grid pattern:

```cuda
__global__ void MyEffectKernel(
    float4 const *src,
    float4       *dst,
    int           srcPitch,    // in float4 elements, NOT bytes
    int           dstPitch,    // in float4 elements, NOT bytes
    unsigned int  width,
    unsigned int  height,
    float         param1,
    float         param2)
{
    unsigned int x = blockIdx.x * blockDim.x + threadIdx.x;
    unsigned int y = blockIdx.y * blockDim.y + threadIdx.y;

    if (x >= width || y >= height)
        return;

    // Read pixel using pitch (not width!)
    float4 pixel = src[y * srcPitch + x];

    // pixel.x = Blue
    // pixel.y = Green
    // pixel.z = Red
    // pixel.w = Alpha

    // Process...
    pixel.x = 1.0f - pixel.x;  // Invert blue
    pixel.y = 1.0f - pixel.y;  // Invert green
    pixel.z = 1.0f - pixel.z;  // Invert red
    // Leave alpha unchanged

    dst[y * dstPitch + x] = pixel;
}
```

### Channel Order: BGRA

| `float4` field | Channel | Notes |
|----------------|---------|-------|
| `.x` | Blue | |
| `.y` | Green | |
| `.z` | Red | |
| `.w` | Alpha | |

This is `PF_PixelFormat_GPU_BGRA128`. The CPU path uses `PF_PixelFloat` which is ARGB (`.alpha`, `.red`, `.green`, `.blue`). **This is the #1 source of GPU color bugs.** Any effect that treats R/G/B differently (color correction, channel operations, film grain) will produce swapped red/blue if the kernel assumes RGBA.

---

## Pitch vs Width: The Critical Distinction

### What "Pitch" Means in AE GPU Buffers

AE GPU buffers may have padding at the end of each row for alignment. The `rowbytes` field on `PF_EffectWorld` gives the stride in **bytes**. Your kernel needs the stride in **float4 elements**:

```cpp
A_long src_row_bytes = input_worldP->rowbytes;
A_long dst_row_bytes = output_worldP->rowbytes;

int bytes_per_pixel = 16;  // sizeof(float4) = 4 channels * 4 bytes

int srcPitch = src_row_bytes / bytes_per_pixel;  // stride in float4 elements
int dstPitch = dst_row_bytes / bytes_per_pixel;
```

### Why This Matters

```
Row 0: [pixel_0][pixel_1]...[pixel_W-1][padding...]
Row 1: [pixel_0][pixel_1]...[pixel_W-1][padding...]
```

- `width` = number of valid pixels per row
- `pitch` = distance between rows in float4 elements (width + padding)
- **pitch >= width**, always

If you index as `src[y * width + x]` instead of `src[y * srcPitch + x]`, you will read from the wrong row. The effect will appear to work for small images but produce diagonal shearing artifacts on larger ones.

> **Pitfall**: Source and destination buffers can have **different** pitches. Never assume `srcPitch == dstPitch`. The SDK sample uses separate pitch values for each buffer.

---

## Kernel Launch Patterns

### Standard 2D Grid Launch

The SDK sample uses 16x16 thread blocks:

```cpp
void MyEffect_CUDA(
    float const *src,
    float *dst,
    unsigned int srcPitch,
    unsigned int dstPitch,
    unsigned int width,
    unsigned int height,
    float param1)
{
    dim3 blockDim(16, 16, 1);
    dim3 gridDim(
        (width  + blockDim.x - 1) / blockDim.x,
        (height + blockDim.y - 1) / blockDim.y,
        1);

    MyEffectKernel<<<gridDim, blockDim, 0>>>(
        (float4 const*)src,
        (float4*)dst,
        srcPitch,
        dstPitch,
        width,
        height,
        param1);

    cudaDeviceSynchronize();
}
```

### Thread Block Size Considerations

| Block Size | Threads | Notes |
|-----------|---------|-------|
| 16 x 16 | 256 | SDK default. Good general choice. |
| 32 x 8 | 256 | Better for row-major access patterns (horizontal blur). |
| 8 x 32 | 256 | Better for column-major access patterns (vertical blur). |
| 32 x 32 | 1024 | Maximum for most GPUs. Higher occupancy but more register pressure. |

For simple per-pixel effects, 16x16 is fine. For effects that read neighboring pixels (blur, edge detection), align the block shape with the access pattern to improve cache coherence.

### Calling Convention

The dispatch function is declared `extern "C"` in the `.cpp` file and defined in the `.cu` file:

```cpp
// In your .cpp file:
extern void MyEffect_CUDA(
    float const *src,
    float *dst,
    unsigned int srcPitch,
    unsigned int dstPitch,
    unsigned int width,
    unsigned int height,
    float param1);
```

The `.cu` file is compiled by nvcc, which handles the `<<<>>>` kernel launch syntax. The `.cpp` file is compiled by MSVC/Clang normally and calls the dispatch function as a regular C function.

---

## Passing Parameters to Kernels

### Scalar Parameters (Simple Approach)

For a small number of effect parameters, pass them as kernel arguments directly:

```cuda
__global__ void ColorCorrectionKernel(
    float4 const *src,
    float4       *dst,
    int           srcPitch,
    int           dstPitch,
    unsigned int  width,
    unsigned int  height,
    float         brightness,
    float         contrast,
    float         saturation)
{
    // ...
}
```

### Struct Parameters (Many Parameters)

When you have many parameters, use a struct. Define it in a shared header included by both `.cpp` and `.cu`:

```cpp
// SharedTypes.h -- included by both .cpp and .cu
typedef struct {
    int   mSrcPitch;
    int   mDstPitch;
    int   mWidth;
    int   mHeight;
    float mBrightness;
    float mContrast;
    float mHueCosSaturation;
    float mHueSinSaturation;
} EffectParams;
```

Pass by value (for small structs) or by pointer (for large structs via constant memory):

```cuda
// By value -- for structs up to ~256 bytes
__global__ void MyKernel(float4 const *src, float4 *dst, EffectParams params)
{
    // Access as params.mWidth, params.mBrightness, etc.
}

// Via constant memory -- for larger or frequently reused structs
__constant__ EffectParams d_params;

__global__ void MyKernel(float4 const *src, float4 *dst)
{
    // Access as d_params.mWidth, d_params.mBrightness, etc.
}

// Host side:
cudaMemcpyToSymbol(d_params, &hostParams, sizeof(EffectParams));
MyKernel<<<grid, block>>>(src, dst);
```

---

## Shared Memory Usage

Shared memory is valuable for kernels that access neighboring pixels (convolution, blur, morphological operations).

### Tiled Convolution Example

```cuda
__global__ void BoxBlurKernel(
    float4 const *src,
    float4       *dst,
    int           srcPitch,
    int           dstPitch,
    unsigned int  width,
    unsigned int  height,
    int           radius)
{
    // Tile dimensions with halo for the blur radius
    extern __shared__ float4 tile[];

    int tx = threadIdx.x;
    int ty = threadIdx.y;
    int bw = blockDim.x;
    int bh = blockDim.y;

    // Shared memory tile width includes halo on both sides
    int tile_w = bw + 2 * radius;

    int gx = blockIdx.x * blockDim.x + threadIdx.x;
    int gy = blockIdx.y * blockDim.y + threadIdx.y;

    // Load center region
    int sx = tx + radius;
    int sy = ty + radius;
    int cx = min(max(gx, 0), (int)width - 1);
    int cy = min(max(gy, 0), (int)height - 1);
    tile[sy * tile_w + sx] = src[cy * srcPitch + cx];

    // Load halo regions (left, right, top, bottom)
    if (tx < radius) {
        int hx = min(max(gx - radius, 0), (int)width - 1);
        tile[sy * tile_w + tx] = src[cy * srcPitch + hx];

        hx = min(gx + bw, (int)width - 1);
        tile[sy * tile_w + sx + bw] = src[cy * srcPitch + hx];
    }
    if (ty < radius) {
        int hy = min(max(gy - radius, 0), (int)height - 1);
        tile[ty * tile_w + sx] = src[hy * srcPitch + cx];

        hy = min(gy + bh, (int)height - 1);
        tile[(sy + bh) * tile_w + sx] = src[hy * srcPitch + cx];
    }

    __syncthreads();

    if (gx >= width || gy >= height) return;

    // Compute box blur from shared memory
    float4 sum = make_float4(0, 0, 0, 0);
    int count = 0;
    for (int dy = -radius; dy <= radius; dy++) {
        for (int dx = -radius; dx <= radius; dx++) {
            float4 p = tile[(sy + dy) * tile_w + (sx + dx)];
            sum.x += p.x;
            sum.y += p.y;
            sum.z += p.z;
            sum.w += p.w;
            count++;
        }
    }
    float inv = 1.0f / (float)count;
    dst[gy * dstPitch + gx] = make_float4(
        sum.x * inv, sum.y * inv, sum.z * inv, sum.w * inv);
}
```

Launch with dynamic shared memory:

```cpp
int tile_w = 16 + 2 * radius;
int tile_h = 16 + 2 * radius;
size_t shared_bytes = tile_w * tile_h * sizeof(float4);

BoxBlurKernel<<<gridDim, blockDim, shared_bytes>>>(
    src, dst, srcPitch, dstPitch, width, height, radius);
```

> **Pitfall**: Shared memory is limited (typically 48KB per block, configurable up to 96KB or 164KB on newer architectures). For large blur radii, switch to a separable two-pass approach (horizontal then vertical) to keep the tile size manageable.

---

## Error Checking

### During SmartRenderGPU

After dispatching kernels, check for errors before returning to AE:

```cpp
// Option 1: cudaPeekAtLastError (checks launch errors without resetting)
if (cudaPeekAtLastError() != cudaSuccess) {
    err = PF_Err_INTERNAL_STRUCT_DAMAGED;
}

// Option 2: cudaDeviceSynchronize + cudaGetLastError (catches async errors too)
cudaDeviceSynchronize();
cudaError_t cuda_err = cudaGetLastError();
if (cuda_err != cudaSuccess) {
    // For debugging: cudaGetErrorString(cuda_err)
    err = PF_Err_INTERNAL_STRUCT_DAMAGED;
}
```

The SDK sample calls `cudaDeviceSynchronize()` inside each dispatch function, then checks with `cudaPeekAtLastError()` afterward. This is the safest pattern.

### Common CUDA Errors in AE Plugins

| Error | Likely Cause |
|-------|-------------|
| `cudaErrorLaunchOutOfResources` | Block size too large, too many registers per thread |
| `cudaErrorIllegalAddress` | Accessing memory outside allocated buffer (wrong pitch calculation) |
| `cudaErrorLaunchTimeout` | Kernel taking too long (Windows TDR, typically 2 seconds) |
| `cudaErrorInvalidDevice` | CUDA context not current on this thread |
| `cudaErrorMemoryAllocation` | GPU out of memory; try `PurgeDeviceMemory` before allocating |

### TDR (Timeout Detection and Recovery) on Windows

Windows enforces a 2-second GPU execution timeout by default. For complex effects on large frames (4K+), a single kernel launch can exceed this. Solutions:

- Break processing into multiple smaller kernel launches
- Process the image in horizontal strips
- Reduce thread block sizes to allow more time-slicing

---

## Multi-Pass Kernel Patterns

Many real effects require multiple passes. Use `CreateGPUWorld` for intermediate buffers:

```cpp
static PF_Err SmartRenderGPU(/* ... */)
{
    // Get input/output GPU pointers
    void *src_gpu, *dst_gpu, *tmp_gpu;
    gpu_suite->GetGPUWorldData(in_dataP->effect_ref, input_worldP, &src_gpu);
    gpu_suite->GetGPUWorldData(in_dataP->effect_ref, output_worldP, &dst_gpu);

    // Allocate intermediate buffer
    PF_EffectWorld *temp_world;
    gpu_suite->CreateGPUWorld(in_dataP->effect_ref,
        extraP->input->device_index,
        input_worldP->width, input_worldP->height,
        input_worldP->pix_aspect_ratio, in_dataP->field,
        PF_PixelFormat_GPU_BGRA128, false, &temp_world);

    gpu_suite->GetGPUWorldData(in_dataP->effect_ref, temp_world, &tmp_gpu);

    A_long tmp_pitch = temp_world->rowbytes / 16;

    // Pass 1: src -> temp (horizontal blur)
    HorizontalBlur_CUDA((float*)src_gpu, (float*)tmp_gpu,
        src_pitch, tmp_pitch, width, height, radius);
    cudaDeviceSynchronize();

    // Pass 2: temp -> dst (vertical blur)
    VerticalBlur_CUDA((float*)tmp_gpu, (float*)dst_gpu,
        tmp_pitch, dst_pitch, width, height, radius);
    cudaDeviceSynchronize();

    // MUST dispose intermediate buffer
    gpu_suite->DisposeGPUWorld(in_dataP->effect_ref, temp_world);

    if (cudaPeekAtLastError() != cudaSuccess) {
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }
    return PF_Err_NONE;
}
```

---

## The SDK's Cross-Platform Kernel Macros

The official `SDK_Invert_ProcAmp` sample uses cross-platform kernel macros from `PrGPU/KernelSupport/KernelCore.h`:

```cuda
GF_KERNEL_FUNCTION(InvertColorKernel,
    ((GF_PTR_READ_ONLY(float4))(inSrc))
    ((GF_PTR(float4))(outDst)),
    ((int)(inSrcPitch))
    ((int)(inDstPitch))
    ((int)(in16f))
    ((unsigned int)(inWidth))
    ((unsigned int)(inHeight)),
    ((uint2)(inXY)(KERNEL_XY)))
{
    if (inXY.x < inWidth && inXY.y < inHeight)
    {
        float4 pixel = ReadFloat4(inSrc, inXY.y * inSrcPitch + inXY.x, !!in16f);
        // ... process ...
        WriteFloat4(pixel, outDst, inXY.y * inDstPitch + inXY.x, !!in16f);
    }
}
```

These macros expand differently for CUDA, OpenCL, Metal, and HLSL, allowing a single kernel source to compile for all backends. They are provided in the SDK under `GPUUtils/PrGPU/KernelSupport/`.

| Macro | Purpose |
|-------|---------|
| `GF_KERNEL_FUNCTION` | Declares a kernel with named parameters |
| `GF_PTR_READ_ONLY(type)` | Read-only buffer parameter |
| `GF_PTR(type)` | Read-write buffer parameter |
| `KERNEL_XY` | Built-in 2D thread coordinate |
| `ReadFloat4` / `WriteFloat4` | Pitch-aware pixel access with 16f support |
| `GF_DEVICE_TARGET_DEVICE` | Preprocessor guard: true when compiling for GPU |

> **Note**: These macros originate from Premiere Pro's GPU filter architecture. They work in AE but are optional. Many production plugins write native CUDA kernels directly.

---

## Build Configuration

### Visual Studio / CUDA Toolkit Integration

Your `.vcxproj` must:

1. Include the CUDA build customization (`.targets` / `.props` from the CUDA Toolkit)
2. Set the `.cu` file's item type to "CUDA C/C++"
3. Link against `cudart_static.lib`

The SDK defines `PF_CUDAVersion 11080` (CUDA 11.8) in `AE_EffectGPUSuites.h`. Your plugin should target this version or newer.

### CMake Example

```cmake
enable_language(CUDA)
set(CMAKE_CUDA_STANDARD 17)

add_library(MyEffect SHARED
    MyEffect.cpp
    MyEffect_Kernel.cu
)

target_compile_options(MyEffect PRIVATE
    $<$<COMPILE_LANGUAGE:CUDA>:
        --use_fast_math
        -arch=sm_50        # Minimum compute capability
        -gencode=arch=compute_50,code=sm_50
        -gencode=arch=compute_75,code=sm_75
        -gencode=arch=compute_86,code=sm_86
    >
)

target_link_libraries(MyEffect PRIVATE
    CUDA::cudart_static
)
```

### Compute Capability Targets

| Architecture | Compute Capability | GPUs |
|-------------|-------------------|------|
| Maxwell | sm_50 | GTX 900 series |
| Pascal | sm_60 / sm_61 | GTX 1000 series |
| Turing | sm_75 | RTX 2000 series |
| Ampere | sm_86 | RTX 3000 series |
| Ada Lovelace | sm_89 | RTX 4000 series |

Target `sm_50` as the minimum for broad compatibility. Include `compute_XX` (PTX) targets for forward compatibility with future architectures.

---

## Common Pitfalls

1. **Pitch vs width confusion** -- Always divide `rowbytes` by `sizeof(float4)` to get the pitch in elements. Never use width as the row stride.

2. **BGRA channel order** -- GPU buffers use BGRA (`.x=B, .y=G, .z=R, .w=A`). CPU buffers use ARGB. Any per-channel operation must account for this.

3. **Forgetting bounds checks** -- The grid is rounded up to block boundaries. Threads beyond `(width, height)` must early-return to avoid out-of-bounds writes.

4. **Not synchronizing** -- Call `cudaDeviceSynchronize()` after kernel dispatches and before AE reads the output buffer. The SDK sample synchronizes inside each dispatch function.

5. **Not checking errors** -- Always check `cudaPeekAtLastError()` or `cudaGetLastError()` before returning to AE. Silent CUDA errors will produce garbage output or crash AE later.

6. **Allocating GPU memory directly** -- Use `PF_GPUDeviceSuite1::AllocateDeviceMemory` or `CreateGPUWorld`, not `cudaMalloc`. AE tracks GPU memory budgets and can purge caches when needed. Direct allocation bypasses this accounting.

7. **Windows TDR timeout** -- Keep individual kernel launches under ~1.5 seconds on Windows. Split large operations into multiple launches.

8. **Missing `cudaDeviceSynchronize` between passes** -- If pass 2 reads from pass 1's output, ensure pass 1 has completed. In practice, kernel launches on the same stream are serialized, so this is only necessary if you use multiple streams.

---

## See Also

- [GPU Memory Management](gpu-memory.md) -- CPU/GPU transfer patterns, `PF_GPUDeviceSuite1` reference
- [CPU/GPU Render Parity](cpu-gpu-parity.md) -- Ensuring matching results between CPU and GPU paths
- [OpenCL Setup](opencl-setup.md) -- OpenCL equivalent patterns for AMD/Intel GPUs
- [Metal Compute Shaders](metal-compute.md) -- Metal equivalent for macOS
