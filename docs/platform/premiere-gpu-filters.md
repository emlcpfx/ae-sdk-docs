# Premiere Pro GPU Filters: PrSDKGPUFilter and Kernel Framework

Premiere Pro provides a dedicated GPU filter pipeline that is architecturally distinct from After Effects' SmartFX GPU rendering. Where AE's GPU path is driven through `PF_Cmd_GPU_RENDER` selectors within the standard effect entry point, Premiere's GPU filters use a completely separate entry point (`xGPUFilterEntry`) and a self-contained lifecycle built around `PrGPUFilter` callbacks. This guide covers the full pipeline from entry point registration through cross-platform kernel authoring.

**SDK Headers Covered:**
- `PrSDKGPUFilter.h` -- GPU filter entry points, callbacks, and data structures
- `PrSDKGPUDeviceSuite.h` -- Device enumeration, memory allocation, GPU PPix creation
- `PrSDKGPUImageProcessingSuite.h` -- Host-provided GPU image operations
- `PrGPU/KernelSupport/KernelCore.h` -- Cross-platform kernel macros and abstractions
- `PrGPU/KernelSupport/KernelWrapper.h` -- Kernel function declaration via Boost.Preprocessor
- `PrGPU/KernelSupport/KernelMemory.h` -- 16f/32f pixel read/write helpers
- `PrGPU/KernelSupport/KernelHalf.h` -- Half-float conversion utilities

---

## Architecture Overview

### How Premiere's GPU Pipeline Differs from AE's

| Aspect | After Effects | Premiere Pro |
|--------|---------------|--------------|
| Entry point | Same `EffectMain()` with `PF_Cmd_GPU_RENDER` | Separate `xGPUFilterEntry` export |
| Instance model | One global effect instance | Per-parameter-set instances (`PrGPUFilterInstance`) |
| GPU frameworks | Metal, OpenCL (via AE's own abstraction) | CUDA, OpenCL, Metal, DirectX |
| Pixel formats | `PF_PixelFormat_GPU_*` | `PrPixelFormat_GPU_BGRA_4444_16f` / `_32f` |
| Render phases | PreRender + SmartRender | GetFrameDependencies + optional Precompute + Render |
| Parameter access | `PF_ParamDef` via checkout | `PrSDKVideoSegmentSuite::GetParam` using node ID |
| Fallback | Returns error from GPU_RENDER, host falls back to CPU | Returns error from `CreateInstance`, host uses software filter |
| Memory management | Host-managed | Plugin allocates via `PrSDKGPUDeviceSuite` |

The key conceptual difference: in Premiere, a GPU filter instance is created for a specific set of parameters on a specific track item. When parameters change, the old instance is disposed and a new one is created. This means you cannot rely on persistent state across parameter changes.

### GPU Pixel Formats

Premiere GPU frames are always BGRA with origin at top-left:

```c
// From PrSDKGPUDeviceSuite.h
#define PrPixelFormat_GPU_BGRA_4444_16f  MAKE_ADOBE_PRIVATE_PIXEL_FORMAT_FOURCC('C', 'D', 'a')
#define PrPixelFormat_GPU_BGRA_4444_32f  MAKE_ADOBE_PRIVATE_PIXEL_FORMAT_FOURCC('C', 'D', 'A')
```

The `16f` variant uses half-precision floats (8 bytes per pixel); the `32f` variant uses single-precision (16 bytes per pixel). Your kernel code must handle both -- the `KernelMemory.h` helpers make this straightforward.

---

## The GPU Filter Entry Point

### xGPUFilterEntry

Every GPU filter module exports a single C function:

```c
// From PrSDKGPUFilter.h
#define PrGPUFilterEntryPointName "xGPUFilterEntry"

typedef prSuiteError (*PrGPUFilterEntry)(
    csSDK_uint32   inHostInterfaceVersion,  // Current host version
    csSDK_int32*   ioIndex,                 // Increment for multiple filters
    prBool         inStartup,               // 1 = startup, 0 = shutdown
    piSuitesPtr    piSuites,
    PrGPUFilter*   outFilter,               // Fill in callback pointers
    PrGPUFilterInfo* outFilterInfo);         // Fill in match name + version
```

During startup (`inStartup == 1`), you populate the `PrGPUFilter` callback table and the `PrGPUFilterInfo` structure. During shutdown (`inStartup == 0`), you clean up any global resources.

### PrGPUFilterInfo

```c
typedef struct {
    csSDK_uint32 outInterfaceVersion;
    PrSDKString  outMatchName;  // Must match a registered software filter's match name
} PrGPUFilterInfo;
```

The `outMatchName` ties this GPU implementation to its corresponding software (CPU) effect. If `outMatchName` is null, it defaults to the module's PiPL. Set `outInterfaceVersion` to `PrSDKGPUFilterInterfaceVersion` (currently version 2, introduced in CC 9.0).

### Supporting Multiple Filters

To register multiple GPU filters from one module, increment `*ioIndex` during startup. The host will call the entry point repeatedly until you stop incrementing:

```cpp
prSuiteError xGPUFilterEntry(
    csSDK_uint32 inHostInterfaceVersion,
    csSDK_int32* ioIndex,
    prBool inStartup,
    piSuitesPtr piSuites,
    PrGPUFilter* outFilter,
    PrGPUFilterInfo* outFilterInfo)
{
    if (inStartup)
    {
        csSDK_int32 index = *ioIndex;
        if (index == 0)
        {
            // Register first filter
            outFilterInfo->outMatchName = /* ... */;
            *ioIndex = 1;  // Signal there is another
        }
        else if (index == 1)
        {
            // Register second filter (do NOT increment ioIndex further)
            outFilterInfo->outMatchName = /* ... */;
        }

        outFilterInfo->outInterfaceVersion = PrSDKGPUFilterInterfaceVersion;
        outFilter->CreateInstance = MyCreateInstance;
        outFilter->DisposeInstance = MyDisposeInstance;
        outFilter->GetFrameDependencies = MyGetFrameDependencies;
        outFilter->Precompute = MyPrecompute;
        outFilter->Render = MyRender;

        return suiteError_NoError;
    }
    else
    {
        return suiteError_NoError;  // Shutdown
    }
}
```

---

## The PrGPUFilter Callback Table

The `PrGPUFilter` struct defines the five callbacks that drive a GPU filter's lifecycle:

```c
typedef struct {
    prSuiteError (*CreateInstance)(PrGPUFilterInstance* ioInstanceData);
    prSuiteError (*DisposeInstance)(PrGPUFilterInstance* ioInstanceData);

    prSuiteError (*GetFrameDependencies)(
        PrGPUFilterInstance* inInstanceData,
        const PrGPUFilterRenderParams* inRenderParams,
        csSDK_int32* ioQueryIndex,
        PrGPUFilterFrameDependency* outFrameDependencies);

    prSuiteError (*Precompute)(
        PrGPUFilterInstance* inInstanceData,
        const PrGPUFilterRenderParams* inRenderParams,
        csSDK_int32 inIndex,
        PPixHand inFrame);

    prSuiteError (*Render)(
        PrGPUFilterInstance* inInstanceData,
        const PrGPUFilterRenderParams* inRenderParams,
        const PPixHand* inFrames,
        csSDK_size_t inFrameCount,
        PPixHand* outFrame);
} PrGPUFilter;
```

### CreateInstance / DisposeInstance

`CreateInstance` is called when Premiere needs a GPU rendering instance for an effect on a track item. The `PrGPUFilterInstance` provides:

```c
typedef struct {
    piSuitesPtr    piSuites;           // Standard suites
    csSDK_uint32   inDeviceIndex;      // For PrSDKGPUDeviceSuite
    PrTimelineID   inTimelineID;       // For PrSDKVideoSegmentSuite
    csSDK_int32    inNodeID;           // Node in the segment tree
    void*          ioPrivatePluginData; // Your instance data pointer
    prBool         outIsRealtime;      // Affects timeline segment coloring
} PrGPUFilterInstance;
```

**Returning an error from `CreateInstance` causes the host to fall back to software rendering** for this node with its current parameters. This is the correct way to decline GPU rendering (for example, if the current GPU lacks a required feature).

Multiple instances may be called concurrently -- your implementation must be thread-safe.

### GetFrameDependencies

Called before rendering to determine what inputs the filter needs. The host calls this repeatedly, incrementing `*ioQueryIndex` each time, until the plugin stops modifying it. Return `suiteError_NotImplemented` if you only need the current frame.

Dependency types (from `PrGPUFilterFrameDependencyType`):

| Flag | Purpose |
|------|---------|
| `PrGPUDependency_InputFrame` (0x1) | Request a frame from a specific track/time |
| `PrGPUDependency_Precompute` (0x2) | Request a CPU precomputation phase |
| `PrGPUDependency_FieldSeparation` (0x4) | Control field separation behavior |
| `PrGPUDependency_TransitionInputFrame` (0x8) | Request transition input frames |

For `PrGPUDependency_InputFrame`, set `outTrackID` (0 = current track) and `outSequenceTime`. For `PrGPUDependency_Precompute`, specify the precompute frame format and dimensions -- the host will allocate pinned host memory and call your `Precompute` callback, then upload the result to the GPU.

### Precompute

Only called if you returned `PrGPUDependency_Precompute` from `GetFrameDependencies`. Runs on the CPU to fill a preallocated host-memory buffer (the `PPixHand`). May be called ahead of actual render time. Results are uploaded to GPU memory by the host. If the precompute pixel format is not custom, frames are automatically converted to the GPU pixel format.

### Render

The main GPU rendering callback. Input frames arrive as an array of `PPixHand` handles:
- For effects: `inFrames[0]` is always the frame at the current time. Additional frames follow in the order returned from `GetFrameDependencies`.
- For transitions: `inFrames[0]` is the incoming frame, `inFrames[1]` is the outgoing frame. Transitions may not have other frame dependencies.

The result must be in the same pixel format as the input. You can either:
1. Allocate a new output GPU PPix via `PrSDKGPUDeviceSuite::CreateGPUPPix` and write to it
2. Render in-place by setting `*outFrame = inFrames[0]`

### PrGPUFilterRenderParams

```c
typedef struct {
    PrTime          inClipTime;
    PrTime          inSequenceTime;
    PrRenderQuality inQuality;
    float           inDownsampleFactorX;
    float           inDownsampleFactorY;
    csSDK_uint32    inRenderWidth;
    csSDK_uint32    inRenderHeight;
    csSDK_uint32    inRenderPARNum;
    csSDK_uint32    inRenderPARDen;
    prFieldType     inRenderFieldType;
    PrTime          inRenderTicksPerFrame;
    pmFieldDisplay  inRenderField;  // Which field is being rendered
} PrGPUFilterRenderParams;
```

GPU rendering always operates on full-height progressive frames unless `outNeedsFieldSeparation` was set to false via `PrGPUDependency_FieldSeparation`. The `inRenderField` indicates which field is being rendered when field separation is active.

---

## PrSDKGPUDeviceSuite

Suite name: `"MediaCore GPU Device Suite"` (version 2)

This suite is your interface to GPU hardware. All device memory and GPU PPix allocation must go through it.

### Device Enumeration

```c
prSuiteError (*GetDeviceCount)(csSDK_uint32* outDeviceCount);
prSuiteError (*GetDeviceInfo)(
    csSDK_uint32 inSuiteVersion,
    csSDK_uint32 inDeviceIndex,
    PrGPUDeviceInfo* outDeviceInfo);
```

The `PrGPUDeviceInfo` structure reveals which GPU framework is in use:

```c
typedef enum {
    PrGPUDeviceFramework_CUDA,
    PrGPUDeviceFramework_OpenCL,
    PrGPUDeviceFramework_Metal,
    PrGPUDeviceFramework_DirectX,
} PrGPUDeviceFramework;

typedef struct {
    PrGPUDeviceFramework outDeviceFramework;
    prBool               outMeetsMinimumRequirementsForAcceleration;
    void*                outPlatformHandle;      // cl_platform_id
    void*                outDeviceHandle;         // CUdevice / cl_device_id / ID3D12Device*
    void*                outContextHandle;        // CUcontext / cl_context
    void*                outCommandQueueHandle;   // CUstream / cl_command_queue / ID3D12CommandQueue*
    void*                outOffscreenOpenGLContextHandle;  // Primary device only
    void*                outOffscreenOpenGLDeviceHandle;   // Primary device only
} PrGPUDeviceInfo;
```

CUDA versions: Windows uses CUDA 11080, macOS uses CUDA 9020. OpenCL version is 1000.

### Exclusive Device Access

For full GPU filter plugins (those using `xGPUFilterEntry`), exclusive device access is always held -- you do not need to call these. They exist for other plugin types that may need GPU access:

```c
prSuiteError (*AcquireExclusiveDeviceAccess)(csSDK_uint32 inDeviceIndex);
prSuiteError (*ReleaseExclusiveDeviceAccess)(csSDK_uint32 inDeviceIndex);
```

### Memory Allocation

All GPU memory must be allocated through this suite, not through raw CUDA/OpenCL/Metal calls:

```c
// Device (GPU) memory
prSuiteError (*AllocateDeviceMemory)(csSDK_uint32 inDeviceIndex, size_t inSizeInBytes, void** outMemory);
prSuiteError (*FreeDeviceMemory)(csSDK_uint32 inDeviceIndex, void* inMemory);
prSuiteError (*PurgeDeviceMemory)(csSDK_uint32 inDeviceIndex, size_t inRequestedBytesToPurge, size_t* outBytesPurged);

// Host (pinned) memory
prSuiteError (*AllocateHostMemory)(csSDK_uint32 inDeviceIndex, size_t inSizeInBytes, void** outMemory);
prSuiteError (*FreeHostMemory)(csSDK_uint32 inDeviceIndex, void* inMemory);
prSuiteError (*PurgeHostMemory)(csSDK_uint32 inDeviceIndex, size_t inRequestedBytesToPurge, size_t* outBytesPurged);
```

For DirectX, device memory returns `ID3D12Resource*` on the Default Heap; host memory returns `ID3D12Resource*` on the Upload Heap. `PurgeDeviceMemory` and `PurgeHostMemory` should only be called in emergency situations when working with GPU memory that cannot be allocated through this suite (e.g., OpenGL memory).

### GPU PPix Operations

```c
prSuiteError (*CreateGPUPPix)(
    csSDK_uint32 inDeviceIndex, PrPixelFormat inPixelFormat,
    int inWidth, int inHeight,
    int inPARNumerator, int inPARDenominator,
    prFieldType inFieldType,
    PPixHand* outPPixHand);

prSuiteError (*GetGPUPPixData)(PPixHand inPPixHand, void** outData);
prSuiteError (*GetGPUPPixDeviceIndex)(PPixHand inPPixHand, csSDK_uint32* outDeviceIndex);
prSuiteError (*GetGPUPPixSize)(PPixHand inPPixHand, size_t* outSize);
```

Standard `PrSDKPPixSuite` functions also work on GPU PPix handles: `Dispose`, `GetBounds`, `GetRowBytes`, `GetPixelAspectRatio`, `GetPixelFormat`, and `PrSDKPPix2Suite::GetFieldOrder`.

---

## PrSDKGPUImageProcessingSuite

Suite name: `"MediaCore GPU Image Processing Suite"` (version 1)

Provides host-implemented GPU image operations. All functions require the pixel format to be one of the GPU formats (`_16f` or `_32f`).

### PixelFormatConvert

Converts between host (CPU) pixel formats and GPU formats. One of the source/dest formats must be a host format, the other must be a GPU format:

```c
prSuiteError (*PixelFormatConvert)(
    csSDK_uint32 inDeviceIndex,
    const void* inSrc, csSDK_int32 inSrcRowBytes, PrPixelFormat inSrcPixelFormat,
    void* inDest, csSDK_int32 inDestRowBytes, PrPixelFormat inDestPixelFormat,
    csSDK_uint32 inWidth, csSDK_uint32 inHeight,
    PrRenderQuality inQuality);
```

### Scale

```c
prSuiteError (*Scale)(
    csSDK_uint32 inDeviceIndex,
    const void* inSrc, csSDK_int32 inSrcRowBytes,
    csSDK_uint32 inSrcWidth, csSDK_uint32 inSrcHeight,
    void* inDest, csSDK_int32 inDestRowBytes,
    csSDK_uint32 inDestWidth, csSDK_uint32 inDestHeight,
    PrPixelFormat inPixelFormat,
    float inScaleX, float inScaleY,
    PrRenderQuality inQuality);
```

### GaussianBlur

```c
prSuiteError (*GaussianBlur)(
    csSDK_uint32 inDeviceIndex,
    const void* inSrc, csSDK_int32 inSrcRowBytes,
    csSDK_uint32 inSrcWidth, csSDK_uint32 inSrcHeight,
    void* inDest, csSDK_int32 inDestRowBytes,
    csSDK_uint32 inDestWidth, csSDK_uint32 inDestHeight,
    PrPixelFormat inPixelFormat,
    float inSigmaX, float inSigmaY,
    prBool inRepeatEdgePixels,
    prBool inBlurHorizontally, prBool inBlurVertically,
    PrRenderQuality inQuality);
```

---

## The PrGPU/KernelSupport Framework

The KernelSupport framework lets you write a single kernel source file that compiles for CUDA, OpenCL, Metal, and (internally) DirectX. It uses preprocessor macros and Boost.Preprocessor sequences to abstract away API differences.

### KernelCore.h -- Device Target Detection

KernelCore.h sets up target-specific macros based on the compiler:

| Macro | Set When |
|-------|----------|
| `GF_DEVICE_TARGET_CUDA` | Compiled with `__CUDACC__` (nvcc) |
| `GF_DEVICE_TARGET_OPENCL` | `GF_DEVICE_TARGET_OPENCL` defined before include |
| `GF_DEVICE_TARGET_METAL` | `GF_DEVICE_TARGET_METAL` defined before include |
| `GF_DEVICE_TARGET_HOST` | None of the above (C++ host code) |
| `GF_DEVICE_TARGET_DEVICE` | Any GPU target is active |

### Section Macros

For single-line target-specific code:

```c
GF_CUDA_SECTION(width = 16;)
GF_OPENCL_SECTION(/* OpenCL-specific pragma */)
GF_METAL_SECTION(/* Metal-specific code */)
GF_HOST_SECTION(/* Host-only code */)
GF_DEVICE_SECTION(/* Any GPU target */)
```

### Memory Pointer Abstractions

```c
GF_PTR(float4)          // Global memory pointer
                        //   CUDA/Host: float4*
                        //   OpenCL: __global float4*
                        //   Metal: device float4*

GF_THREAD_PTR(float2)   // Thread-private pointer
                        //   CUDA/Host/OpenCL: float2*
                        //   Metal: float2 thread *
```

### Alignment and Constants

```c
GF_ALIGN(8)             // Structure alignment
GF_CONSTANT(float)      // GPU constant memory declaration
GF_SHARED(Type)         // Shared/threadgroup memory
```

### GF_KERNEL_FUNCTION -- Writing Cross-Platform Kernels

This is the central macro for declaring GPU kernels. It uses Boost.Preprocessor sequences to express typed parameter pairs:

```c
GF_KERNEL_FUNCTION(MyKernel,
    // Buffers (GPU memory pointers)
    ((GF_PTR(float4))(inSrc))
    ((GF_PTR(float4))(outDest)),

    // Values (scalar parameters)
    ((int)(inPitch))
    ((int)(inWidth))
    ((int)(inHeight))
    ((float)(inBrightness)),

    // Position arguments (auto-filled by GPU runtime)
    ((uint2)(inXY)(KERNEL_XY)))
{
    if (inXY.x < inWidth && inXY.y < inHeight)
    {
        int index = inXY.y * inPitch + inXY.x;
        float4 pixel = ReadFloat4(inSrc, index, false);
        pixel.x *= inBrightness;
        pixel.y *= inBrightness;
        pixel.z *= inBrightness;
        WriteFloat4(pixel, outDest, index, false);
    }
}
```

The position argument kinds:

| Kind | Description |
|------|-------------|
| `KERNEL_XY` | Global thread position (most common) |
| `THREAD_ID` | Thread position within threadgroup |
| `BLOCK_ID` | Threadgroup position in grid |
| `BLOCK_SIZE` | Threads per threadgroup |

For kernels needing shared memory, use `GF_KERNEL_FUNCTION_SHARED` with a fifth parameter for the shared memory declaration.

### 2D Array Access

Optimized macros for reading/writing 2D image data:

```c
GF_READ2D(inPtr, inRowStride, inX, inY)     // Read pixel at (x,y)
GF_WRITE2D(inValue, outPtr, inRowStride, inX, inY)  // Write pixel at (x,y)
GF_FASTMULTIPLY(inX, inY)   // Uses mul24 on OpenCL for performance
```

### Texture Support

For texture sampling (linear interpolation, edge clamping, etc.):

```c
// Declare globally
GF_TEXTURE_GLOBAL(float4, inSrcTexture, GF_DOMAIN_NATURAL, GF_RANGE_NATURAL_CUDA, GF_EDGE_CLAMP, GF_FILTER_LINEAR)

// In kernel parameter list, use:
((GF_TEXTURE_TYPE(float))(GF_TEXTURE_NAME(inSrcTexture)))

// Sample in kernel body:
float4 color = GF_READTEXTURE(inSrcTexture, srcX + 0.5f, srcY + 0.5f);
```

### Vector Constructors

Portable vector constructors that work across all targets:

```c
make_float4(r, g, b, a)
make_float3(x, y, z)
make_float2(x, y)
make_int2(x, y)
make_uint2(x, y)
make_uchar4(b, g, r, a)
```

### Thread Synchronization

```c
SyncThreads()        // Full barrier (shared + device memory)
SyncThreadsShared()  // Shared memory barrier only
SyncThreadsDevice()  // Device memory barrier only
```

**Warning:** All threads in a group must call `SyncThreads` if any do, or the kernel will hang on some devices.

### KernelMemory.h -- 16f/32f Read/Write

These functions handle both half-precision and full-precision pixel data transparently:

```c
float4 ReadFloat4(GF_PTR(float4 const) inImage, int inIndex, bool is16Bit);
void WriteFloat4(float4 inPixel, GF_PTR(float4) outImage, int inIndex, bool is16Bit);

float2 ReadFloat2(GF_PTR(float2 const) inImage, int inIndex, bool is16Bit);
void WriteFloat2(float2 inPixel, GF_PTR(float2) outImage, int inIndex, bool is16Bit);

float ReadFloat(GF_PTR(float const) inImage, int inIndex, bool is16Bit);
void WriteFloat(float inPixel, GF_PTR(float) outImage, int inIndex, bool is16Bit);
```

Pass `is16Bit = true` when the pixel format is `PrPixelFormat_GPU_BGRA_4444_16f`. These functions automatically handle the half-to-float conversion on each platform:
- **CUDA**: Uses `cuda_fp16.h` with a custom `half4` struct
- **OpenCL**: Uses `vload_half4` / `vstorea_half4_rtz` (guarded by `GF_OPENCL_SUPPORTS_16F`)
- **Metal**: Uses native `half4` type with cast helpers

---

## The PrGPUFilterModule Helper

The SDK provides a C++ template framework in `PrGPUFilterModule.h` that simplifies GPU filter development. It includes:

### PrGPUFilterBase

A base class that handles suite acquisition and common operations:

```cpp
class PrGPUFilterBase {
public:
    virtual prSuiteError Initialize(PrGPUFilterInstance* ioInstanceData);
    virtual prSuiteError GetFrameDependencies(/* ... */);
    virtual prSuiteError Precompute(/* ... */);
    virtual prSuiteError Render(/* ... */) = 0;  // Pure virtual

protected:
    // Get a property from the video segment node
    template<typename T> prSuiteError GetProperty(const char* inKey, T& outValue);

    // Get a parameter value (index is 1-based; 0 is the input frame which GPU filters skip)
    PrParam GetParam(csSDK_int32 inIndex, PrTime inTime);

    // Utilities
    size_t RoundUp(size_t inValue, size_t inMultiple);
    int GetGPUBytesPerPixel(PrPixelFormat inPixelFormat);  // 8 for 16f, 16 for 32f

    PrSDKGPUDeviceSuite* mGPUDeviceSuite;
    PrSDKGPUImageProcessingSuite* mGPUImageProcessingSuite;
    PrSDKMemoryManagerSuite* mMemoryManagerSuite;
    PrSDKPPixSuite* mPPixSuite;
    PrSDKPPix2Suite* mPPix2Suite;
    PrSDKVideoSegmentSuite* mVideoSegmentSuite;

    piSuitesPtr mSuites;
    PrTimelineID mTimelineID;
    csSDK_int32 mNodeID;
    csSDK_uint32 mDeviceIndex;
    PrGPUDeviceInfo mDeviceInfo;
};
```

**Important:** `GetParam` subtracts 1 from the index because GPU filters do not include the input frame in their parameter list. Parameter index 1 in the CPU effect corresponds to index 0 in the segment node params.

### PrGPUFilterModule Template

Routes the C entry point callbacks to your derived class:

```cpp
template<class GPUFilter>
struct PrGPUFilterModule {
    static prSuiteError Startup(piSuitesPtr, csSDK_int32*, PrGPUFilterInfo*);
    static prSuiteError Shutdown(piSuitesPtr, csSDK_int32*);
    static prSuiteError CreateInstance(PrGPUFilterInstance*);
    static prSuiteError DisposeInstance(PrGPUFilterInstance*);
    static prSuiteError GetFrameDependencies(/* ... */);
    static prSuiteError Precompute(/* ... */);
    static prSuiteError Render(/* ... */);
};
```

### DECLARE_GPUFILTER_ENTRY Macro

Generates the `xGPUFilterEntry` export:

```cpp
DECLARE_GPUFILTER_ENTRY(PrGPUFilterModule<MyGPUEffect>)
```

---

## Complete Example: ProcAmp-Style GPU Filter

Based on the SDK's `SDK_ProcAmp` example:

### Kernel Parameter Structure

```cpp
typedef struct {
    int   mPitch;
    int   m16f;
    int   mWidth;
    int   mHeight;
    float mBrightness;
    float mContrast;
    float mHueCosSaturation;
    float mHueSinSaturation;
} ProcAmpParams;
```

### GPU Filter Class

```cpp
class MyProcAmp : public PrGPUFilterBase {
public:
    prSuiteError Initialize(PrGPUFilterInstance* ioInstanceData) override
    {
        PrGPUFilterBase::Initialize(ioInstanceData);

        if (mDeviceInfo.outDeviceFramework == PrGPUDeviceFramework_OpenCL)
        {
            cl_int result = CL_SUCCESS;
            char const* k16fString = "#define GF_OPENCL_SUPPORTS_16F 1\n";
            size_t sizes[] = { strlen(k16fString), strlen(kKernelSource) };
            char const* strings[] = { k16fString, kKernelSource };

            cl_context context = (cl_context)mDeviceInfo.outContextHandle;
            cl_device_id device = (cl_device_id)mDeviceInfo.outDeviceHandle;

            cl_program program = clCreateProgramWithSource(
                context, 2, strings, sizes, &result);
            clBuildProgram(program, 1, &device,
                "-cl-single-precision-constant -cl-fast-relaxed-math", 0, 0);
            mKernel = clCreateKernel(program, "ProcAmpKernel", &result);
        }

        return suiteError_NoError;
    }

    prSuiteError Render(
        const PrGPUFilterRenderParams* inRenderParams,
        const PPixHand* inFrames,
        csSDK_size_t inFrameCount,
        PPixHand* outFrame) override
    {
        // Get input frame data
        void* srcData = nullptr;
        mGPUDeviceSuite->GetGPUPPixData(inFrames[0], &srcData);

        PrPixelFormat pixelFormat;
        mPPixSuite->GetPixelFormat(inFrames[0], &pixelFormat);

        prRect bounds;
        mPPixSuite->GetBounds(inFrames[0], &bounds);
        int width = bounds.right - bounds.left;
        int height = bounds.bottom - bounds.top;

        csSDK_int32 rowBytes;
        mPPixSuite->GetRowBytes(inFrames[0], &rowBytes);
        int pitch = rowBytes / GetGPUBytesPerPixel(pixelFormat);

        bool is16f = (pixelFormat == PrPixelFormat_GPU_BGRA_4444_16f);

        // Read parameters from the video segment
        PrParam brightness = GetParam(1, inRenderParams->inClipTime);
        PrParam contrast = GetParam(2, inRenderParams->inClipTime);

        // Dispatch kernel (OpenCL example)
        if (mDeviceInfo.outDeviceFramework == PrGPUDeviceFramework_OpenCL)
        {
            ProcAmpParams params = { pitch, is16f ? 1 : 0, width, height,
                                     brightness.mFloat64, contrast.mFloat64,
                                     /* ... */ };

            cl_command_queue queue = (cl_command_queue)mDeviceInfo.outCommandQueueHandle;
            size_t threadBlock[2] = { 16, 16 };
            size_t grid[2] = {
                DivideRoundUp(width, threadBlock[0]) * threadBlock[0],
                DivideRoundUp(height, threadBlock[1]) * threadBlock[1]
            };

            clSetKernelArg(mKernel, 0, sizeof(cl_mem), &srcData);
            clSetKernelArg(mKernel, 1, sizeof(ProcAmpParams), &params);
            clEnqueueNDRangeKernel(queue, mKernel, 2, NULL, grid, threadBlock, 0, NULL, NULL);
        }

        // Render in-place
        *outFrame = inFrames[0];
        return suiteError_NoError;
    }

private:
    cl_kernel mKernel = nullptr;
};

DECLARE_GPUFILTER_ENTRY(PrGPUFilterModule<MyProcAmp>)
```

---

## Pitfalls and Best Practices

### Memory Management

- **Never allocate GPU memory directly** through CUDA/OpenCL/Metal APIs. Always use `PrSDKGPUDeviceSuite::AllocateDeviceMemory`. The host needs to track allocations for its memory pressure management.
- GPU PPix handles that you create must be disposed via `PrSDKPPixSuite::Dispose`.
- All GPU frames may have outstanding asynchronous operations on the compute stream. Do not assume completion until after your render callback returns.

### Parameter Access

- GPU filters access parameters through `PrSDKVideoSegmentSuite::GetParam`, NOT through `PF_ParamDef` checkout. The node ID comes from `PrGPUFilterInstance::inNodeID`.
- Parameter indices are offset by -1 compared to the CPU effect because the input frame (param index 0 in the CPU effect) is not a parameter in the GPU path.

### Thread Safety

- Multiple GPU filter instances may render concurrently on different threads. Static data (like compiled kernel caches) needs synchronization.
- The SDK examples use a simple static array cache (`sKernelCache[kMaxDevices]`) indexed by device index, which is safe because device indices are stable and distinct threads operate on distinct devices.

### Fallback Strategy

- Return an error from `CreateInstance` to gracefully fall back to software rendering. This is the intended mechanism -- do not crash or assert.
- Check `mDeviceInfo.outMeetsMinimumRequirementsForAcceleration` during initialization.

### Field Rendering

- GPU rendering defaults to full-height progressive frames. If your effect is non-spatial and does not vary over time, return `PrGPUDependency_FieldSeparation` with `outNeedsFieldSeparation = kPrFalse` to allow the host to skip field separation and render both fields simultaneously.

### Kernel Compilation

- A production plugin should cache compiled kernel binaries to disk to avoid recompilation on every launch. The SDK examples do not demonstrate this for brevity.
- For OpenCL, always define `GF_OPENCL_SUPPORTS_16F` if the device supports the `cl_khr_fp16` extension (most modern GPUs do).
- OpenCL build flags from the SDK examples: `-cl-single-precision-constant -cl-fast-relaxed-math`
