# OpenCL Setup for AE GPU Effects

> OpenCL context from AE, kernel compilation, buffer creation, dispatch patterns, synchronization, and cross-platform considerations.

## Overview

After Effects offers `PF_GPU_Framework_OPENCL` on systems with AMD or Intel GPUs (and optionally NVIDIA, though CUDA takes priority on NVIDIA hardware). AE provides the OpenCL context and command queue through `PF_GPUDeviceInfo`; your plugin compiles kernels at device setup time and dispatches them during `PF_Cmd_SMART_RENDER_GPU`.

OpenCL is the most broadly compatible GPU framework in AE, working on Windows and macOS across GPU vendors. However, Apple deprecated OpenCL on macOS (favoring Metal), and AMD's OpenCL support on Windows can be inconsistent. Test thoroughly on target hardware.

---

## Lifecycle

```
PF_Cmd_GLOBAL_SETUP
    Set PF_OutFlag2_SUPPORTS_GPU_RENDER_F32

PF_Cmd_GPU_DEVICE_SETUP
    Check what_gpu == PF_GPU_Framework_OPENCL
    Get cl_context and cl_device_id from PF_GPUDeviceInfo
    Compile kernel program from source
    Create cl_kernel objects
    Store in gpu_data

PF_Cmd_SMART_PRE_RENDER
    Set PF_RenderOutputFlag_GPU_RENDER_POSSIBLE

PF_Cmd_SMART_RENDER_GPU
    Retrieve gpu_data (kernels)
    Get GPU buffer pointers (cl_mem objects)
    Set kernel arguments
    Enqueue kernel
    (no explicit sync needed -- AE handles it)

PF_Cmd_GPU_DEVICE_SETDOWN
    Release cl_kernel objects
    Free gpu_data handle
```

---

## Device Setup: Kernel Compilation

Unlike CUDA (statically linked), OpenCL kernels are compiled at runtime from source strings. This happens during `PF_Cmd_GPU_DEVICE_SETUP`.

### The Complete Pattern

```cpp
struct OpenCLGPUData {
    cl_kernel myEffectKernel;
    cl_kernel secondPassKernel;
};

static PF_Err GPUDeviceSetup(
    PF_InData *in_dataP,
    PF_OutData *out_dataP,
    PF_GPUDeviceSetupExtra *extraP)
{
    PF_Err err = PF_Err_NONE;

    if (extraP->input->what_gpu != PF_GPU_Framework_OPENCL)
        return err;

    // Get OpenCL handles from AE
    PF_GPUDeviceInfo device_info;
    AEFX_CLR_STRUCT(device_info);

    AEFX_SuiteScoper<PF_GPUDeviceSuite1> gpuSuite =
        AEFX_SuiteScoper<PF_GPUDeviceSuite1>(in_dataP,
            kPFGPUDeviceSuite, kPFGPUDeviceSuiteVersion1, out_dataP);

    gpuSuite->GetDeviceInfo(in_dataP->effect_ref,
        extraP->input->device_index, &device_info);

    cl_context context = (cl_context)device_info.contextPV;
    cl_device_id device = (cl_device_id)device_info.devicePV;

    // Prepare kernel source
    // Option 1: Embedded string (from .cl.h file generated at build time)
    char const *defines = "#define GF_OPENCL_SUPPORTS_16F 0\n";
    char const *strings[] = { defines, kMyKernel_OpenCLString };
    size_t sizes[] = { strlen(defines), strlen(kMyKernel_OpenCLString) };

    // Create program
    cl_int cl_err = CL_SUCCESS;
    cl_program program = clCreateProgramWithSource(
        context, 2, strings, sizes, &cl_err);

    if (cl_err != CL_SUCCESS) {
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    // Compile
    cl_err = clBuildProgram(program, 1, &device,
        "-cl-single-precision-constant -cl-fast-relaxed-math",
        NULL, NULL);

    if (cl_err != CL_SUCCESS) {
        // Get build log for debugging
        size_t log_size;
        clGetProgramBuildInfo(program, device,
            CL_PROGRAM_BUILD_LOG, 0, NULL, &log_size);
        char *log = (char*)malloc(log_size);
        clGetProgramBuildInfo(program, device,
            CL_PROGRAM_BUILD_LOG, log_size, log, NULL);
        // log contains the compiler error messages
        free(log);
        clReleaseProgram(program);
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    // Create kernel objects
    AEFX_SuiteScoper<PF_HandleSuite1> handleSuite =
        AEFX_SuiteScoper<PF_HandleSuite1>(in_dataP,
            kPFHandleSuite, kPFHandleSuiteVersion1, out_dataP);

    PF_Handle gpu_dataH = handleSuite->host_new_handle(sizeof(OpenCLGPUData));
    OpenCLGPUData *clData = reinterpret_cast<OpenCLGPUData*>(*gpu_dataH);

    clData->myEffectKernel = clCreateKernel(program, "MyEffectKernel", &cl_err);
    if (cl_err != CL_SUCCESS) {
        handleSuite->host_dispose_handle(gpu_dataH);
        clReleaseProgram(program);
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    clData->secondPassKernel = clCreateKernel(program, "SecondPassKernel", &cl_err);
    if (cl_err != CL_SUCCESS) {
        clReleaseKernel(clData->myEffectKernel);
        handleSuite->host_dispose_handle(gpu_dataH);
        clReleaseProgram(program);
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    // Program can be released after kernels are created
    clReleaseProgram(program);

    // Store for render-time use
    extraP->output->gpu_data = gpu_dataH;
    out_dataP->out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;

    return err;
}
```

### Build Options

Common `clBuildProgram` options for AE plugins:

| Option | Effect |
|--------|--------|
| `-cl-single-precision-constant` | Treat floating-point constants as single precision. Avoids double-precision promotion. |
| `-cl-fast-relaxed-math` | Allow aggressive FP optimizations. May slightly change results. |
| `-cl-mad-enable` | Allow fused multiply-add. Minor precision tradeoff for speed. |
| `-D SYMBOL=VALUE` | Define preprocessor symbols for conditional compilation |

> **Pitfall**: Some AMD OpenCL implementations have bugs with `-cl-fast-relaxed-math`. If you see NaN or incorrect results on AMD hardware, try removing this flag.

---

## Device Setdown: Cleanup

Release kernel objects and free the handle:

```cpp
static PF_Err GPUDeviceSetdown(
    PF_InData *in_dataP,
    PF_OutData *out_dataP,
    PF_GPUDeviceSetdownExtra *extraP)
{
    if (extraP->input->what_gpu != PF_GPU_Framework_OPENCL)
        return PF_Err_NONE;

    PF_Handle gpu_dataH = (PF_Handle)extraP->input->gpu_data;
    OpenCLGPUData *clData = reinterpret_cast<OpenCLGPUData*>(*gpu_dataH);

    clReleaseKernel(clData->myEffectKernel);
    clReleaseKernel(clData->secondPassKernel);

    AEFX_SuiteScoper<PF_HandleSuite1> handleSuite =
        AEFX_SuiteScoper<PF_HandleSuite1>(in_dataP,
            kPFHandleSuite, kPFHandleSuiteVersion1, out_dataP);

    handleSuite->host_dispose_handle(gpu_dataH);

    return PF_Err_NONE;
}
```

---

## Writing OpenCL Kernels

### Basic Kernel Structure

```opencl
__kernel void MyEffectKernel(
    __global const float4 *src,
    __global       float4 *dst,
    int srcPitch,
    int dstPitch,
    int is16f,
    int width,
    int height,
    float brightness,
    float contrast)
{
    int x = get_global_id(0);
    int y = get_global_id(1);

    if (x >= width || y >= height)
        return;

    float4 pixel = src[y * srcPitch + x];

    // BGRA order: .x=Blue, .y=Green, .z=Red, .w=Alpha
    // (same as .s0=Blue, .s1=Green, .s2=Red, .s3=Alpha)

    pixel.x = clamp(pixel.x * contrast + brightness, 0.0f, 1.0f);
    pixel.y = clamp(pixel.y * contrast + brightness, 0.0f, 1.0f);
    pixel.z = clamp(pixel.z * contrast + brightness, 0.0f, 1.0f);

    dst[y * dstPitch + x] = pixel;
}
```

### Channel Order

OpenCL float4 uses the same BGRA layout as CUDA and Metal in AE:

| Accessor | Channel |
|----------|---------|
| `.x` / `.s0` | Blue |
| `.y` / `.s1` | Green |
| `.z` / `.s2` | Red |
| `.w` / `.s3` | Alpha |

### OpenCL vs CUDA Accessor Comparison

| Operation | CUDA | OpenCL |
|-----------|------|--------|
| Create float4 | `make_float4(b, g, r, a)` | `(float4)(b, g, r, a)` |
| Component access | `pixel.x` | `pixel.x` or `pixel.s0` |
| Thread X coordinate | `blockIdx.x * blockDim.x + threadIdx.x` | `get_global_id(0)` |
| Thread Y coordinate | `blockIdx.y * blockDim.y + threadIdx.y` | `get_global_id(1)` |
| Local thread ID | `threadIdx.x` | `get_local_id(0)` |
| Sync shared memory | `__syncthreads()` | `barrier(CLK_LOCAL_MEM_FENCE)` |
| Shared/local memory | `__shared__` | `__local` |

---

## Dispatch: Setting Arguments and Enqueueing

### Setting Kernel Arguments

OpenCL requires setting each argument individually by index:

```cpp
static PF_Err SmartRenderGPU_OpenCL(
    PF_InData *in_dataP,
    PF_OutData *out_dataP,
    PF_EffectWorld *input_worldP,
    PF_EffectWorld *output_worldP,
    PF_SmartRenderExtra *extraP,
    float brightness,
    float contrast)
{
    PF_Err err = PF_Err_NONE;

    AEFX_SuiteScoper<PF_GPUDeviceSuite1> gpuSuite =
        AEFX_SuiteScoper<PF_GPUDeviceSuite1>(in_dataP,
            kPFGPUDeviceSuite, kPFGPUDeviceSuiteVersion1, out_dataP);

    PF_GPUDeviceInfo device_info;
    gpuSuite->GetDeviceInfo(in_dataP->effect_ref,
        extraP->input->device_index, &device_info);

    // Get GPU buffer pointers (these are cl_mem objects)
    void *src_mem = nullptr, *dst_mem = nullptr;
    gpuSuite->GetGPUWorldData(in_dataP->effect_ref, input_worldP, &src_mem);
    gpuSuite->GetGPUWorldData(in_dataP->effect_ref, output_worldP, &dst_mem);

    cl_mem cl_src = (cl_mem)src_mem;
    cl_mem cl_dst = (cl_mem)dst_mem;

    // Calculate pitch
    int bytes_per_pixel = 16;
    int srcPitch = input_worldP->rowbytes / bytes_per_pixel;
    int dstPitch = output_worldP->rowbytes / bytes_per_pixel;
    int is16f = 0;
    int width  = input_worldP->width;
    int height = input_worldP->height;

    // Retrieve compiled kernel
    PF_Handle gpu_dataH = (PF_Handle)extraP->input->gpu_data;
    OpenCLGPUData *clData = reinterpret_cast<OpenCLGPUData*>(*gpu_dataH);

    // Set arguments by index
    cl_uint idx = 0;
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(cl_mem), &cl_src));
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(cl_mem), &cl_dst));
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(int), &srcPitch));
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(int), &dstPitch));
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(int), &is16f));
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(int), &width));
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(int), &height));
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(float), &brightness));
    CL_ERR(clSetKernelArg(clData->myEffectKernel, idx++, sizeof(float), &contrast));

    // Enqueue
    size_t localSize[2]  = { 16, 16 };
    size_t globalSize[2] = {
        RoundUp(width,  localSize[0]),
        RoundUp(height, localSize[1])
    };

    CL_ERR(clEnqueueNDRangeKernel(
        (cl_command_queue)device_info.command_queuePV,
        clData->myEffectKernel,
        2,          // work dimensions
        NULL,       // global offset (always NULL)
        globalSize,
        localSize,
        0, NULL, NULL));  // events

    return err;
}
```

### The RoundUp Helper

Global work size must be a multiple of local work size in OpenCL:

```cpp
static size_t RoundUp(size_t value, size_t multiple)
{
    return value ? ((value + multiple - 1) / multiple) * multiple : 0;
}
```

> **Critical**: Unlike CUDA (where the grid size is in blocks), OpenCL's `global_work_size` is in total work items. If your image is 1920x1080 and local size is 16x16, global size is 1920 rounded up to 1936 by 1088.

---

## Error Handling

### The CL2Err Pattern

The SDK uses a helper to convert OpenCL errors to AE errors:

```cpp
inline PF_Err CL2Err(cl_int cl_result) {
    if (cl_result == CL_SUCCESS)
        return PF_Err_NONE;
    else
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
}

#define CL_ERR(FUNC) ERR(CL2Err(FUNC))
```

### Common OpenCL Errors

| Error Code | Name | Likely Cause |
|-----------|------|-------------|
| -11 | `CL_BUILD_PROGRAM_FAILURE` | Kernel source has syntax errors |
| -38 | `CL_INVALID_MEM_OBJECT` | Passing invalid cl_mem to kernel |
| -46 | `CL_INVALID_KERNEL_NAME` | Kernel function name not found in program |
| -47 | `CL_INVALID_KERNEL_DEFINITION` | Kernel signature mismatch |
| -48 | `CL_INVALID_KERNEL` | Using a released or invalid kernel object |
| -51 | `CL_INVALID_ARG_SIZE` | sizeof mismatch on clSetKernelArg |
| -52 | `CL_INVALID_ARG_VALUE` | NULL pointer or invalid value for argument |
| -54 | `CL_INVALID_WORK_GROUP_SIZE` | Local size does not divide global size |
| -55 | `CL_INVALID_WORK_ITEM_SIZE` | Local size exceeds device maximum |

### Getting the Build Log

When kernel compilation fails, the build log tells you exactly what went wrong:

```cpp
cl_int build_result = clBuildProgram(program, 1, &device, options, NULL, NULL);
if (build_result != CL_SUCCESS) {
    size_t log_size;
    clGetProgramBuildInfo(program, device,
        CL_PROGRAM_BUILD_LOG, 0, NULL, &log_size);

    std::vector<char> log(log_size);
    clGetProgramBuildInfo(program, device,
        CL_PROGRAM_BUILD_LOG, log_size, log.data(), NULL);

    // In debug builds, output to debugger
    OutputDebugStringA(log.data());  // Windows
}
```

---

## Passing Structs to OpenCL Kernels

Unlike CUDA (where you can pass structs by value), OpenCL kernel arguments must be set one at a time. Two approaches:

### Approach 1: Individual Arguments (SDK Pattern)

Set each field as a separate kernel argument. This is the most portable approach and what the SDK sample uses:

```cpp
CL_ERR(clSetKernelArg(kernel, 0, sizeof(cl_mem), &cl_src));
CL_ERR(clSetKernelArg(kernel, 1, sizeof(cl_mem), &cl_dst));
CL_ERR(clSetKernelArg(kernel, 2, sizeof(int), &params.width));
CL_ERR(clSetKernelArg(kernel, 3, sizeof(int), &params.height));
CL_ERR(clSetKernelArg(kernel, 4, sizeof(float), &params.brightness));
```

### Approach 2: Buffer-Backed Struct

Create a cl_mem buffer containing the struct, and pass it as a single buffer argument:

```cpp
// Host side
cl_mem param_buf = clCreateBuffer(
    (cl_context)device_info.contextPV,
    CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR,
    sizeof(MyParams), &params, &cl_err);

clSetKernelArg(kernel, 2, sizeof(cl_mem), &param_buf);

// Kernel side
__kernel void MyKernel(
    __global const float4 *src,
    __global       float4 *dst,
    __global const MyParams *params)
{
    int w = params->width;
    // ...
}

// Cleanup after dispatch
clReleaseMemObject(param_buf);
```

> **Warning**: Struct layout must match between host C++ and device OpenCL. Use `__attribute__((packed))` or explicit padding if needed. Different compilers may pad structs differently.

---

## Synchronization

### Within a Single Command Queue

OpenCL commands enqueued to the same command queue execute in order. If your effect dispatches two kernels sequentially (pass 1, pass 2), they will execute in order without explicit synchronization:

```cpp
// Pass 1 enqueue
clEnqueueNDRangeKernel(queue, kernel1, ...);

// Pass 2 enqueue -- guaranteed to execute after pass 1 completes
clEnqueueNDRangeKernel(queue, kernel2, ...);
```

### Event-Based Synchronization (Advanced)

For finer-grained control or timing:

```cpp
cl_event pass1_event;
clEnqueueNDRangeKernel(queue, kernel1, 2, NULL,
    globalSize, localSize, 0, NULL, &pass1_event);

// Pass 2 waits for pass 1
clEnqueueNDRangeKernel(queue, kernel2, 2, NULL,
    globalSize, localSize, 1, &pass1_event, NULL);

clReleaseEvent(pass1_event);
```

### Explicit Flush and Finish

```cpp
clFlush(queue);   // Submit all enqueued commands to the device
clFinish(queue);  // Block until all commands complete
```

In the AE GPU rendering context, you typically do NOT need to call `clFinish`. AE manages synchronization. However, if you need to read back data to the CPU within the same render call, you must ensure completion first.

---

## Creating Additional Buffers

### Through PF_GPUDeviceSuite (Preferred)

```cpp
// Image buffer with metadata
PF_EffectWorld *temp = nullptr;
gpuSuite->CreateGPUWorld(in_dataP->effect_ref,
    device_index, width, height,
    par, field, PF_PixelFormat_GPU_BGRA128, false, &temp);

void *temp_mem = nullptr;
gpuSuite->GetGPUWorldData(in_dataP->effect_ref, temp, &temp_mem);
cl_mem cl_temp = (cl_mem)temp_mem;

// ... use in kernel ...

gpuSuite->DisposeGPUWorld(in_dataP->effect_ref, temp);
```

### Through clCreateBuffer Directly (Use Sparingly)

For non-image data (LUTs, small parameter buffers):

```cpp
cl_context context = (cl_context)device_info.contextPV;

// Create and upload a lookup table
float lut[1024];
// ... fill lut ...

cl_int cl_err;
cl_mem cl_lut = clCreateBuffer(context,
    CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR,
    sizeof(lut), lut, &cl_err);

// Set as kernel argument
clSetKernelArg(kernel, argIndex, sizeof(cl_mem), &cl_lut);

// Cleanup
clReleaseMemObject(cl_lut);
```

> **Note**: Buffers created directly via `clCreateBuffer` bypass AE's memory tracking. Use this only for small auxiliary data, not for image buffers.

---

## Cross-Platform Considerations

### Platform Differences

| Aspect | NVIDIA (OpenCL) | AMD | Intel | Apple (deprecated) |
|--------|-----------------|-----|-------|-------------------|
| Max work group size | 1024 | 256 or 1024 | 512 | 512 or 1024 |
| Local memory size | 48 KB | 32 KB | 64 KB | 32 KB |
| Double precision | Usually supported | Varies | Varies | Varies |
| Image support | Yes | Yes | Yes | Yes |
| `-cl-fast-relaxed-math` | Works | May have bugs | Works | Works |

### Safe Work Group Size

Query the device to determine the maximum work group size instead of hardcoding:

```cpp
size_t max_wg_size;
clGetKernelWorkGroupInfo(kernel, device,
    CL_KERNEL_WORK_GROUP_SIZE, sizeof(size_t), &max_wg_size, NULL);

// Use the smaller of 256 and the device maximum
size_t local_x = 16;
size_t local_y = 16;
if (local_x * local_y > max_wg_size) {
    local_x = 16;
    local_y = max_wg_size / local_x;
}
```

### OpenCL Headers

Platform-specific include paths:

```cpp
#ifdef _WIN32
    #include <CL/cl.h>
#else  // macOS
    #include <OpenCL/cl.h>
#endif
```

### macOS Deprecation

Apple deprecated OpenCL on macOS in favor of Metal. The runtime still works as of macOS 14 (Sonoma) but shows compile warnings:

```
'OpenCL/cl.h' is deprecated: first deprecated in macOS 10.14
```

For new macOS development, Metal is preferred. For cross-platform plugins, OpenCL remains viable but test on current macOS versions to ensure continued compatibility.

---

## Kernel Source Management

### Embedding as C Strings

The SDK converts `.cl` files to C string headers at build time. You can do this manually or with a build script:

```python
# convert_cl_to_header.py
import sys

with open(sys.argv[1], 'r') as f:
    source = f.read()

name = sys.argv[2]
with open(sys.argv[3], 'w') as f:
    f.write(f'static const char* {name} = R"CL(\n')
    f.write(source)
    f.write('\n)CL";\n')
```

Or use C++11 raw string literals directly in a header:

```cpp
static const char* kMyKernel_OpenCLString = R"CL(
__kernel void MyEffectKernel(
    __global const float4 *src,
    __global       float4 *dst,
    int srcPitch,
    int dstPitch,
    int width,
    int height,
    float brightness)
{
    int x = get_global_id(0);
    int y = get_global_id(1);

    if (x >= width || y >= height) return;

    float4 pixel = src[y * srcPitch + x];
    pixel.xyz = clamp(pixel.xyz + brightness, 0.0f, 1.0f);
    dst[y * dstPitch + x] = pixel;
}
)CL";
```

### Caching Compiled Programs

For production plugins, cache the compiled binary to avoid recompilation on every device setup:

```cpp
// After successful build, extract binary
size_t binary_size;
clGetProgramInfo(program, CL_PROGRAM_BINARY_SIZES,
    sizeof(size_t), &binary_size, NULL);

std::vector<unsigned char> binary(binary_size);
unsigned char* binary_ptr = binary.data();
clGetProgramInfo(program, CL_PROGRAM_BINARIES,
    sizeof(unsigned char*), &binary_ptr, NULL);

// Save to disk...

// On subsequent loads, create from binary:
cl_int binary_status;
cl_program cached = clCreateProgramWithBinary(
    context, 1, &device,
    &binary_size, (const unsigned char**)&binary_ptr,
    &binary_status, &cl_err);
```

> **Warning**: OpenCL binaries are device-specific. A binary compiled for one GPU will not work on a different model. Include the device name in your cache key.

---

## Local (Shared) Memory

OpenCL's `__local` memory is equivalent to CUDA's `__shared__` memory:

```opencl
__kernel void TiledBlur(
    __global const float4 *src,
    __global       float4 *dst,
    __local        float4 *tile,   // allocated per work group
    int srcPitch,
    int dstPitch,
    int width,
    int height,
    int radius)
{
    int lx = get_local_id(0);
    int ly = get_local_id(1);
    int gx = get_global_id(0);
    int gy = get_global_id(1);
    int bw = get_local_size(0);

    int tile_w = bw + 2 * radius;

    // Load tile
    int sx = lx + radius;
    int sy = ly + radius;
    int cx = clamp(gx, 0, width - 1);
    int cy = clamp(gy, 0, height - 1);
    tile[sy * tile_w + sx] = src[cy * srcPitch + cx];

    // Load halo...

    barrier(CLK_LOCAL_MEM_FENCE);

    // Process from tile...
}
```

Set the local memory argument size when dispatching:

```cpp
int tile_w = 16 + 2 * radius;
int tile_h = 16 + 2 * radius;
size_t local_mem_size = tile_w * tile_h * sizeof(cl_float4);

clSetKernelArg(kernel, argIndex, local_mem_size, NULL);  // NULL pointer = local memory
```

---

## Common Pitfalls

1. **Global work size not a multiple of local work size** -- OpenCL requires this. Use `RoundUp` and add bounds checking in the kernel.

2. **Forgetting to release cl_kernel objects** -- Each `clCreateKernel` must have a matching `clReleaseKernel` in device setdown.

3. **Struct alignment differences** -- Host and device structs may be padded differently. Prefer individual arguments or use `__attribute__((packed))`.

4. **Using sizeof(cl_mem) wrong** -- `clSetKernelArg` for buffer arguments takes `sizeof(cl_mem)`, not the buffer size. For scalars, it takes the scalar size.

5. **Assuming NVIDIA behavior on AMD** -- Work group sizes, local memory limits, and optimization behavior differ. Always query device capabilities.

6. **Not checking build errors** -- `clBuildProgram` failure without checking the build log leaves you with no diagnostic information.

7. **Global work offset** -- The SDK always passes `NULL` for the global work offset parameter. Non-zero offsets are supported but rarely useful in AE plugins.

8. **macOS deprecation** -- OpenCL works on macOS today but is deprecated. Plan for Metal as the long-term macOS solution.

---

## See Also

- [GPU Memory Management](gpu-memory.md) -- `PF_GPUDeviceSuite1` reference, transfer patterns
- [CUDA Kernel Patterns](cuda-kernels.md) -- NVIDIA-specific GPU patterns
- [Metal Compute Shaders](metal-compute.md) -- macOS-native alternative to OpenCL
- [CPU/GPU Render Parity](cpu-gpu-parity.md) -- Pixel format differences and channel order
