# Optimization

> 22 Q&As · source: AE plugin dev community Discord

### Can AE plugins target AVX2 instruction set for better performance?

AE has required AVX2 since version 2023, and Steam surveys show 96%+ CPU support for AVX2. It's reasonable to target x86-64-v3 (AVX2) instead of v2 (SSSE3), which could double performance for SIMD-heavy processing. ISPC is a good option for automatic multi-ISA compilation (compiles multiple versions and auto-switches at runtime). Alternatively, use runtime detection with if(HasAVX()) dispatch, though this adds complexity.

*Tags: `avx2`, `instruction-set`, `ispc`, `optimization`, `performance`, `simd`*

---

### How do you check if an input frame has changed to avoid unnecessary re-processing?

Use PF_GetCurrentState to detect if an input has changed. In SmartFX, the PreRender/SmartRender pipeline handles this more naturally. You can also use GuidMixInPtr to mix in any data that should trigger re-rendering - if parameters or other state change, mix that into the GUID and AE will know to call SmartRender again. Note: if the layer is continuously rasterized, transforms are applied before your effect receives the input.

*Tags: `caching`, `guid-mixin`, `input-change`, `optimization`, `pf-get-current-state`*

---

### Why does Premiere give a NULL input_world in the Render call when compiled with fast optimizations (-Ofast), but works without optimizations?

This is a known issue that occurs in Premiere but not AE when using -Ofast compiler optimizations. A workaround is to add __attribute__((optnone)) before the render function to disable optimization for that specific function while keeping the rest of the plugin optimized. Another suggestion is to check if the input world is null and return PF_Err_OUT_OF_MEMORY (not bad param), which may cause Premiere to retry the render. Also verify your colorspace setup in GlobalSetup and consider trying BGRA instead of ARGB.

```cpp
__attribute__ ((optnone))
static PF_Err Render(...) {
    // render code here
}
```

*Tags: `compiler`, `null-input-world`, `ofast`, `optimization`, `premiere`, `render`, `workaround`*

---

### What algorithm does the Matte/Simple Choker Effect use to find and create reduced/expanded edges on alpha channels?

The questioner describes their own implementation which analyzes each transparent pixel and looks in 8 directions to find nearby opaque pixels to identify outline/choker matte pixels. However, no answer was provided in the conversation about the actual method used by Adobe's Matte/Simple Choker Effect.

*Tags: `gpu`, `memory`, `mfr`, `optimization`, `render-loop`*

---

### What algorithm does the Matte/Simple Choker Effect use to find and create reduced/expanded edges in alpha channels?

The user describes their current implementation which analyzes each transparent pixel and looks in 8 directions to find nearby opaque pixels to identify outline/choker matte pixels. However, this approach is very slow (30+ seconds). The conversation does not provide a definitive answer about what method Adobe's Choker effect actually uses, only describes the user's current slow implementation.

*Tags: `gpu`, `memory`, `mfr`, `optimization`, `render-loop`*

---

### What algorithm does the Matte/Simple Choker Effect use to create reduced/expanded edges on alpha channels?

The Matte/Simple Choker Effect likely uses a distance field approach rather than checking neighboring pixels. A distance field uses an intermediate 2D buffer of singular distance values (such as int16_t for signed distance values) to exchange memory for speed. The Jump Flooding Algorithm (JFA) is a fast method to generate distance fields and is well-suited for this purpose, avoiding the O(n²) complexity of checking neighbors for each pixel.

*Tags: `gpu`, `memory`, `optimization`, `render-loop`*

---

### Why is the neighbor-checking algorithm for creating choker mattes too slow and what is a better approach?

The neighbor-checking approach is too slow because checking 8 directions for each transparent pixel to find nearby opaque pixels has O(n²) complexity when expanding outlines by larger amounts (e.g., 30 pixels). A distance field approach is significantly faster as it uses a 2D buffer of distance values to trade memory for speed, with Jump Flooding Algorithm being a recommended fast implementation.

*Tags: `memory`, `optimization`, `render-loop`*

---

### How do you handle different color spaces when developing plugins for both After Effects and Premiere?

In After Effects, plugins typically work in ARGB colorspace. Premiere uses BGRA_8u with a special suite for 8-bit processing. For 32-bit float, you need to write your own conversion function. You can use the Iterate8Suite1 (or SuiteV2 since 2022) for ARGB, but Premiere requires a special BGRA suite. You don't have to support all color spaces—you can optimize for BGRA only and skip YUVA if needed. Basic rendering in 8/16 bit works without optimization, though you won't get the 32-bit icon next to the plugin. Check the SDK noise example for reference implementations of BGRA and YUVA colorspace handling.

*Tags: `argb`, `bgra`, `color-space`, `optimization`, `premiere`, `render-loop`*

---

### How can VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension (available on Windows) avoids the need to allocate a staging buffer by allowing a CPU-side void* to be turned directly into a VkDeviceMemory object. This enables copying data directly from an EffectWorld onto the GPU and allows the GPU to write back into it without redundant upload/download copies, resulting in significant gains in both memory and speed, especially for large 4K frames.

*Tags: `gpu`, `memory`, `optimization`, `vulkan`, `windows`*

---

### Should you use cv::mixChannels with memcpy or the IterateSuite for channel swizzling performance?

You should benchmark both approaches. Using cv::mixChannels followed by memcpy may be slower than using After Effects' IterateSuite (which provides one thread per row) and performing channel swizzling manually during the copy operation from matrix to layer. The IterateSuite approach can provide better performance by combining the channel conversion and copy in a single operation per row.

*Tags: `aegp`, `optimization`, `render-loop`, `threading`*

---

### Does smart render provide benefits for effects that require the full image for processing?

Smart render helps by only requesting pixels required for specific processing. For effects that require the full image each time they are processed (when visible), smart render may not provide additional optimization benefits since the entire image must be fetched anyway. The advantage of smart render is primarily for effects that can work on partial regions of interest.

*Tags: `optimization`, `output-rect`, `render-loop`, `smart-render`*

---

### Can hiding layers by default improve performance when dealing with many layers in After Effects?

Hiding layers by default can be an effective optimization strategy to reduce the performance load on the UI and API calls, though the specific performance gains depend on the implementation and number of layers being processed.

*Tags: `layer-checkout`, `optimization`, `performance`, `ui`*

---

### What is the purpose of the rowbytes field in layer data structures?

The rowbytes field exists as an optimization to allow layers to be cropped without allocating extra memory. A layer can be cropped by simply updating width and height and making the data pointer reference the new top-left pixel, while leaving rowbytes untouched. After Effects uses this technique frequently, for example when dragging a layer partially off the composition.

*Tags: `layer-checkout`, `memory`, `optimization`*

---

### What is the purpose of rowbytes in After Effects layer data, and how does it enable memory optimization?

Rowbytes exists as an optimization so that layers can be cropped without allocating extra memory. You can crop a layer by updating width and height and making the data pointer reference the new top-left pixel, while leaving rowbytes unchanged and maintaining a reference to the original data for proper deallocation. After Effects uses this optimization frequently, such as when dragging a layer partially off the composition. Rowbytes is calculated as 4*width for 32-bit data, but may be larger if a layer is loaded and moved partially off-comp; however, purging cache typically resets it back to 4*width.

*Tags: `caching`, `memory`, `optimization`, `output-rect`*

---

### How much faster is GPU rendering compared to CPU path for plugin operations?

According to benchmarks, there is a significant difference: GPU processing can complete operations in milliseconds while the CPU path takes several seconds. However, performance depends on the actual operations being offloaded—if critical work like UV unwrapping still falls back to CPU, the GPU advantage is negated.

*Tags: `benchmark`, `cpu`, `gpu`, `optimization`, `performance`*

---

### What approach works well for implementing GPU acceleration in After Effects plugins?

A practical approach is to present your plugin as a CPU plugin to After Effects while implementing GPU rendering (Vulkan, OpenGL, or WebGPU) under the hood. This avoids compatibility issues while still leveraging GPU acceleration. Ensure that critical operations are actually executed on the GPU—falling back to CPU paths defeats the purpose.

*Tags: `gpu`, `optimization`, `rendering`, `vulkan`, `webgpu`*

---

### How can you prevent the compiler from optimizing a render function that is causing issues in an After Effects plugin?

Add the `__attribute__ ((optnone))` compiler attribute before the render function declaration. This tells the compiler to skip optimization for that specific function, even if the rest of the plugin is optimized. This was identified as a workaround by the community, though it may indicate an underlying issue worth investigating further.

```cpp
__attribute__ ((optnone))
void render() {
  // render function implementation
}
```

*Tags: `build`, `debugging`, `optimization`, `render-loop`*

---

### Is it better to apply multiple transform_world operations separately or combine them into a single matrix multiplication?

Always combine transformations into a single matrix multiplication rather than applying multiple transform_world renders. Matrix multiplication of 3x3 matrices is extremely cheap (~20 operations), while each transform_world render performs millions of pixel operations. Additionally, multiple sequential transforms cause image degradation (blur/jaggedness) because pixels that don't land on exact whole pixels get blended, compounding with each render. A single combined transformation applied once produces superior quality and performance.

*Tags: `optimization`, `performance`, `render-loop`, `transform_world`*

---

### Does the number of available threads returned by AEGP_GetNumThreads change during an After Effects session?

To the best of knowledge, the number of available threads does not change throughout an AE session. The decision on using PF_Iterations_ONCE_PER_PROCESSOR is an optimization decision that depends on your algorithm. For image processing, PF_Iterations_ONCE_PER_PROCESSOR is rarely optimal because some threads finish their chunk before others and cannot help with remaining work on other threads. Using n iterations instead assures minimal loss of available CPU power for most image processing algorithms.

*Tags: `aegp`, `optimization`, `render-loop`, `threading`*

---

### Why is there no performance improvement when calling suite functions repeatedly inside iterate_generic callbacks?

Repeatedly acquiring suites inside iteration callbacks creates significant overhead that negates parallel execution benefits. Suite acquisition should be done once from the main thread before the iteration, and then pointers to the callbacks should be passed to the iteration function. Pre-acquiring suites can improve performance by orders of magnitude compared to acquiring them for every iteration.

*Tags: `caching`, `optimization`, `performance`, `smartfx`, `threading`*

---

### Should I use the Iterate Suite with many iterations containing inner loops, or fewer iterations with larger inner loops?

When using the Iterate Suite with multithreading, the best approach depends on how well tasks can be split between cores rather than mathematical efficiency. If you have 4-75 outer iterations with 10,000 inner loop iterations, fewer large iterations would have less overhead. However, the optimal strategy is to break the work into smaller tasks that can run in parallel across available threads. With variable inner loop sizes (10-10,000), consider distributing tasks evenly across cores to avoid situations where some threads finish early and wait for others. Testing different configurations is recommended to find the optimal thread count for your specific workstation.

*Tags: `iterate-suite`, `multithreading`, `optimization`, `performance`, `threading`*

---

### What is the performance impact of acquiring and releasing the sample suite on every pixel during iteration?

Acquiring and releasing the sample suite on every pixel significantly weighs down performance. For optimal results when you need to sample many adjacent pixels, it's recommended to access the input and output buffers directly and use iterate_generic instead, which allows each thread to handle a single buffer line and eliminates the overhead of repeated suite acquisition/release operations.

*Tags: `ccu`, `memory`, `optimization`, `performance`, `threading`*

---
