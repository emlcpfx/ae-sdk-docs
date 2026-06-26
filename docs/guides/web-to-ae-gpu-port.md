# Porting WebGPU to After Effects Native GPU API

This document describes how to port a plugin from WebGPU (with CPU readback) to After Effects' native GPU API using OpenCL (Windows) or Metal (macOS). The key benefit is eliminating CPU-to-GPU data transfers, keeping textures on the GPU throughout the render pipeline.

## Why Port?

**WebGPU Limitation**: WebGPU requires staging buffers and CPU readback to get rendered results back to After Effects. This creates a significant bottleneck (~150ms+ per frame for HD content).

**AE Native GPU**: After Effects provides GPU memory directly to plugins via `PF_GPUDeviceSuite1`. Textures stay on GPU throughout the pipeline, reducing render times to ~10-20ms.

## Prerequisites

### Windows
- **OpenCL SDK**: Included with GPU drivers (NVIDIA, AMD, Intel all support it)
- **After Effects SDK**: `Headers/AE_EffectGPUSuites.h` contains the GPU suite definitions

### macOS
- **Metal Framework**: Built into macOS
- **After Effects SDK**: Same header files

## Step 1: Update PiPL Resource

Add GPU support flag to `AE_Effect_Global_OutFlags_2`:

```c
// In your .r PiPL file
AE_Effect_Global_OutFlags_2 {
    0x0AA81480  // Include SUPPORTS_GPU_RENDER_F32
}
```

**Flag breakdown for the hex value 0x0AA81480:**
- `PF_OutFlag2_SUPPORTS_SMART_RENDER` = `1L << 10` = 0x00000400
- `PF_OutFlag2_FLOAT_COLOR_AWARE` = `1L << 12` = 0x00001000
- `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` = `1L << 27` = 0x08000000
- `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` = `1L << 25` = 0x02000000
- Additional flags combined into the value (check AE_Effect.h for a full list of OutFlag2 bit definitions and verify which flags your plugin needs)

**IMPORTANT**: PiPL files don't support the `|` operator. You must pre-compute the combined hex value.

## Step 2: Update GlobalSetup

Add the GPU flag in code (must match PiPL exactly):

```cpp
static PF_Err GlobalSetup(PF_InData* in_data, PF_OutData* out_data, ...) {
    out_data->out_flags2 =
        PF_OutFlag2_SUPPORTS_SMART_RENDER |
        PF_OutFlag2_FLOAT_COLOR_AWARE |
        PF_OutFlag2_SUPPORTS_THREADED_RENDERING |
        // ... other flags ...
#ifdef HAS_AE_GPU
        | PF_OutFlag2_SUPPORTS_GPU_RENDER_F32
#endif
        ;
    return PF_Err_NONE;
}
```

## Step 3: Add GPU Command Handlers

Add three new command handlers to your `EffectMain` switch:

```cpp
case PF_Cmd_GPU_DEVICE_SETUP:
    err = AEGPURenderer::GPUDeviceSetup(in_data, out_data,
        reinterpret_cast<PF_GPUDeviceSetupExtra*>(extra));
    break;

case PF_Cmd_GPU_DEVICE_SETDOWN:
    err = AEGPURenderer::GPUDeviceSetdown(in_data, out_data,
        reinterpret_cast<PF_GPUDeviceSetdownExtra*>(extra));
    break;

case PF_Cmd_SMART_RENDER_GPU:
    err = AEGPURenderer::SmartRenderGPU(in_data, out_data,
        reinterpret_cast<PF_SmartRenderExtra*>(extra));
    break;
```

## Step 4: Create GPU Renderer Class

### Header (AEGPURenderer.h)

```cpp
#ifndef AE_GPU_RENDERER_H
#define AE_GPU_RENDERER_H

#ifdef HAS_AE_GPU

#include "AE_Effect.h"
#include "AE_EffectCB.h"
#include "AE_EffectCBSuites.h"
#include "AE_EffectGPUSuites.h"
#include "AEFX_SuiteHelper.h"
#include "AEGP_SuiteHandler.h"

#ifdef AE_OS_WIN
    #include <CL/cl.h>
#endif

// GPU data stored per-device
struct OpenCLGPUData {
    cl_kernel my_kernel;
    // ... add your kernels
};

class AEGPURenderer {
public:
    static PF_Err GPUDeviceSetup(PF_InData*, PF_OutData*, PF_GPUDeviceSetupExtra*);
    static PF_Err GPUDeviceSetdown(PF_InData*, PF_OutData*, PF_GPUDeviceSetdownExtra*);
    static PF_Err SmartRenderGPU(PF_InData*, PF_OutData*, PF_SmartRenderExtra*);

    static AEGPURenderer* GetInstance();

private:
    PF_Err RenderOpenCL(PF_InData*, PF_OutData*, PF_SmartRenderExtra*,
                        PF_EffectWorld* input, PF_EffectWorld* output);
};

#endif // HAS_AE_GPU
#endif
```

### Implementation (AEGPURenderer.cpp)

```cpp
#ifdef HAS_AE_GPU

#include "AEGPURenderer.h"
#include "AE_Macros.h"  // Required for ERR macro

// Your OpenCL kernel source as a string
//
// IMPORTANT: Source and destination buffers may have different pitches.
// Pass both src_pitch and dst_pitch to your GPU kernel.
const char* kMyKernelSource = R"CL(
__kernel void MyKernel(
    __global const float4* src,
    __global float4* dst,
    int src_pitch,
    int dst_pitch,
    int width,
    int height)
{
    int x = get_global_id(0);
    int y = get_global_id(1);
    if (x >= width || y >= height) return;

    int src_idx = y * src_pitch + x;
    int dst_idx = y * dst_pitch + x;
    dst[dst_idx] = src[src_idx];  // Example: copy
}
)CL";

PF_Err AEGPURenderer::GPUDeviceSetup(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_GPUDeviceSetupExtra* extra)
{
    PF_Err err = PF_Err_NONE;

    PF_GPUDeviceInfo device_info;
    AEFX_CLR_STRUCT(device_info);

    AEFX_SuiteScoper<PF_HandleSuite1> handle_suite = AEFX_SuiteScoper<PF_HandleSuite1>(
        in_data, kPFHandleSuite, kPFHandleSuiteVersion1, out_data);

    AEFX_SuiteScoper<PF_GPUDeviceSuite1> gpu_suite = AEFX_SuiteScoper<PF_GPUDeviceSuite1>(
        in_data, kPFGPUDeviceSuite, kPFGPUDeviceSuiteVersion1, out_data);

    gpu_suite->GetDeviceInfo(in_data->effect_ref, extra->input->device_index, &device_info);

#ifdef AE_OS_WIN
    if (extra->input->what_gpu == PF_GPU_Framework_OPENCL) {
        // Allocate GPU data handle
        PF_Handle gpu_dataH = handle_suite->host_new_handle(sizeof(OpenCLGPUData));
        if (!gpu_dataH) return PF_Err_OUT_OF_MEMORY;

        OpenCLGPUData* gpu_data = reinterpret_cast<OpenCLGPUData*>(*gpu_dataH);
        memset(gpu_data, 0, sizeof(OpenCLGPUData));

        cl_context context = (cl_context)device_info.contextPV;
        cl_device_id device = (cl_device_id)device_info.devicePV;

        // Create program from source
        cl_int result;
        const char* sources[] = { kMyKernelSource };
        size_t lengths[] = { strlen(kMyKernelSource) };

        cl_program program = clCreateProgramWithSource(context, 1, sources, lengths, &result);
        if (result != CL_SUCCESS) {
            handle_suite->host_dispose_handle(gpu_dataH);
            return PF_Err_INTERNAL_STRUCT_DAMAGED;
        }

        // Build with fast math
        result = clBuildProgram(program, 1, &device,
            "-cl-single-precision-constant -cl-fast-relaxed-math", nullptr, nullptr);

        if (result != CL_SUCCESS) {
            // Get build log for debugging
            size_t log_size;
            clGetProgramBuildInfo(program, device, CL_PROGRAM_BUILD_LOG, 0, nullptr, &log_size);
            std::vector<char> log(log_size + 1);
            clGetProgramBuildInfo(program, device, CL_PROGRAM_BUILD_LOG, log_size, log.data(), nullptr);
            // Output log.data() to debug

            clReleaseProgram(program);
            handle_suite->host_dispose_handle(gpu_dataH);
            return PF_Err_INTERNAL_STRUCT_DAMAGED;
        }

        // Create kernels
        gpu_data->my_kernel = clCreateKernel(program, "MyKernel", &result);
        clReleaseProgram(program);  // Kernels keep reference

        if (result != CL_SUCCESS) {
            handle_suite->host_dispose_handle(gpu_dataH);
            return PF_Err_INTERNAL_STRUCT_DAMAGED;
        }

        // Store handle and signal GPU support
        extra->output->gpu_data = gpu_dataH;

        // IMPORTANT: Do NOT overwrite out_flags2 during GPUDeviceSetup.
        // Set flags only in GlobalSetup. Use |= to add flags, not = which
        // clears other flags.
        out_data->out_flags2 |= PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;
    }
#endif

    return err;
}

PF_Err AEGPURenderer::GPUDeviceSetdown(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_GPUDeviceSetdownExtra* extra)
{
#ifdef AE_OS_WIN
    if (extra->input->what_gpu == PF_GPU_Framework_OPENCL) {
        PF_Handle gpu_dataH = (PF_Handle)extra->input->gpu_data;
        if (gpu_dataH) {
            OpenCLGPUData* gpu_data = reinterpret_cast<OpenCLGPUData*>(*gpu_dataH);

            if (gpu_data->my_kernel) clReleaseKernel(gpu_data->my_kernel);

            AEFX_SuiteScoper<PF_HandleSuite1> handle_suite = AEFX_SuiteScoper<PF_HandleSuite1>(
                in_data, kPFHandleSuite, kPFHandleSuiteVersion1, out_data);
            handle_suite->host_dispose_handle(gpu_dataH);
        }
    }
#endif
    return PF_Err_NONE;
}

PF_Err AEGPURenderer::SmartRenderGPU(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_SmartRenderExtra* extra)
{
    PF_Err err = PF_Err_NONE;

    // Checkout input/output worlds
    PF_EffectWorld* input_world = nullptr;
    PF_EffectWorld* output_world = nullptr;

    ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world));
    ERR(extra->cb->checkout_output(in_data->effect_ref, &output_world));

    if (err || !input_world || !output_world) {
        return err ? err : PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    // Verify GPU pixel format
    AEFX_SuiteScoper<PF_WorldSuite2> world_suite = AEFX_SuiteScoper<PF_WorldSuite2>(
        in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);

    PF_PixelFormat pixel_format;
    ERR(world_suite->PF_GetPixelFormat(input_world, &pixel_format));

    if (pixel_format != PF_PixelFormat_GPU_BGRA128) {
        return PF_Err_UNRECOGNIZED_PARAM_TYPE;
    }

#ifdef AE_OS_WIN
    if (extra->input->what_gpu == PF_GPU_Framework_OPENCL) {
        AEGPURenderer* instance = GetInstance();
        err = instance->RenderOpenCL(in_data, out_data, extra, input_world, output_world);
    }
#endif

    ERR(extra->cb->checkin_layer_pixels(in_data->effect_ref, 0));
    return err;
}

PF_Err AEGPURenderer::RenderOpenCL(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_SmartRenderExtra* extra,
    PF_EffectWorld* input_world,
    PF_EffectWorld* output_world)
{
    PF_Err err = PF_Err_NONE;

    AEFX_SuiteScoper<PF_GPUDeviceSuite1> gpu_suite = AEFX_SuiteScoper<PF_GPUDeviceSuite1>(
        in_data, kPFGPUDeviceSuite, kPFGPUDeviceSuiteVersion1, out_data);

    PF_GPUDeviceInfo device_info;
    ERR(gpu_suite->GetDeviceInfo(in_data->effect_ref, extra->input->device_index, &device_info));

    // Get GPU data (kernels)
    PF_Handle gpu_dataH = (PF_Handle)extra->input->gpu_data;
    OpenCLGPUData* gpu_data = reinterpret_cast<OpenCLGPUData*>(*gpu_dataH);

    cl_context context = (cl_context)device_info.contextPV;
    cl_command_queue queue = (cl_command_queue)device_info.command_queuePV;

    int width = input_world->width;
    int height = input_world->height;
    int bytes_per_pixel = 16;  // BGRA128 = 4 floats = 16 bytes
    int src_pitch = input_world->rowbytes / bytes_per_pixel;
    int dst_pitch = output_world->rowbytes / bytes_per_pixel;

    // Get GPU memory pointers (these are cl_mem handles)
    void* src_mem = nullptr;
    void* dst_mem = nullptr;
    ERR(gpu_suite->GetGPUWorldData(in_data->effect_ref, input_world, &src_mem));
    ERR(gpu_suite->GetGPUWorldData(in_data->effect_ref, output_world, &dst_mem));

    cl_mem cl_src = (cl_mem)src_mem;
    cl_mem cl_dst = (cl_mem)dst_mem;

    // Set kernel arguments
    clSetKernelArg(gpu_data->my_kernel, 0, sizeof(cl_mem), &cl_src);
    clSetKernelArg(gpu_data->my_kernel, 1, sizeof(cl_mem), &cl_dst);
    clSetKernelArg(gpu_data->my_kernel, 2, sizeof(int), &src_pitch);
    clSetKernelArg(gpu_data->my_kernel, 3, sizeof(int), &dst_pitch);
    clSetKernelArg(gpu_data->my_kernel, 4, sizeof(int), &width);
    clSetKernelArg(gpu_data->my_kernel, 5, sizeof(int), &height);

    // Dispatch kernel
    size_t global_size[2] = { (size_t)((width + 15) / 16 * 16), (size_t)((height + 15) / 16 * 16) };
    size_t local_size[2] = { 16, 16 };

    clEnqueueNDRangeKernel(queue, gpu_data->my_kernel, 2, nullptr,
                           global_size, local_size, 0, nullptr, nullptr);

    return err;
}

#endif // HAS_AE_GPU
```

## Step 5: Update CMakeLists.txt

```cmake
option(USE_AE_GPU "Enable AE native GPU acceleration (OpenCL/Metal)" ON)

if(USE_AE_GPU)
    message(STATUS "AE Native GPU acceleration ENABLED")

    target_compile_definitions(${PROJECT_NAME} PRIVATE HAS_AE_GPU=1)

    target_sources(${PROJECT_NAME} PRIVATE
        src/gpu/AEGPURenderer.cpp
    )

    target_include_directories(${PROJECT_NAME} PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src/gpu
    )

    if(WIN32)
        find_package(OpenCL REQUIRED)
        target_link_libraries(${PROJECT_NAME} PRIVATE OpenCL::OpenCL)
        target_compile_definitions(${PROJECT_NAME} PRIVATE AE_OS_WIN=1)
    elseif(APPLE)
        find_library(METAL_FRAMEWORK Metal REQUIRED)
        find_library(METALKIT_FRAMEWORK MetalKit REQUIRED)
        target_link_libraries(${PROJECT_NAME} PRIVATE
            ${METAL_FRAMEWORK}
            ${METALKIT_FRAMEWORK}
        )
        target_compile_definitions(${PROJECT_NAME} PRIVATE AE_OS_MAC=1)
    endif()
endif()
```

## Step 6: Port WGSL Shaders to OpenCL

### Key Differences

| WGSL | OpenCL |
|------|--------|
| `@group(0) @binding(0)` | `__global` function parameters |
| `var<storage>` | `__global float4*` |
| `textureSampleLevel()` | Manual buffer indexing |
| `vec4<f32>` | `float4` |
| `fn` | `__kernel void` |
| `@workgroup_size(8,8)` | Specified at dispatch time |

### Example: Blur Kernel

**WGSL:**
```wgsl
@compute @workgroup_size(8, 8)
fn blur_horizontal(@builtin(global_invocation_id) id: vec3<u32>) {
    let x = i32(id.x);
    let y = i32(id.y);
    // ...
}
```

**OpenCL:**
```c
__kernel void BlurHorizontalKernel(
    __global const float4* src,
    __global float4* dst,
    int src_pitch,
    int dst_pitch,
    int width,
    int height,
    float blur_radius)
{
    int x = get_global_id(0);
    int y = get_global_id(1);
    if (x >= width || y >= height) return;

    // Access pixels via: src[y * src_pitch + x]
    // Note: src and dst may have different pitches
    // ...
}
```

## Common Pitfalls

1. **PiPL vs Code Flag Mismatch**: AE checks that `out_flags2` matches PiPL exactly. Compute your hex values carefully.

2. **Missing AE_Macros.h**: The `ERR()` macro is defined here. Include it or you'll get template errors.

3. **Pitch vs Width**: GPU buffers have pitch (rowbytes / bytes_per_pixel), not width. Always use pitch for indexing. Source and destination buffers may have different pitches -- pass both to your kernel.

4. **Pixel Format**: GPU rendering uses `PF_PixelFormat_GPU_BGRA128` (4x float32). Verify this in SmartRenderGPU.

5. **Kernel Build Errors**: Always get and log the build output when `clBuildProgram` fails.

6. **out_flags2 overwrite in GPUDeviceSetup**: Do NOT use `=` to set `out_data->out_flags2` in GPUDeviceSetup, as this clears all flags set by GlobalSetup. Use `|=` to add flags.

## Debugging Tips

1. **OutputDebugString (Windows)**: Use DebugView to see log messages.

2. **Benchmarking**: Add `clFinish()` after each kernel dispatch and measure with `std::chrono`.

3. **Start Simple**: First just copy input to output, then add complexity.

## Performance Expectations

| Metric | WebGPU | AE Native GPU |
|--------|--------|---------------|
| Frame Time (1080p) | ~150ms | ~10-20ms |
| GPU Memory | Duplicated | Shared |
| CPU Load | High (transfers) | Minimal |

## Files Summary

```
src/
+-- YourPlugin.cpp        # Add GPU command handlers
+-- gpu/
|   +-- AEGPURenderer.h   # GPU renderer class
|   +-- AEGPURenderer.cpp # OpenCL/Metal implementation
|   +-- YourKernel.h      # Kernel source as string constant
resources/
+-- YourPluginPiPL.r      # Update outflags2
CMakeLists.txt            # Add OpenCL/Metal dependencies
```
