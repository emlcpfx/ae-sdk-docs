# CPU/GPU Render Parity

> Ensuring CPU and GPU render paths produce matching results in AE plugins.

### What pixel format does AE use for GPU worlds vs CPU worlds?

AE GPU worlds (`PF_GPUDeviceSuite::GetGPUWorldData`) use `PF_PixelFormat_GPU_BGRA128` — **BGRA float4**:

- `.x` = **Blue**
- `.y` = **Green**
- `.z` = **Red**
- `.w` = **Alpha**

This is different from both CPU `PF_PixelFloat` (ARGB: `.alpha, .red, .green, .blue`) and most GPU tutorials that assume RGBA.

If your plugin supports both CPU and GPU paths, the pixel swizzle matters:

- **CPU path**: receives ARGB-ordered `PF_PixelFloat`. Access channels as `.red`, `.green`, `.blue`, `.alpha`.
- **GPU path**: receives BGRA float4 device pointers. Extract channels as: `float pR = px.z, pG = px.y, pB = px.x, pA = px.w;`

**This is the #1 source of GPU color bugs.** Effects that treat all RGB channels identically (blur, erode, composite) won't notice the swap. But any effect that operates differently per channel (film grain, color correction, channel mixer) will have R and B swapped if the kernel assumes RGBA.

The enum name `PF_PixelFormat_GPU_BGRA128` is authoritative — the format is BGRA.

*Tags: `argb`, `buffer`, `cpu`, `gpu`, `pixel-format`, `rgba`, `smart-render`*

---

### What are the most common causes of CPU/GPU render mismatches?

Several categories of logic can drift between CPU and GPU implementations. Use this checklist when validating parity:

**Octave count / LOD thresholds** — CPU render paths sometimes auto-drop detail levels (e.g., skipping a second noise octave when the feature scale falls below 2 pixels). GPU kernels must replicate the same LOD logic or grain/noise character will visibly differ between CPU and GPU renders.

**Response curves and LUTs** — CPU code may use a 256-entry lookup table for a response curve while GPU computes it analytically per pixel. Minor floating-point precision differences between the two are acceptable, but the underlying math must be identical. If the CPU uses a piecewise approximation, the GPU must use the same breakpoints and coefficients.

**Optimized fast paths** — CPU code often has shortcuts: monochrome pixel detection, fully-opaque skip paths, early-out for zero-strength params. The GPU kernel may not have these optimizations, which is fine, but it must produce equivalent output for those same input cases. Test edge cases like fully transparent pixels, zero-strength sliders, and single-channel inputs.

**Pixel aspect ratio (PAR)** — If your effect applies spatial operations (blur radii, distance thresholds, coordinate transforms), PAR correction must be applied identically in both paths. A common bug is correcting for PAR on CPU but using square-pixel math on GPU, causing stretched results in non-square-pixel compositions.

**Seed computation** — Frame-based random seeds (for noise, scatter, jitter) must use the same hash chain in both paths. If the CPU path hashes `(frame_number, seed_param)` with a specific algorithm, the GPU kernel must use the identical hash, not a "close enough" alternative.

*Tags: `cpu`, `gpu`, `lut`, `noise`, `par`, `parity`, `render`, `seed`*

---

### How do you load Metal shader libraries in AE plugins?

`newDefaultLibrary` does **not** work for AE plugins — it looks for a `.metallib` compiled into the app bundle, which is the host application (After Effects), not your plugin. You must load your `.metal` source file from the plugin bundle's Resources directory and compile it at runtime:

```objc
NSString *path = [[NSBundle bundleForClass:[YourPluginClass class]]
                    pathForResource:@"MyKernel" ofType:@"metal"
                    inDirectory:@""];
// Or use the plugin's resource path from AE's in_data
NSString *source = [NSString stringWithContentsOfFile:path
                             encoding:NSUTF8StringEncoding error:&err];
id<MTLLibrary> library = [device newLibraryWithSource:source
                                   options:nil error:&err];
```

Your **CMakeLists.txt** must copy `.metal` files into the plugin bundle:

```cmake
set_source_files_properties(gpu/MyKernel.metal PROPERTIES
    MACOSX_PACKAGE_LOCATION "Resources")
```

This ensures the `.metal` file ends up in `YourPlugin.plugin/Contents/Resources/MyKernel.metal`.

*Tags: `bundle`, `cmake`, `macos`, `metal`, `shader`*

---

### What are the Metal command buffer and encoder rules for AE GPU rendering?

Use **one command buffer** and **one compute command encoder** for all kernel dispatches within a single render call. Creating multiple command buffers or encoders causes synchronization issues and can crash AE.

```objc
@autoreleasepool {
    id<MTLCommandBuffer> cmdBuf = [queue commandBuffer];
    id<MTLComputeCommandEncoder> encoder = [cmdBuf computeCommandEncoder];

    // Dispatch all kernels through this single encoder
    [encoder setComputePipelineState:pipeline1];
    [encoder setBuffer:inputBuf offset:0 atIndex:0];
    [encoder dispatchThreadgroups:gridSize1
            threadsPerThreadgroup:threadGroupSize1];

    [encoder setComputePipelineState:pipeline2];
    [encoder dispatchThreadgroups:gridSize2
            threadsPerThreadgroup:threadGroupSize2];

    [encoder endEncoding];
    [cmdBuf commit];
    [cmdBuf waitUntilCompleted];
}
```

Key rules:

- Always wrap Metal calls in `@autoreleasepool` to prevent leaking Objective-C objects across render calls.
- Use `dispatchThreadgroups:threadsPerThreadgroup:` (indirect grid size), not `dispatchThreads:threadsPerThreadgroup:`. The latter requires non-uniform threadgroup support and may not be optimal for all GPU architectures.
- Choose threadgroup sizes that align with your pipeline's `threadExecutionWidth` for best occupancy.

*Tags: `command-buffer`, `gpu`, `macos`, `metal`, `threading`*

---

### How should x86 SIMD intrinsics be guarded for cross-platform AE builds?

CPU render paths that use AVX2, SSE4, or other x86 SIMD intrinsics must be guarded with architecture checks, or they will fail to compile on ARM64 macOS (Apple Silicon) builds:

```cpp
#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
    #include <immintrin.h>
    // AVX2 / SSE optimized path
    __m256 acc = _mm256_setzero_ps();
    // ...
#else
    // Scalar fallback for ARM64 / other architectures
    float acc[8] = {0};
    // ...
#endif
```

For runtime detection on x86 (supporting both SSE-only and AVX2-capable CPUs):

```cpp
#if defined(__x86_64__) || defined(_M_X64)
    #ifdef _MSC_VER
        int cpuInfo[4];
        __cpuid(cpuInfo, 7);
        bool hasAVX2 = (cpuInfo[1] & (1 << 5)) != 0;
    #else
        bool hasAVX2 = __builtin_cpu_supports("avx2");
    #endif
#endif
```

If your plugin ships on macOS, it must compile cleanly as a Universal Binary (x86_64 + arm64). Any unguarded x86 intrinsic will cause a build failure for the arm64 slice.

*Tags: `arm64`, `avx2`, `cpu`, `cross-platform`, `macos`, `simd`, `sse`, `windows`*

---
