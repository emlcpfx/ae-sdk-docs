# AE GPU Framework Selection: CUDA vs OpenCL

## Current Implementation (December 2024)

MyGPUPlugin now supports **both CUDA and OpenCL** for AE native GPU rendering:
- **NVIDIA GPUs**: Uses CUDA (AE's preferred framework on NVIDIA)
- **AMD/Intel GPUs**: Uses OpenCL

```
[MyGPUPlugin AE-GPU] GPUDeviceSetup called - AE offering framework: CUDA
[MyGPUPlugin AE-GPU]   device_info.device_framework = 3
[MyGPUPlugin AE-GPU] Setting up CUDA (statically linked kernels)...
[MyGPUPlugin AE-GPU] CUDA setup complete - kernels are statically linked
```

## Background: The Original Problem

When we initially implemented only OpenCL, we discovered that on Windows with NVIDIA GPUs, **AE only offers CUDA** - it does NOT fall back to OpenCL. This document explains the GPU framework architecture and our solution.

## How AE GPU Framework Selection Works

### The Call Flow

1. AE calls `PF_Cmd_GPU_DEVICE_SETUP` for each GPU framework it wants to query
2. `extra->input->what_gpu` tells you which framework AE is asking about
3. `PF_GPUDeviceInfo` contains framework-specific pointers:
   - `contextPV` - CUDA context (`CUcontext`) or OpenCL context (`cl_context`)
   - `devicePV` - CUDA device (`CUdevice`) or OpenCL device (`cl_device_id`)
   - `command_queuePV` - CUDA stream (`CUstream`) or OpenCL queue (`cl_command_queue`)
4. You set `out_data->out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` ONLY if you accept that framework

### Available Frameworks (Windows)

```c
PF_GPU_Framework_NONE     = 0
PF_GPU_Framework_OPENCL   = 1
PF_GPU_Framework_METAL    = 2  // macOS only
PF_GPU_Framework_CUDA     = 3
PF_GPU_Framework_DIRECTX  = 4
```

### Framework Priority on NVIDIA GPUs

On systems with NVIDIA GPUs, AE typically offers CUDA first because:
- NVIDIA's CUDA drivers are well-optimized
- AE's internal GPU effects use CUDA on NVIDIA hardware
- CUDA has better debugging tools

**AE may only call GPUDeviceSetup once with CUDA and never offer OpenCL** if you don't have specific OpenCL drivers installed or configured.

## Why You Can't Mix Frameworks

### The Crash

If you try to use OpenCL APIs with a CUDA context:

```cpp
// BAD: AE gave us a CUDA context, but we call OpenCL
cl_context context = (cl_context)device_info.contextPV;  // This is actually a CUcontext!
cl_program program = clCreateProgramWithSource(context, ...);  // CRASH!
```

This crashes with `INVALID_POINTER_READ` because:
1. CUDA and OpenCL contexts are completely different memory structures
2. The OpenCL ICD loader tries to dispatch through the context's vtable
3. The CUDA context has no valid OpenCL vtable -> invalid memory read

### Memory Space Incompatibility

Even if you created your own OpenCL context, you **cannot use AE's GPU worlds**:

```cpp
// AE's GPU worlds are in CUDA memory space
void* ae_gpu_buffer;
gpu_suite->GetGPUWorldData(effect_ref, world, &ae_gpu_buffer);
// ae_gpu_buffer is a CUdeviceptr (CUDA device pointer)

// Our OpenCL buffer is in a separate memory space
cl_mem our_buffer = clCreateBuffer(our_context, ...);
// These cannot directly reference each other!
```

To work around this, you'd need:
1. Download from CUDA -> CPU
2. Upload from CPU -> OpenCL
3. Process in OpenCL
4. Download from OpenCL -> CPU
5. Upload from CPU -> CUDA

This defeats the purpose of GPU rendering.

## Solutions

### Option 1: Only Accept OpenCL (Current Approach)

```cpp
PF_Err GPUDeviceSetup(PF_InData* in_data, PF_OutData* out_data,
                      PF_GPUDeviceSetupExtra* extra) {
    PF_GPUDeviceInfo device_info;
    gpu_suite->GetDeviceInfo(in_data->effect_ref, extra->input->device_index, &device_info);

    // Only accept if AE is offering OpenCL
    if (extra->input->what_gpu == PF_GPU_Framework_OPENCL &&
        device_info.device_framework == PF_GPU_Framework_OPENCL) {
        // Safe to use OpenCL APIs with device_info.contextPV
        // Set up kernels...
        out_data->out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;
    }
    // For CUDA/DirectX: don't set out_flags2 -> AE falls back to CPU

    return PF_Err_NONE;
}
```

**Pros:**
- Simple, safe
- Works on AMD/Intel GPUs where OpenCL is preferred

**Cons:**
- Falls back to CPU on NVIDIA if AE doesn't offer OpenCL
- May need user to configure AE preferences

### Option 2: Implement CUDA Support (IMPLEMENTED)

Write CUDA kernels in addition to OpenCL. This is what MyGPUPlugin now does.

**Key Implementation Details:**

1. **CUDA kernels are statically linked** (unlike OpenCL which compiles at runtime)
2. **GPUDeviceSetup for CUDA is simple** - just set the support flag
3. **Use `extern "C"` wrapper functions** to call CUDA kernels from C++

```cpp
// In GPUDeviceSetup:
if (extra->input->what_gpu == PF_GPU_Framework_CUDA &&
    device_info.device_framework == PF_GPU_Framework_CUDA) {
    // CUDA kernels are statically linked - no runtime setup needed
    out_data->out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;
    instance->m_activeFramework = PF_GPU_Framework_CUDA;
}
else if (extra->input->what_gpu == PF_GPU_Framework_OPENCL &&
         device_info.device_framework == PF_GPU_Framework_OPENCL) {
    // OpenCL needs runtime compilation
    cl_context context = (cl_context)device_info.contextPV;
    // Create program from source...
    out_data->out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;
    instance->m_activeFramework = PF_GPU_Framework_OPENCL;
}

// In SmartRenderGPU - dispatch based on active framework:
if (instance->m_activeFramework == PF_GPU_Framework_CUDA) {
    err = instance->RenderCUDA(...);
} else if (instance->m_activeFramework == PF_GPU_Framework_OPENCL) {
    err = instance->RenderOpenCL(...);
}
```

**CUDA Kernel File Structure (MyPlugin_Kernel.cu):**

```cuda
// Kernel implementation
__global__ void BlurHorizontalKernel_CUDA(...) { ... }

// Host wrapper with extern "C" linkage
extern "C" {
void MyPlugin_BlurHorizontal_CUDA(const float* src, float* dst, ...) {
    dim3 blockDim(16, 16, 1);
    dim3 gridDim(...);
    BlurHorizontalKernel_CUDA<<<gridDim, blockDim>>>(src, dst, ...);
}
}
```

**CMake Configuration:**

```cmake
project(MyGPUPlugin LANGUAGES CXX C CUDA)

# Add CUDA source
target_sources(MyGPUPlugin PRIVATE src/gpu/MyPlugin_Kernel.cu)

# Set architectures
set_target_properties(MyGPUPlugin PROPERTIES
    CUDA_ARCHITECTURES "75;86;89"
)

# Match runtime library with C++ (/MT for AE plugins)
target_compile_options(MyGPUPlugin PRIVATE
    $<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=/MT>
)
```

**Pros:**
- Works on all GPUs
- Best performance on NVIDIA
- No CPU<->GPU transfers during rendering

**Cons:**
- Must maintain two codebases (CUDA + OpenCL)
- CUDA requires NVIDIA toolkit for compilation
- More complex build system

### Option 3: Implement DirectX Support (Windows)

For Windows-only plugins, DirectX 12 compute shaders work on all GPUs:

```cpp
if (extra->input->what_gpu == PF_GPU_Framework_DIRECTX) {
    ID3D12Device* device = (ID3D12Device*)device_info.devicePV;
    ID3D12CommandQueue* queue = (ID3D12CommandQueue*)device_info.command_queuePV;
    // Load HLSL shaders...
    out_data->out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;
}
```

**Pros:**
- Single codebase for Windows
- Works on NVIDIA, AMD, Intel

**Cons:**
- DirectX 12 is complex
- Windows only

## Debugging GPU Framework Issues

### Use DebugView to See Framework Offers

Add logging to track what AE is offering:

```cpp
const char* framework_name = "UNKNOWN";
switch (extra->input->what_gpu) {
    case PF_GPU_Framework_NONE: framework_name = "NONE"; break;
    case PF_GPU_Framework_OPENCL: framework_name = "OPENCL"; break;
    case PF_GPU_Framework_CUDA: framework_name = "CUDA"; break;
    case PF_GPU_Framework_DIRECTX: framework_name = "DIRECTX"; break;
}
OutputDebugStringA(("[MyPlugin] GPUDeviceSetup - Framework: " +
                    std::string(framework_name)).c_str());
```

### Check Both Framework Indicators

Always verify both `extra->input->what_gpu` AND `device_info.device_framework`:

```cpp
// Defensive: check both match
if (extra->input->what_gpu == PF_GPU_Framework_OPENCL &&
    device_info.device_framework == PF_GPU_Framework_OPENCL) {
    // Safe to proceed with OpenCL
}
```

### Verify Context Pointer Before Use

```cpp
cl_context context = (cl_context)device_info.contextPV;
if (!context) {
    GPU_LOG("ERROR: Null context pointer");
    return PF_Err_NONE;  // Fall back to CPU
}
```

## Summary

| Scenario | Solution |
|----------|----------|
| Only have OpenCL kernels, NVIDIA GPU | Falls back to CPU |
| Only have OpenCL kernels, AMD/Intel GPU | GPU rendering works |
| **Have CUDA + OpenCL kernels** | **GPU works on all hardware** |
| Have DirectX + Metal kernels | GPU works cross-platform |

**Implementation:** We now have both CUDA and OpenCL kernels, providing native GPU acceleration on all hardware.

## Key Lessons Learned

1. **Never use OpenCL APIs with a non-OpenCL context** - The `device_info` pointers are framework-specific and cannot be interchanged.

2. **CUDA kernels are statically linked** - Unlike OpenCL which compiles at runtime, CUDA code is compiled at build time and linked into your plugin.

3. **Include order matters** - Include `cuda_runtime.h` BEFORE `CL/cl.h` in your .cpp files to avoid macro conflicts (OpenCL defines `constant` which conflicts with CUDA).

4. **Runtime library must match** - AE plugins require static runtime (`/MT`). Use `-Xcompiler=/MT` for CUDA to match your C++ code.

5. **Track active framework** - Store which framework was accepted in `GPUDeviceSetup` so `SmartRenderGPU` knows which renderer to call.

## Files

- `src/gpu/MyPlugin_Kernel.cu` - CUDA kernels (statically linked)
- `src/gpu/MyPlugin_Kernel.cl` - OpenCL kernels (runtime compiled)
- `src/gpu/AEGPURenderer.cpp` - Framework detection and rendering dispatch
- `src/gpu/AEGPURenderer.h` - CUDA function declarations with `extern "C"`

---

# Advanced GPU Implementation Notes

## Buffer Stride vs Width

**Critical Issue:** AE GPU buffers may have padding at the end of each row.

```cpp
// Get buffer size from AE
size_t buffer_size = 0;
gpuSuite->GetGPUWorldSize(in_data->effect_ref, worldP, &buffer_size);

// Calculate stride (in float4 units, not bytes)
A_u_long row_bytes = buffer_size / height;
int stride = row_bytes / sizeof(float) / 4;  // stride in pixels

// Example: width=3840, but stride=3842 (2 pixel padding per row)
```

**Always use stride, not width, for buffer indexing in kernels:**

```cuda
int idx = y * stride + x;  // Correct
int idx = y * width + x;   // WRONG - will cause horizontal stripe artifacts
```

## Input vs Output Buffer Differences

When using `PF_Cmd_SMART_PRE_RENDER` to expand output rect (for blur radius), the input and output buffers have **different sizes**:

```
PRERENDER: expansion=65
IN rect: (0,0,3840,2160)     -> Input: 3840x2160
OUT rect: (-65,-65,3905,2225) -> Output: 3970x2290
```

**Both buffers may also have different strides!**

```cpp
// Get sizes for BOTH buffers
size_t output_buffer_size, input_buffer_size;
gpuSuite->GetGPUWorldSize(effect_ref, output_worldP, &output_buffer_size);
gpuSuite->GetGPUWorldSize(effect_ref, input_worldP, &input_buffer_size);

// Calculate strides separately
int output_stride = (output_buffer_size / out_height) / sizeof(float) / 4;
int input_stride = (input_buffer_size / in_height) / sizeof(float) / 4;

// Calculate offset for centering input in output
int offset_x = (out_width - in_width) / 2;
int offset_y = (out_height - in_height) / 2;
```

## Copy Kernel with Offset

When input is smaller than output, you need a copy kernel that handles the offset:

```cuda
__global__ void CopyKernelWithOffset(
    const float4* __restrict__ src,
    float4* __restrict__ dst,
    int src_width,
    int src_height,
    int src_stride,
    int dst_stride,
    int offset_x,
    int offset_y)
{
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;

    if (x >= src_width || y >= src_height) return;

    int src_idx = y * src_stride + x;
    int dst_idx = (y + offset_y) * dst_stride + (x + offset_x);
    dst[dst_idx] = src[src_idx];
}
```

**Typical GPU pipeline with expansion:**
1. Clear output-sized temp buffer
2. Copy input to temp buffer at offset position
3. Process at output size
4. All temp buffers use output stride

## Alpha-Weighted Blur (Avoiding Black Edge Artifacts)

Standard box blur averages all pixels equally. When transparent (black) pixels are included, this causes **black edge artifacts**.

**Solution: Alpha-weighted blur**

```cuda
__global__ void BlurHKernel(
    const float4* __restrict__ src,
    float4* __restrict__ dst,
    int width, int height, int stride, int radius)
{
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;
    if (x >= width || y >= height) return;

    float4 sum = make_float4(0.0f, 0.0f, 0.0f, 0.0f);
    float alpha_sum = 0.0f;

    for (int kx = -radius; kx <= radius; kx++) {
        int sx = clamp(x + kx, 0, width - 1);
        float4 p = src[y * stride + sx];
        float alpha = p.w;

        // Weight RGB by alpha - transparent pixels don't contribute color
        sum.x += p.x * alpha;
        sum.y += p.y * alpha;
        sum.z += p.z * alpha;
        sum.w += alpha;
        alpha_sum += alpha;
    }

    float4 result;
    if (alpha_sum > 0.0001f) {
        float inv_alpha = 1.0f / alpha_sum;
        result.x = sum.x * inv_alpha;
        result.y = sum.y * inv_alpha;
        result.z = sum.z * inv_alpha;
        result.w = sum.w / (float)(2 * radius + 1);  // Alpha: normal average
    } else {
        result = make_float4(0.0f, 0.0f, 0.0f, 0.0f);
    }
    dst[y * stride + x] = result;
}
```

**Key insight:** RGB values are weighted by source alpha and then normalized by total alpha. Alpha itself uses normal averaging to spread properly.

## CUDA Stream from AE

AE provides a CUDA stream via `device_info.command_queuePV`:

```cpp
PF_GPUDeviceInfo device_info;
gpuSuite->GetDeviceInfo(in_data->effect_ref, extra->input->device_index, &device_info);
cudaStream_t cuda_stream = (cudaStream_t)device_info.command_queuePV;

// Use this stream for all kernel launches
MyKernel<<<gridDim, blockDim, 0, cuda_stream>>>(...);

// Synchronize before returning
cudaStreamSynchronize(cuda_stream);
```

## Temp Buffer Allocation

### Option 1: Raw CUDA Allocation (cudaMalloc)

For simple temp buffers that only need CUDA kernel access:

```cpp
size_t ae_buffer_size = (size_t)output_stride * out_height * sizeof(float) * 4;

void* temp1_gpu = nullptr;
cudaError_t err = cudaMalloc(&temp1_gpu, ae_buffer_size);

// ... use temp buffers ...

// Always cleanup
cudaFree(temp1_gpu);
```

### Option 2: SDK GPU World Allocation (CreateGPUWorld)

**CRITICAL: Use `CreateGPUWorld` instead of `PF_NewWorld` for GPU buffers!**

When you need to use SDK functions like `transfer_rect` with GPU buffers, you **MUST** use the GPU suite's `CreateGPUWorld` function, NOT `PF_NewWorld`. This is shown in `SDK_Invert_ProcAmp.cpp` line 854.

```cpp
// WRONG - PF_NewWorld creates CPU-side buffers, fails with "Unsupported PF_PixelFormat"
PF_EffectWorld cpu_world;
wsP->PF_NewWorld(in_data->effect_ref, width, height, TRUE, 0, &cpu_world);  // ERROR!

// CORRECT - CreateGPUWorld creates GPU-compatible buffers
PF_EffectWorld* gpu_worldP = nullptr;
err = gpuSuite->CreateGPUWorld(
    in_data->effect_ref,
    extra->input->device_index,    // GPU device index
    width, height,
    input_worldP->pix_aspect_ratio,
    in_data->field,
    PF_PixelFormat_GPU_BGRA128,    // GPU pixel format
    false,                          // clear_pixels
    &gpu_worldP);

// Get raw GPU pointer for CUDA kernels
void* gpu_data = nullptr;
gpuSuite->GetGPUWorldData(in_data->effect_ref, gpu_worldP, &gpu_data);

// Copy data from CUDA buffer to SDK GPU world
cudaMemcpyAsync(gpu_data, my_cuda_buffer, buffer_size,
                cudaMemcpyDeviceToDevice, cuda_stream);

// Now you can use SDK functions with this GPU world!
// For example, transfer_rect with PF_MaskWorld works on GPU buffers
wtransformSuite->transfer_rect(
    in_data->effect_ref,
    PF_Quality_HI,
    PF_MF_Alpha_PREMUL,
    PF_Field_FRAME,
    &gpu_worldP->extent_hint,
    gpu_worldP,
    &comp_mode,
    &mask_world,  // SDK masking works!
    0, 0,
    output_worldP);

// Cleanup with DisposeGPUWorld, NOT PF_DisposeWorld
gpuSuite->DisposeGPUWorld(in_data->effect_ref, gpu_worldP);
```

### When to Use Each Approach

| Scenario | Use |
|----------|-----|
| Simple temp buffer for CUDA kernels only | `cudaMalloc` |
| Need SDK functions (transfer_rect, fill_float, etc.) | `CreateGPUWorld` |
| Need to pass buffer to PF_MaskWorld | `CreateGPUWorld` |
| Intermediate processing, no SDK calls | `cudaMalloc` |

### Key Differences

| Feature | cudaMalloc | CreateGPUWorld |
|---------|------------|----------------|
| Allocation | Raw CUDA memory | SDK-managed GPU world |
| SDK compatible | No | Yes |
| transfer_rect support | No | Yes |
| PF_MaskWorld support | No | Yes |
| Cleanup | `cudaFree()` | `DisposeGPUWorld()` |
| Returns | `void*` | `PF_EffectWorld*` |

## Performance: GPU vs CPU

On a 4K frame (3840x2160), measured performance for edge extension effect:

| Platform | Time |
|----------|------|
| CPU (multi-threaded) | ~1800ms |
| GPU (CUDA) | ~50ms |

**~36x speedup** with GPU acceleration.

## Common Pitfalls

1. **Using width instead of stride** -> Horizontal stripe artifacts
2. **Not handling input/output size differences** -> Image offset/cropping
3. **Simple blur averaging transparent pixels** -> Black edge halos
4. **Forgetting to synchronize stream** -> Incomplete rendering
5. **Not clearing output buffer before partial copy** -> Garbage in expanded areas

## Debug Logging Pattern

```cpp
#define ENABLE_GPU_VERBOSE_DEBUG 0

void DebugLog(const char* msg) {
#ifdef _WIN32
    OutputDebugStringA(("[MYPLUGIN GPU] " + std::string(msg) + "\n").c_str());
#endif
}

void VerboseLog(const char* msg) {
#if ENABLE_GPU_VERBOSE_DEBUG
    DebugLog(msg);
#endif
}
```

Use `VerboseLog` for per-iteration messages, `DebugLog` for important status only.

---

# After Effects 2025 (CC 25.x) GPU Requirements

## Overview

After Effects 2025 introduced significant changes to GPU rendering requirements. Plugins that worked with GPU in AE 2024 may fall back to CPU rendering in AE 2025 without proper updates.

> **Note:** The specific AE version requirements and DirectX flag claims in this section are unverified and may not be accurate. Always check the latest SDK headers.

## Required Dependencies for AE 2025 GPU

### 1. CUDA 12.8+
AE 2025 (v25.4+) uses CUDA 12.8. While older CUDA versions may work, best compatibility requires:
- **CUDA Toolkit 12.8** or newer
- Download from: https://developer.nvidia.com/cuda-downloads

### 2. DirectX Shader Compiler (DXC)
AE 2025 supports DirectX 12 GPU rendering alongside CUDA/OpenCL:
- Set environment variable: `DXC_SDK_BASE_PATH`
- Download from: https://github.com/microsoft/DirectXShaderCompiler/releases

### 3. Boost Libraries (Optional)
Some AE 2025 GPU kernel processing uses Boost:
- Install via vcpkg: `vcpkg install boost:x64-windows-static`
- Or download from: https://www.boost.org/users/download/

## Required PiPL Flag: PF_OutFlag2_SUPPORTS_DIRECTX_RENDERING

**Critical**: AE 2025 requires the DirectX rendering flag even if you only use CUDA/OpenCL.

### PiPL Resource Update

```c
AE_Effect_Global_OutFlags_2 {
    0x2A001400  // Combined flags:
                // - PF_OutFlag2_SUPPORTS_GPU_RENDER_F32 (bit 25 = 0x02000000)
                // - PF_OutFlag2_SUPPORTS_DIRECTX_RENDERING (bit 29 = 0x20000000)
                // - PF_OutFlag2_FLOAT_COLOR_AWARE (bit 10 = 0x00000400)
                // - PF_OutFlag2_SUPPORTS_THREADED_RENDERING (bit 12 = 0x00001000)
},
```

**Before (AE 2024):** `0x0A001400`
**After (AE 2025):** `0x2A001400` (added bit 29)

### Code Update in GlobalSetup

```cpp
out_data->out_flags2 = PF_OutFlag2_SUPPORTS_SMART_RENDER |
                       PF_OutFlag2_FLOAT_COLOR_AWARE |
                       PF_OutFlag2_SUPPORTS_THREADED_RENDERING
#ifdef ENABLE_GPU_RENDERING
                       | PF_OutFlag2_SUPPORTS_GPU_RENDER_F32
                       | PF_OutFlag2_SUPPORTS_DIRECTX_RENDERING  // Required for AE 2025+
#endif
                       ;
```

## CMake Configuration for AE 2025

```cmake
option(ENABLE_GPU_RENDERING "Enable GPU acceleration (CUDA/OpenCL/Metal)" OFF)
option(ENABLE_DIRECTX_RENDERING "Enable DirectX GPU rendering (AE 2025+)" OFF)

if(ENABLE_GPU_RENDERING)
    if(WIN32)
        enable_language(CUDA)
        find_package(CUDAToolkit REQUIRED)
        message(STATUS "GPU Rendering: ENABLED (CUDA ${CUDAToolkit_VERSION})")

        # Check CUDA version for AE 2025 compatibility
        if(CUDAToolkit_VERSION VERSION_LESS "12.0")
            message(WARNING "CUDA ${CUDAToolkit_VERSION} detected. AE 2025 uses CUDA 12.8")
        endif()

        # Find Boost (optional for AE 2025)
        find_package(Boost QUIET)
        if(Boost_FOUND)
            message(STATUS "Boost: FOUND (${Boost_VERSION})")
        else()
            message(STATUS "Boost: NOT FOUND - Some AE 2025 GPU features may not work")
        endif()

        # Check for DirectX Compiler
        if(ENABLE_DIRECTX_RENDERING)
            if(DEFINED ENV{DXC_SDK_BASE_PATH})
                set(DXC_PATH $ENV{DXC_SDK_BASE_PATH})
                message(STATUS "DirectX Compiler: FOUND at ${DXC_PATH}")
                add_compile_definitions(ENABLE_DIRECTX_RENDERING)
            else()
                message(WARNING "DXC_SDK_BASE_PATH not set")
            endif()
        endif()

        find_package(OpenCL)
    endif()
endif()
```

## Debugging AE 2025 GPU Issues

### 1. Check AE GPU Preferences
**Common Gotcha**: AE may be set to "Software Only" mode!
- Go to: Edit -> Preferences -> Previews -> GPU Information
- Ensure GPU is enabled, not "Software Only"

### 2. Enable Verbose GPU Logging

```cpp
#define ENABLE_GPU_VERBOSE_DEBUG 1

// In GPUDeviceSetup, log framework details:
std::stringstream ss;
ss << "GPUDeviceSetup - AE offering: " << framework_name
   << " (" << (int)extra->input->what_gpu << ")";
ss << ", device_framework = " << device_framework_name
   << " (" << (int)device_info.device_framework << ")";
ss << ", device_index = " << extra->input->device_index;
DebugLog(ss.str().c_str());
```

### 3. Use DebugView to Monitor
- Download Sysinternals DebugView
- Filter for your plugin prefix (e.g., `[MYPLUGIN GPU]`)
- Watch for framework offers and acceptance

## AE 2025 GPU Framework Values

```cpp
PF_GPU_Framework_NONE     = 0
PF_GPU_Framework_OPENCL   = 1
PF_GPU_Framework_METAL    = 2  // macOS only
PF_GPU_Framework_CUDA     = 3
PF_GPU_Framework_DIRECTX  = 4  // New in AE 2025
```

## Troubleshooting Checklist

| Symptom | Check |
|---------|-------|
| GPU worked in AE 2024, not in 2025 | Add `PF_OutFlag2_SUPPORTS_DIRECTX_RENDERING` flag |
| Plugin falls back to CPU | Check AE Preferences -> not set to "Software Only" |
| CUDA not detected | Upgrade to CUDA 12.8+ |
| Build warnings about Boost | Install Boost via vcpkg (optional) |
| DirectX path not found | Set `DXC_SDK_BASE_PATH` environment variable |

## Summary

To upgrade an AE 2024 GPU plugin to AE 2025:

1. **Update PiPL flags**: Add `0x20000000` (DirectX bit) to `AE_Effect_Global_OutFlags_2`
2. **Update code**: Add `PF_OutFlag2_SUPPORTS_DIRECTX_RENDERING` in GlobalSetup
3. **Upgrade CUDA**: Use CUDA 12.8+ for best compatibility
4. **Install dependencies**: Boost and DXC (optional but recommended)
5. **Rebuild plugin**: Clean build with new flags
6. **Check AE settings**: Ensure GPU is enabled in AE preferences

---

# GPU Alpha Handling: Premultiplied vs Unpremultiplied (Critical!)

## The Core Problem

When implementing GPU effects that involve compositing and masking, the **alpha handling must match what After Effects expects**. Getting this wrong causes:
- Black rings/halos around edges
- Color contamination from transparent areas
- Washed out or over-bright colors

## After Effects Expects UNPREMULTIPLIED Output

!!! danger "RETRACTED 2026-07-13 — this section is WRONG"
    Everything below claiming AE GPU worlds are unpremultiplied is **incorrect**,
    and it cost real debugging time (a keyer shipped straight RGB out of CUDA for
    months, producing a hot fringe on every soft edge that read as a keying
    problem). **AE GPU worlds are PREMULTIPLIED, exactly like CPU worlds.**

    Measured, not reasoned: a probe wrote raw values straight into a GPU output
    world and the composite was read back, in an isolated comp over black with no
    other layers or effects.

    | band | wrote | premult predicts | straight predicts | CPU read | GPU read |
    |------|-------|------------------|-------------------|----------|----------|
    | A | `rgb=0.5, a=0.5`  | 0.50  | 0.25  | 0.50 | **0.50** |
    | B | `rgb=0.25, a=0.5` | 0.25  | 0.125 | 0.25 | **0.25** |
    | C | `rgb=0, a=0`      | 0     | 0     | 0    | 0 |

    The GPU renders byte-identically to the premultiplied CPU world. Straight would
    have come out half as bright.

    **But "premultiplied worlds" does NOT mean "premultiply at the end."** That
    inference is a trap: knowing the *world* convention says nothing about what
    your buffer already holds. An "apply matte" kernel that scales only alpha and
    leaves RGB alone is **already producing premultiplied output** when the input
    world was premultiplied — premultiplying again darkens every soft edge.

    Settle it by measurement, not argument: **straight colour (`premult / alpha`)
    is invariant across a correct premultiply.** Read a pixel back before and
    after. If the straight colour falls by a factor of alpha, you just did it
    twice. See
    [PreRender GPU Gating](../gpu/prerender-gpu-gating.md#premultiplied-worlds-does-not-mean-premultiply-at-the-end).

    Two traps that made this hard to see, and that will fool the next person too:

    - **Test in an isolated comp.** A grey solid sitting behind the layer makes the
      straight and premultiplied models both look plausible. Include a fully
      transparent band so the probe *measures* the background instead of assuming it.
    - **AE's frame cache does not reliably invalidate when you switch render
      engines.** Flipping to Mercury Software Only can hand you back the cached GPU
      frame. Purge between switches, and make the CPU and GPU probes visually
      distinct (e.g. horizontal vs vertical bands) so you cannot be fooled.

**Critical Discovery (WRONG - see the retraction above):** After Effects GPU worlds expect **UNPREMULTIPLIED** (straight alpha) data in the output buffer.

When you see:
```
output_worldP->pix_aspect_ratio
output_worldP->world_flags
```

The pixel data in GPU worlds is UNPREMULTIPLIED by default.

## CPU vs GPU: The SDK Confusion

### CPU Path with SDK `transfer_rect`

The CPU path uses `PF_MF_Alpha_PREMUL` flag:
```cpp
ERR(wtransformSuite->transfer_rect(
    in_data->effect_ref,
    PF_Quality_HI,
    PF_MF_Alpha_PREMUL,  // <-- This flag
    NULL,
    &source->extent_hint,
    source,
    &comp_mode,
    NULL,
    0, 0,
    dest));
```

PF_MF_Alpha_PREMUL is a compositing instruction that tells AE how to interpret the data, not a description of the storage format. This distinction is important. The SDK internally handles conversions. The world buffers themselves remain unpremultiplied.

### GPU Path Must Stay Unpremultiplied

Since SDK functions like `transfer_rect` don't work on GPU worlds directly, you must write custom kernels. These kernels must keep data **UNPREMULTIPLIED** throughout:

```cuda
// WRONG - Premultiplying causes black ring artifacts
__global__ void ApplyMaskPremultKernel(
    const float4* src,   // If src is premult, this works
    const float4* mask,
    float4* out)
{
    // This formula only works for PREMULTIPLIED data:
    result.x = src.x * mask_alpha;  // <-- Darkens semi-transparent pixels!
    result.y = src.y * mask_alpha;
    result.z = src.z * mask_alpha;
    result.w = src.w * mask_alpha;
}

// CORRECT - For UNPREMULTIPLIED data, only multiply ALPHA
__global__ void ApplyAlphaMaskKernel(
    const float4* src,   // UNPREMULTIPLIED
    const float4* mask,
    float4* out)
{
    // For unpremultiplied: RGB stays the same, only alpha changes
    result.x = src.x;  // RGB unchanged - color values stay the same
    result.y = src.y;
    result.z = src.z;
    result.w = src.w * mask_alpha;  // Only alpha is multiplied
}
```

## Complete Example: EdgeBlur Effect (MyBlurPlugin)

This effect demonstrates correct GPU alpha handling:

### Step 1: Composite (UNPREMULTIPLIED)
```cuda
// Composite eroded ON TOP of extended - both UNPREMULTIPLIED
MyPlugin_CompositeOver_CUDA(
    (const float*)temp2_gpu,       // Source: eroded (goes on top)
    (const float*)temp3_gpu,       // Dest: extended (goes below)
    (float*)output_gpu_data,       // Output: composite (unpremult)
    out_width, out_height, output_stride, cuda_stream);
```

The `CompositeOverKernel` for unpremultiplied data:
```cuda
__global__ void CompositeOverKernel(
    const float4* src,   // Top layer (UNPREMULTIPLIED)
    const float4* dst,   // Bottom layer (UNPREMULTIPLIED)
    float4* out)
{
    float top_alpha = src.w;
    float bottom_alpha = dst.w;
    float inv_top_alpha = 1.0f - top_alpha;
    float out_alpha = top_alpha + bottom_alpha * inv_top_alpha;

    if (out_alpha > 0.001f) {
        // Weight colors by their contribution to final alpha
        float top_contrib = top_alpha;
        float bottom_contrib = bottom_alpha * inv_top_alpha;
        float inv_out_alpha = 1.0f / out_alpha;

        // Blend RGB weighted by contribution, normalized by output alpha
        result.x = (top.x * top_contrib + bottom.x * bottom_contrib) * inv_out_alpha;
        result.y = (top.y * top_contrib + bottom.y * bottom_contrib) * inv_out_alpha;
        result.z = (top.z * top_contrib + bottom.z * bottom_contrib) * inv_out_alpha;
    } else {
        result = make_float4(0, 0, 0, 0);
    }
    result.w = out_alpha;
}
```

### Step 2: Create Alpha Mask
```cuda
// Gaussian blur of original alpha
MyPlugin_GaussianBlurAlpha_CUDA(
    (const float*)original_gpu,    // Original input (for alpha)
    (float*)temp3_gpu,             // Output: blurred alpha in all channels
    out_width, out_height, output_stride,
    blur_radius, cuda_stream);
```

### Step 3: Apply Mask (UNPREMULTIPLIED)
```cuda
// Apply mask - ONLY multiply alpha, NOT RGB
MyPlugin_ApplyAlphaMask_CUDA(
    (const float*)composite_gpu,   // Source: composite (unpremult)
    (const float*)mask_gpu,        // Mask: blurred alpha
    (float*)output_gpu_data,       // Output (unpremult)
    out_width, out_height, output_stride, cuda_stream);
```

The unpremultiplied mask kernel:
```cuda
__global__ void ApplyAlphaMaskKernel(
    const float4* src,   // UNPREMULTIPLIED
    const float4* mask,
    float4* out)
{
    float mask_alpha = mask.w;

    // For unpremultiplied: RGB unchanged, alpha multiplied
    result.x = src.x;  // Color stays the same
    result.y = src.y;
    result.z = src.z;
    result.w = src.w * mask_alpha;  // Only visibility changes
}
```

## The Black Ring Problem: Root Cause

When you premultiply data that was unpremultiplied:
```cuda
// Converting unpremult -> premult
result.x = src.x * src.w;  // RGB gets darkened at semi-transparent edges
```

Then apply operations designed for premultiplied data, the math is wrong. When AE tries to display this (expecting unpremult), you get:
- **Black rings** at semi-transparent edges
- **Color bleeding** from transparent areas
- **Washed out** appearance

## Summary: GPU Alpha Rules

| Operation | Data Type | What to Do |
|-----------|-----------|------------|
| Input from AE | Unpremultiplied | Use directly |
| Blur kernels | Unpremultiplied | Weight by alpha for RGB, average for alpha |
| Composite Over | Unpremultiplied | Blend RGB by alpha contribution |
| Apply Mask | Unpremultiplied | Only multiply ALPHA channel |
| Output to AE | Unpremultiplied | Return as-is |

**Golden Rule:** If your GPU extension kernels output unpremultiplied data (dividing by alpha at the end), keep EVERYTHING unpremultiplied throughout the pipeline. Don't convert to premultiplied just because the CPU path mentions `PF_MF_Alpha_PREMUL`.

## Debug Output Modes

When debugging alpha issues, add output mode options to visualize each step:

```cpp
enum OutputMode {
    OUTPUT_FINAL_RESULT = 1,
    OUTPUT_EROSION_ONLY,
    OUTPUT_EXTEND_ONLY,
    OUTPUT_EDGE_BLUR,
    OUTPUT_EDGE_BLUR_MASK,
    // Debug modes for EdgeBlur steps
    OUTPUT_DEBUG_EXTENDED,      // Show raw extended output
    OUTPUT_DEBUG_ERODED,        // Show raw eroded output
    OUTPUT_DEBUG_COMPOSITE,     // Show composite before mask
    OUTPUT_DEBUG_MASK,          // Show the alpha mask
    OUTPUT_DEBUG_FINAL          // Show result before final step
};
```

This lets you compare GPU vs CPU at each step to find where the mismatch occurs.
