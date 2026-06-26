# Metal Compute Shaders for AE GPU Effects

> Metal compute shader setup, pipeline creation, dispatch patterns, buffer handling, and threadgroup sizing for After Effects GPU plugins on macOS.

## Overview

On macOS, After Effects offers `PF_GPU_Framework_METAL` to GPU-capable plugins. AE provides an `MTLDevice` and `MTLCommandQueue` through `PF_GPUDeviceInfo`. Your plugin compiles Metal shaders at device setup time, then dispatches compute work during `PF_Cmd_SMART_RENDER_GPU`.

The GPU pixel format is `PF_PixelFormat_GPU_BGRA128` -- four 32-bit floats per pixel in **BGRA** order. Metal buffers (`id<MTLBuffer>`) hold the pixel data; you bind them to compute encoders and dispatch threadgroups.

---

## Device Setup: Compiling Metal Shaders

Metal shaders must be compiled at runtime during `PF_Cmd_GPU_DEVICE_SETUP`. Unlike CUDA (statically linked), Metal requires creating a library from source, extracting kernel functions, and building pipeline state objects.

### Embedding Shader Source

The SDK sample embeds the Metal shader as a C string in a header file (`SDK_Invert_ProcAmp_Kernel.metal.h`). This avoids file I/O at runtime:

```cpp
// Generated at build time from your .metal file:
// const char* kMyKernel_MetalString = "...shader source...";
```

Alternatively, load from the plugin bundle at runtime:

```objc
NSString *path = [[NSBundle bundleForClass:[YourPluginClass class]]
                    pathForResource:@"MyKernel" ofType:@"metal"];
NSString *source = [NSString stringWithContentsOfFile:path
                             encoding:NSUTF8StringEncoding error:&err];
```

> **Warning**: `[device newDefaultLibrary]` does NOT work for AE plugins. It searches the host application bundle (After Effects), not your plugin. Always compile from source or load from your plugin's bundle path.

### Full Device Setup Pattern

```objc
struct MetalGPUData {
    id<MTLComputePipelineState> myEffectPipeline;
};

static PF_Err GPUDeviceSetup(
    PF_InData *in_dataP,
    PF_OutData *out_dataP,
    PF_GPUDeviceSetupExtra *extraP)
{
    PF_Err err = PF_Err_NONE;

    if (extraP->input->what_gpu != PF_GPU_Framework_METAL)
        return err;

    ScopedAutoreleasePool pool;  // REQUIRED -- see below

    // Get the Metal device from AE
    PF_GPUDeviceInfo device_info;
    AEFX_CLR_STRUCT(device_info);

    AEFX_SuiteScoper<PF_GPUDeviceSuite1> gpuSuite =
        AEFX_SuiteScoper<PF_GPUDeviceSuite1>(in_dataP,
            kPFGPUDeviceSuite, kPFGPUDeviceSuiteVersion1, out_dataP);

    gpuSuite->GetDeviceInfo(in_dataP->effect_ref,
        extraP->input->device_index, &device_info);

    id<MTLDevice> device = (id<MTLDevice>)device_info.devicePV;

    // Compile shader from embedded source string
    NSString *source = [NSString stringWithCString:kMyKernel_MetalString
                                 encoding:NSUTF8StringEncoding];
    NSError *compileError = nil;
    id<MTLLibrary> library = [[device newLibraryWithSource:source
                                       options:nil
                                       error:&compileError] autorelease];

    if (!library) {
        // compileError.localizedDescription has details
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    // Extract the kernel function
    id<MTLFunction> function = [[library newFunctionWithName:@"MyEffectKernel"]
                                 autorelease];
    if (!function) {
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    // Create pipeline state
    AEFX_SuiteScoper<PF_HandleSuite1> handleSuite =
        AEFX_SuiteScoper<PF_HandleSuite1>(in_dataP,
            kPFHandleSuite, kPFHandleSuiteVersion1, out_dataP);

    PF_Handle dataH = handleSuite->host_new_handle(sizeof(MetalGPUData));
    MetalGPUData *metalData = reinterpret_cast<MetalGPUData*>(*dataH);

    NSError *pipelineError = nil;
    metalData->myEffectPipeline =
        [device newComputePipelineStateWithFunction:function
                                              error:&pipelineError];

    if (!metalData->myEffectPipeline) {
        handleSuite->host_dispose_handle(dataH);
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    // Store for use during render
    extraP->output->gpu_data = dataH;
    out_dataP->out_flags2 = PF_OutFlag2_SUPPORTS_GPU_RENDER_F32;

    return err;
}
```

### Device Setdown

Metal pipeline state objects are reference-counted Objective-C objects. Release them during setdown:

```objc
static PF_Err GPUDeviceSetdown(
    PF_InData *in_dataP,
    PF_OutData *out_dataP,
    PF_GPUDeviceSetdownExtra *extraP)
{
    if (extraP->input->what_gpu == PF_GPU_Framework_METAL) {
        PF_Handle dataH = (PF_Handle)extraP->input->gpu_data;
        MetalGPUData *metalData = reinterpret_cast<MetalGPUData*>(*dataH);

        [metalData->myEffectPipeline release];

        AEFX_SuiteScoper<PF_HandleSuite1> handleSuite =
            AEFX_SuiteScoper<PF_HandleSuite1>(in_dataP,
                kPFHandleSuite, kPFHandleSuiteVersion1, out_dataP);
        handleSuite->host_dispose_handle(dataH);
    }
    return PF_Err_NONE;
}
```

---

## Autorelease Pool: Mandatory

AE plugins cannot rely on a host autorelease pool. Any Objective-C or Metal call that internally autoreleases objects will leak without one. Wrap every Metal code path:

```objc
struct ScopedAutoreleasePool {
    ScopedAutoreleasePool() : mPool([[NSAutoreleasePool alloc] init]) {}
    ~ScopedAutoreleasePool() { [mPool release]; }
    NSAutoreleasePool *mPool;
};
```

Use it at the top of `GPUDeviceSetup`, `GPUDeviceSetdown`, and `SmartRenderGPU`:

```objc
ScopedAutoreleasePool pool;
// ... all Metal calls inside this scope ...
```

> **Pitfall**: Omitting the autorelease pool causes slow memory leaks that accumulate over many renders. AE will gradually consume more RAM until it runs out. This is very difficult to debug because the leak is in Cocoa internals, not your code.

---

## Writing Metal Compute Shaders

### Basic Kernel

```metal
#include <metal_stdlib>
using namespace metal;

struct EffectParams {
    int   srcPitch;
    int   dstPitch;
    int   is16f;
    int   width;
    int   height;
    float brightness;
    float contrast;
};

kernel void MyEffectKernel(
    device const float4 *src      [[buffer(0)]],
    device       float4 *dst      [[buffer(1)]],
    device const EffectParams &p  [[buffer(2)]],
    uint2 gid                     [[thread_position_in_grid]])
{
    if (gid.x >= (uint)p.width || gid.y >= (uint)p.height)
        return;

    float4 pixel = src[gid.y * p.srcPitch + gid.x];

    // BGRA order: .x=Blue, .y=Green, .z=Red, .w=Alpha
    float luma = pixel.z * 0.299 + pixel.y * 0.587 + pixel.x * 0.114;
    luma = (p.contrast * luma) + p.brightness;

    pixel.x = luma;  // B
    pixel.y = luma;  // G
    pixel.z = luma;  // R
    // Alpha unchanged

    dst[gid.y * p.dstPitch + gid.x] = pixel;
}
```

### Channel Order on Metal

Metal GPU buffers in AE use the same `PF_PixelFormat_GPU_BGRA128` as CUDA:

| `float4` component | Channel |
|--------------------|---------|
| `.x` (or `.r` in Metal swizzle) | **Blue** |
| `.y` (or `.g`) | **Green** |
| `.z` (or `.b`) | **Red** |
| `.w` (or `.a`) | **Alpha** |

> **Warning**: Metal's `.r`, `.g`, `.b`, `.a` swizzle accessors are syntactic sugar for `.x`, `.y`, `.z`, `.w`. They do NOT correspond to actual Red/Green/Blue/Alpha channels in AE's BGRA layout. Using `pixel.r` to mean "red" is wrong -- it accesses `.x` which is Blue. Either use `.x/.y/.z/.w` consistently, or define named accessors:

```metal
#define BLUE(p)  (p.x)
#define GREEN(p) (p.y)
#define RED(p)   (p.z)
#define ALPHA(p) (p.w)
```

---

## Dispatch Patterns

### Single Encoder, Single Command Buffer

The correct pattern for AE Metal rendering: one command buffer, one compute encoder, all dispatches through that encoder.

```objc
static PF_Err SmartRenderGPU_Metal(
    PF_InData *in_dataP,
    PF_OutData *out_dataP,
    PF_EffectWorld *input_worldP,
    PF_EffectWorld *output_worldP,
    PF_SmartRenderExtra *extraP,
    EffectParams *params)
{
    ScopedAutoreleasePool pool;

    PF_GPUDeviceInfo device_info;
    AEFX_SuiteScoper<PF_GPUDeviceSuite1> gpuSuite =
        AEFX_SuiteScoper<PF_GPUDeviceSuite1>(in_dataP,
            kPFGPUDeviceSuite, kPFGPUDeviceSuiteVersion1, out_dataP);
    gpuSuite->GetDeviceInfo(in_dataP->effect_ref,
        extraP->input->device_index, &device_info);

    // Retrieve stored pipeline state
    Handle metal_handle = (Handle)extraP->input->gpu_data;
    MetalGPUData *metalData = reinterpret_cast<MetalGPUData*>(*metal_handle);

    // Get GPU buffer pointers (these are id<MTLBuffer> cast to void*)
    void *src_mem, *dst_mem;
    gpuSuite->GetGPUWorldData(in_dataP->effect_ref, input_worldP, &src_mem);
    gpuSuite->GetGPUWorldData(in_dataP->effect_ref, output_worldP, &dst_mem);

    id<MTLBuffer> srcBuffer = (id<MTLBuffer>)src_mem;
    id<MTLBuffer> dstBuffer = (id<MTLBuffer>)dst_mem;

    // Create parameter buffer
    id<MTLDevice> device = (id<MTLDevice>)device_info.devicePV;
    id<MTLBuffer> paramBuffer = [[device newBufferWithBytes:params
                                         length:sizeof(EffectParams)
                                         options:MTLResourceStorageModeManaged]
                                  autorelease];

    // Create command buffer and encoder
    id<MTLCommandQueue> queue = (id<MTLCommandQueue>)device_info.command_queuePV;
    id<MTLCommandBuffer> cmdBuf = [queue commandBuffer];
    id<MTLComputeCommandEncoder> encoder = [cmdBuf computeCommandEncoder];

    // Dispatch
    [encoder setComputePipelineState:metalData->myEffectPipeline];
    [encoder setBuffer:srcBuffer   offset:0 atIndex:0];
    [encoder setBuffer:dstBuffer   offset:0 atIndex:1];
    [encoder setBuffer:paramBuffer offset:0 atIndex:2];

    MTLSize threadsPerGroup = {
        [metalData->myEffectPipeline threadExecutionWidth],
        16,
        1
    };
    MTLSize numThreadgroups = {
        (params->width  + threadsPerGroup.width  - 1) / threadsPerGroup.width,
        (params->height + threadsPerGroup.height - 1) / threadsPerGroup.height,
        1
    };

    [encoder dispatchThreadgroups:numThreadgroups
            threadsPerThreadgroup:threadsPerGroup];

    [encoder endEncoding];
    [cmdBuf commit];
    [cmdBuf waitUntilCompleted];

    NSError *cmdError = [cmdBuf error];
    if (cmdError) {
        return PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    return PF_Err_NONE;
}
```

### Multi-Pass Dispatch

For multi-pass effects, dispatch multiple kernels through the **same encoder**:

```objc
// Pass 1: Invert
[encoder setComputePipelineState:metalData->invertPipeline];
[encoder setBuffer:srcBuffer   offset:0 atIndex:0];
[encoder setBuffer:tmpBuffer   offset:0 atIndex:1];
[encoder setBuffer:invertParams offset:0 atIndex:2];
[encoder dispatchThreadgroups:groups1 threadsPerThreadgroup:tpg1];

// Pass 2: Color correction (reads from tmpBuffer)
[encoder setComputePipelineState:metalData->colorPipeline];
[encoder setBuffer:tmpBuffer   offset:0 atIndex:0];
[encoder setBuffer:dstBuffer   offset:0 atIndex:1];
[encoder setBuffer:colorParams offset:0 atIndex:2];
[encoder dispatchThreadgroups:groups2 threadsPerThreadgroup:tpg2];

// Finish
[encoder endEncoding];
[cmdBuf commit];
[cmdBuf waitUntilCompleted];
```

Metal guarantees that dispatches within a single compute command encoder execute in order. No explicit barriers are needed between passes on the same encoder.

> **Warning**: Do NOT create multiple command buffers or multiple encoders within a single render call. This causes synchronization issues and can crash AE.

---

## Threadgroup Sizing

### Choosing Threadgroup Dimensions

The pipeline's `threadExecutionWidth` property returns the GPU's SIMD width (typically 32 on Apple GPUs). The SDK sample uses this as the X dimension:

```objc
MTLSize threadsPerGroup = {
    [pipeline threadExecutionWidth],  // 32 on Apple Silicon
    16,
    1
};
```

This gives 32 x 16 = 512 threads per threadgroup, which is a good default.

### Grid Size Calculation

Use ceiling division to cover the entire image:

```objc
MTLSize numThreadgroups = {
    (width  + threadsPerGroup.width  - 1) / threadsPerGroup.width,
    (height + threadsPerGroup.height - 1) / threadsPerGroup.height,
    1
};
```

### `dispatchThreadgroups` vs `dispatchThreads`

| Method | Behavior |
|--------|----------|
| `dispatchThreadgroups:threadsPerThreadgroup:` | Launches full threadgroups. Threads beyond image bounds must check and early-return. **Use this.** |
| `dispatchThreads:threadsPerThreadgroup:` | Automatically clips to exact thread count. Requires non-uniform threadgroup support. Not recommended for maximum compatibility. |

The SDK sample uses `dispatchThreadgroups`. Always include bounds checking in your kernel:

```metal
if (gid.x >= (uint)p.width || gid.y >= (uint)p.height)
    return;
```

---

## Buffer vs Texture Approaches

### Buffers (Default for AE)

AE provides GPU data as `id<MTLBuffer>` objects. This is the standard approach and what `GetGPUWorldData` returns.

Advantages:
- Direct linear access with pitch indexing
- Simple parameter passing
- Matches CUDA/OpenCL buffer model

### Textures (Advanced)

You can create an `MTLTexture` view over a buffer for texture sampling features (bilinear filtering, clamping):

```metal
// Host side:
MTLTextureDescriptor *texDesc = [MTLTextureDescriptor
    texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA32Float
    width:width height:height mipmapped:NO];

id<MTLTexture> srcTexture = [srcBuffer
    newTextureWithDescriptor:texDesc
    offset:0
    bytesPerRow:srcRowBytes];
```

> **Warning**: When creating a texture view over an AE buffer, the pixel format `MTLPixelFormatRGBA32Float` treats the four floats as RGBA. But AE stores them as BGRA. Your shader must swizzle when reading from the texture, or accept that `.r` = Blue, `.g` = Green, `.b` = Red, `.a` = Alpha. This is confusing; buffers with explicit indexing are usually cleaner.

---

## Passing Parameters via Metal Buffers

Metal kernels receive parameters through buffer bindings. The SDK approach: pack parameters into a struct, create an `MTLBuffer`, and bind it:

```objc
typedef struct {
    int   srcPitch;
    int   dstPitch;
    int   is16f;
    int   width;
    int   height;
    float brightness;
    float contrast;
    float hueCosSat;
    float hueSinSat;
} MyParams;

MyParams params = { /* fill in values */ };

id<MTLBuffer> paramBuf = [[device newBufferWithBytes:&params
                                    length:sizeof(MyParams)
                                    options:MTLResourceStorageModeManaged]
                           autorelease];

[encoder setBuffer:paramBuf offset:0 atIndex:2];
```

The struct layout in the Metal shader must exactly match the C/C++ struct layout. Use explicit padding if necessary:

```metal
struct MyParams {
    int   srcPitch;
    int   dstPitch;
    int   is16f;
    int   width;
    int   height;
    float brightness;
    float contrast;
    float hueCosSat;
    float hueSinSat;
};
```

> **Pitfall**: C++ struct alignment and Metal struct alignment can differ. Use `packed` attribute or manually verify that `sizeof(MyParams)` matches between both sides. Misalignment causes incorrect parameter values in the shader.

---

## Threadgroup Memory (Metal Shared Memory)

Metal's equivalent of CUDA shared memory is `threadgroup` memory:

```metal
kernel void TiledBlurKernel(
    device const float4 *src       [[buffer(0)]],
    device       float4 *dst       [[buffer(1)]],
    device const BlurParams &p     [[buffer(2)]],
    uint2 gid                      [[thread_position_in_grid]],
    uint2 tid                      [[thread_position_in_threadgroup]],
    uint2 tgSize                   [[threads_per_threadgroup]])
{
    // Declare threadgroup memory
    threadgroup float4 tile[32 + 4][16 + 4];  // block + halo

    // Load tile from global memory
    int gx = min(int(gid.x), p.width - 1);
    int gy = min(int(gid.y), p.height - 1);
    tile[tid.x + 2][tid.y + 2] = src[gy * p.srcPitch + gx];

    // Load halo...
    // ...

    threadgroup_barrier(mem_flags::mem_threadgroup);

    // Read from tile for blur computation
    // ...
}
```

### Threadgroup Memory Limits

| Apple GPU | Max Threadgroup Memory |
|-----------|----------------------|
| Apple M1/M2 | 32 KB |
| Apple M1 Pro/Max/Ultra | 32 KB |
| Apple M3 | 32 KB |

For a 32x16 threadgroup processing float4 pixels with a blur radius of 4, the tile is `(32+8) * (16+8) * 16 bytes = 15,360 bytes` -- well within limits.

---

## CMake Integration

### Compiling .metal Files into the Plugin Bundle

Metal shaders can be embedded as strings or compiled at runtime from source files in the bundle:

```cmake
# Copy .metal source files to the plugin bundle's Resources
set_source_files_properties(
    ${CMAKE_CURRENT_SOURCE_DIR}/gpu/MyKernel.metal
    PROPERTIES MACOSX_PACKAGE_LOCATION "Resources"
)

target_sources(MyEffect PRIVATE
    gpu/MyKernel.metal
)
```

### Precompiling to .metallib (Optional)

For production plugins, precompile to a `.metallib` for faster load times:

```cmake
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/MyKernel.metallib
    COMMAND xcrun -sdk macosx metal
            -o ${CMAKE_CURRENT_BINARY_DIR}/MyKernel.air
            ${CMAKE_CURRENT_SOURCE_DIR}/gpu/MyKernel.metal
    COMMAND xcrun -sdk macosx metallib
            -o ${CMAKE_CURRENT_BINARY_DIR}/MyKernel.metallib
            ${CMAKE_CURRENT_BINARY_DIR}/MyKernel.air
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/gpu/MyKernel.metal
)
```

Then load at runtime:

```objc
NSString *libPath = /* path to MyKernel.metallib in bundle */;
NSURL *url = [NSURL fileURLWithPath:libPath];
id<MTLLibrary> library = [device newLibraryWithURL:url error:&error];
```

---

## Error Handling

### Shader Compilation Errors

Metal returns errors through `NSError` objects. The SDK sample checks `library == nil` rather than `error != nil`, because Metal sets an error code for compile **warnings** too:

```objc
id<MTLLibrary> library = [[device newLibraryWithSource:source
                                   options:nil error:&error] autorelease];

// An error code is set for Metal compile warnings, so use nil library
// as the error signal, not a non-nil error object
if (!library) {
    NSString *msg = error.localizedDescription;  // Detailed error text
    return PF_Err_INTERNAL_STRUCT_DAMAGED;
}
```

### Command Buffer Errors

Check after `waitUntilCompleted`:

```objc
[cmdBuf commit];
[cmdBuf waitUntilCompleted];

if ([cmdBuf error]) {
    // [cmdBuf error].localizedDescription has details
    return PF_Err_INTERNAL_STRUCT_DAMAGED;
}
```

> **Note**: The SDK sample checks `[commandBuffer error]` before `waitUntilCompleted` in one place, which may return nil since the buffer has not finished. Always check after `waitUntilCompleted` for reliable error detection.

---

## Common Pitfalls

1. **Missing autorelease pool** -- Causes memory leaks that accumulate over renders. Every Metal code path needs `ScopedAutoreleasePool`.

2. **Using `newDefaultLibrary`** -- Searches the AE host bundle, not your plugin. Always compile from source or load from your plugin bundle.

3. **BGRA vs RGBA confusion** -- Metal's `float4` `.r/.g/.b/.a` accessors map to `.x/.y/.z/.w`, which is Blue/Green/Red/Alpha in AE buffers. Use `.x/.y/.z/.w` for clarity.

4. **Multiple command buffers** -- Create one command buffer and one encoder per render call. Multiple buffers cause synchronization failures.

5. **Forgetting `endEncoding`** -- The encoder must be ended before committing the command buffer. Omitting this crashes.

6. **Struct alignment mismatch** -- Ensure your parameter struct has identical layout in C++ and Metal. Add explicit padding if needed.

7. **Not calling `waitUntilCompleted`** -- Without this, AE may read the output buffer before the GPU finishes writing to it, producing torn or incomplete frames.

8. **Storage mode mismatch** -- Parameter buffers should use `MTLResourceStorageModeManaged` (or `Shared` on Apple Silicon). Using `Private` for host-created data will not work.

---

## See Also

- [GPU Memory Management](gpu-memory.md) -- Transfer patterns and suite reference
- [CPU/GPU Render Parity](cpu-gpu-parity.md) -- BGRA vs ARGB details, shader loading
- [CUDA Kernels](cuda-kernels.md) -- Equivalent patterns for NVIDIA GPUs
- [OpenCL Setup](opencl-setup.md) -- Cross-platform alternative
