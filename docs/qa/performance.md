# Q&A: performance

**46 entries** tagged with `performance`.

---

## What's a good approach for logging in AE plugins?

Use spdlog library which supports file rotation (configurable file size and number of files to keep). For performance-sensitive plugins, use conditional logging: check for the existence of a specific file (e.g., 'plugin_log.txt') during GlobalSetup, and only enable logging when that file is present. This lets users enable logging on demand for debugging.

*Contributors: [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2023-08-17 · Tags: `logging`, `spdlog`, `debugging`, `performance`*

---

## What causes slow AE loading when debugging from Xcode 15?

After upgrading to macOS 14/Xcode 15, AE loading becomes ~3x slower due to 'dlsym cannot find symbol xSDKExport' messages for MediaCore bundles. This is Xcode trying to load symbols for all loaded bundles. Workarounds: (1) Run Xcode 14 binary directly from Terminal. (2) Open AE standalone first, then attach the debugger after AE is fully loaded. (3) Check if 'load symbols' option is accidentally selected in your IDE settings.

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**tlafo**](../contributors/tlafo/), [**gabgren**](../contributors/gabgren/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2023-11-29 · Tags: `xcode`, `debugging`, `performance`, `macos`, `symbol-loading`*

---

## Can AE plugins target AVX2 instruction set for better performance?

AE has required AVX2 since version 2023, and Steam surveys show 96%+ CPU support for AVX2. It's reasonable to target x86-64-v3 (AVX2) instead of v2 (SSSE3), which could double performance for SIMD-heavy processing. ISPC is a good option for automatic multi-ISA compilation (compiles multiple versions and auto-switches at runtime). Alternatively, use runtime detection with if(HasAVX()) dispatch, though this adds complexity.

*Contributors: [**wunk**](../contributors/wunk/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2025-03-19 · Tags: `avx2`, `simd`, `performance`, `ispc`, `instruction-set`, `optimization`*

---

## How can I chain multiple transform_world calls, using the output of one as input for the next?

Never use the same buffer as both input and output of transform_world, as it reads and overwrites simultaneously, causing corrupted output. Never overwrite the input buffer (AE caches it). For repeated transforms: (1) Create a temp buffer, (2) transform input to temp, (3) transform temp to output, (4) transform output to temp, (5) repeat as needed, (6) copy to output if last result is in temp. However, matrix multiplication is extremely cheap (20-something operations for 3x3), while each transform_world render costs millions of operations per image. Always prefer multiplying matrices and doing a single render.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `transform-world`, `matrix-multiplication`, `buffer-management`, `performance`, `rendering`*

---

## What is the difference between thread_indexL and i in iterate_generic?

The 'i' parameter is the iteration index (not sequential - may be reordered to minimize CPU cache conflicts), while thread_indexL is the thread index. With 4 threads and 8 iterations, i values may arrive as 6,0,2,7,4,5,1,3 on threads 3,0,1,3,2,2,0,1. They only match when iterations equals threads (e.g., PF_Iterations_ONCE_PER_PROCESSOR). The thread count doesn't change during an AE session. For image processing, it's better NOT to use PF_Iterations_ONCE_PER_PROCESSOR because some threads will finish early and sit idle. Using N iterations ensures minimal CPU power loss. Never call iterate_generic from within iterate_generic, and don't call suites from non-thread-0 threads.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `iterate-generic`, `multithreading`, `thread-index`, `iteration`, `performance`*

---

## Can I use iterate_generic to parallelize transform_world calls?

No, transform_world is not safe to call from utility threads. AE has main threads (UI and render) and utility threads (used by iteration suite). Interaction with AE is only allowed from main threads. transform_world is already internally multithreaded, so calling it from parallel utility threads won't help and may cause bugs/crashes. Very few API callbacks are safe from non-main threads (e.g., subpixel sample callbacks, but suites must be acquired in advance from main threads). If you need parallel image transformations, write your own transform function with nearest-neighbor sampling that can safely run on multiple threads.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `iterate-generic`, `transform-world`, `multithreading`, `thread-safety`, `performance`*

---

## Should I use multithreading (iterate_generic) or MFR (Multi-Frame Rendering) in my plugin, or both?

Use both. MFR and multithreading can coexist because not all steps in the single-frame rendering pipeline are multi-threadable. MFR shines for sequential parts, while multithreaded parts (via iterate callbacks) are critical for performance. Mixing MFR with MT doesn't necessarily cause competition because MT-friendly code tends to be CPU cache friendly, while other parts may leave the CPU waiting for RAM. When using iteration callbacks, AE may prioritize between MFR and MT threads. The AE engineering team tested these scenarios extensively.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `mfr`, `multithreading`, `multi-frame-rendering`, `iterate-generic`, `performance`*

---

## How should logging be implemented in plugins without slowing down render performance?

For real-time or fast-rendering plugins, logging can impact performance. Implement conditional logging by checking for a marker file (e.g., 'pixel_sorter_log.txt') in global setup, enabling logging only when the file exists. This allows users to request logging without it affecting normal operation. Use spdlog library which includes file rotation capabilities to manage log file sizes and count.

*Tags: `debugging`, `performance`, `pipl`*

---

## What are the practical limits for working with a composition containing thousands of layers in After Effects?

After Effects struggles significantly even at 512 layers. As a user reports, working with over 500 layers becomes a nightmare in terms of performance and usability. The software has inherent limitations that make working with very large numbers of layers (such as 10,000) impractical, even if they are instances of only a small number of source files.

*Tags: `memory`, `performance`, `render-loop`, `caching`*

---

## What causes slowness when displaying a timeline with many layers in After Effects?

The slowness is primarily in the UI rendering rather than the plugin rendering itself. The bottleneck is likely in After Effects internal calls such as 'get layer info' and 'get layer sprite'. Even with many layers hidden by default, performance issues persist. After Effects also has poor memory management, making it inefficient with large numbers of layers (10k+).

*Tags: `ui`, `memory`, `performance`, `aegp`*

---

## Would using an Artisan API approach help speed up rendering with many layers?

An Artisan API approach likely wouldn't meaningfully speed up the performance, since the bottleneck is in After Effects' internal layer information retrieval calls rather than the rendering process itself.

*Tags: `artisan`, `aegp`, `performance`*

---

## What performance improvement was achieved after Adobe fixed the arb handling issue?

A x7 speed up was observed after Adobe fixed something in arb handling, which also fixed copy/paste effect times issues that were previously causing performance problems.

*Tags: `arb-data`, `caching`, `performance`*

---

## How can you achieve incremental compilation for After Effects plugins to improve build times?

Using the Ninja generator with cmake can provide faster incremental compilation. The xcodebuild CLI tool does not appear to support incremental compilation effectively, resulting in full rebuilds taking around 50 seconds. Switching from Xcode project generation to Ninja as the cmake generator should enable proper incremental compilation.

*Tags: `build`, `cmake`, `macos`, `performance`*

---

## How can a plugin require AVX2 support or handle the AVX2 cutoff in After Effects?

After Effects has required AVX2 since version 2023. Given that Steam hardware survey shows 98.25% support for AVX and 96.30% support for AVX2 (with growing adoption rates), targeting AVX2 is practical for modern plugins. Rather than wrapping every dispatch with if(HasAVX()) checks, you can compile your plugin with AVX2 as a baseline requirement, allowing you to leverage AVX2 optimizations throughout your codebase without conditional branching. This approach can potentially double performance compared to targeting x86-64-v2 (up to SSSE3).

*Tags: `build`, `deployment`, `windows`, `performance`*

---

## Does using sequence data in SmartRender have a negative impact on performance?

No significant performance impact was observed. The read-only access pattern through the PF_EffectSequenceDataSuite in SmartRender does not cause noticeable performance degradation.

*Tags: `sequence-data`, `smart-render`, `performance`*

---

## How slow is CPU to GPU upload and has it been benchmarked?

The question was asked but not directly answered in the conversation. Jonah asked for benchmarking data on CPU→GPU upload performance, but no specific benchmark results were provided.

*Tags: `gpu`, `memory`, `performance`*

---

## Why does GPU mode still run slow when the work is being done on GPU?

The performance issue occurs when critical operations fall back to CPU processing. For example, in UV unwrapping for 3D models, if a mask is present in GPU mode, the data passed is post-mask which breaks the mathematical requirements, forcing the hard work back to the CPU path, negating GPU performance benefits.

*Tags: `gpu`, `memory`, `performance`, `render-loop`*

---

## What are the performance characteristics and trade-offs of WebGPU libraries like WGPU versus alternatives?

WGPU library has good compatibility but not great performance, with unavoidable bounds checking in shaders (unless using SPIRV shaders directly). Google's Dawn offers much better performance but is difficult to build. In practice, WebGPU-based plugins are usable but not super fast - performance depends on the effect logic, such as iterating over multiple frames.

*Tags: `gpu`, `webgpu`, `wgpu`, `performance`, `vulkan`*

---

## Did you test the GI rendering performance in quarter resolution?

The conversation suggests testing in quarter resolution as a next step to evaluate performance improvements, though the specific test results are not explicitly stated.

*Tags: `gpu`, `render-loop`, `performance`*

---

## What is the recommended approach for multithreading in Premiere plugins when the iterate suite doesn't work?

If the Iterate8Suite doesn't work reliably in Premiere (particularly with certain colorspace configurations), you should implement your own multithreading code instead of relying on the suite. Standard approaches include using std::parallel on Windows and equivalent threading libraries for Mac (like Grand Central Dispatch or pthreads). The iterate suite should theoretically work for 8-bit ARGB, but if you encounter issues, custom multithreading implementations are a stable fallback.

*Tags: `premiere`, `threading`, `multithreading`, `performance`, `bgra`, `macos`, `windows`*

---

## What is an efficient algorithm for creating edge outlines or choker mattes based on alpha channels?

Instead of checking neighbors for each transparent pixel (which results in O(n^2) complexity), use a distance field approach. Create an intermediate 2D buffer of signed distance values (e.g., int16_t) to exchange memory for speed. The Jump Flooding Algorithm (JFA) is a fast method for generating distance fields and is well-suited for this exact use case.

*Tags: `gpu`, `algorithm`, `performance`, `memory`, `matte`*

---

## Why does Multi-Frame Rendering (MFR) start with multiple threads but reduce to a single thread on longer compositions?

James Whiffin observed that MFR often initiates rendering with 6-10 threads but consistently drops to single-threaded operation when compositions are sufficiently long. This behavior persists even when custom effects are disabled and only first-party Adobe effects are present, suggesting it is a characteristic of MFR itself rather than third-party plugin behavior.

*Tags: `mfr`, `threading`, `render-loop`, `performance`*

---

## Why is the paramUI suite slower when changing a parameter name in After Effects 2023?

The paramUI suite has been observed to be slower when changing a parameter name in After Effects 2023 (version 23.2.1). An alternative approach using the streamSuite has been found to work well as a workaround for this performance issue.

*Tags: `params`, `ui`, `performance`, `aegp`*

---

## How can you add conditional logging to an After Effects plugin without slowing down renders?

Implement a file-based toggle for logging by checking for a specific file (e.g., 'pixel_sorter_log.txt') in global setup at startup. This allows users to enable detailed logging only when needed for debugging, without impacting render performance during normal operation.

*Tags: `debugging`, `macos`, `plugin-development`, `performance`, `logging`*

---

## Why is After Effects taking a very long time to load in the debugger after upgrading to Xcode 15?

The issue may be related to symbol loading being enabled in your debugger settings. Check your Xcode debugger options and disable symbol loading if it's turned on, as this can significantly slow down AE launch times. In Visual Studio, you can stop symbol loading during debugging to speed this up.

*Tags: `debugging`, `macos`, `build`, `performance`*

---

## What are the limitations of working with thousands of layers in After Effects, and how do plugins like Newton handle this?

After Effects struggles with performance even at 512 layers. The user is asking whether Newton (a physics simulation plugin) encounters the same scaling issues when simulating many layers, and considering whether switching to GPU-based sprite rendering would be necessary to maintain flexibility while handling 10,000 layer instances.

*Tags: `memory`, `render-loop`, `gpu`, `performance`, `ae-limits`*

---

## What are the common performance bottlenecks when working with large numbers of layers in After Effects plugins?

Performance issues with large layer counts (800+ layers) typically stem from UI rendering rather than the render engine itself. The bottleneck often comes from repeated After Effects API calls like 'get layer info' and 'get layer sprite', not from Artisan optimization. Additionally, After Effects has poor memory management with large layer counts, making it inefficient to handle thousands of layers.

*Tags: `aegp`, `ui`, `memory`, `performance`, `debugging`*

---

## Can hiding layers by default improve performance when dealing with many layers in After Effects?

Hiding layers by default can be an effective optimization strategy to reduce the performance load on the UI and API calls, though the specific performance gains depend on the implementation and number of layers being processed.

*Tags: `ui`, `performance`, `optimization`, `layer-checkout`*

---

## What performance improvements were observed after Adobe fixed arb handling?

Adobe recently fixed an issue in arb (arbitrary data) handling that resolved copy/paste effect performance issues. A notable case involved applying Trapcode, which showed approximately 7x speed improvement after the fix, though this may have been an unintended consequence of the arb handling changes.

*Tags: `arb-data`, `performance`, `debugging`*

---

## What is the minimum CPU instruction set support required for After Effects plugins in 2025?

After Effects has required AVX2 support since version 2023. Based on Steam hardware survey data, 98.25% of machines support AVX (growing 0.77% monthly) and 96.30% support AVX2 (growing 1.63% monthly). Even considering only video editing machines, the overlap suggests virtually no users in 2025 are on machines without AVX2 support. This makes it safe to target higher instruction sets like x86-64-v2 (up to SSSE3) or AVX2 for significant performance improvements without runtime checks.

*Tags: `build`, `cpu`, `performance`, `windows`*

---

## What tool can automatically compile multiple CPU instruction set versions and switch between them at runtime?

ISPC (Intel SPMD Program Compiler) can compile multiple versions of kernels with different CPU instruction sets (like AVX, AVX2) and automatically switch between them at runtime. This is useful for plugin developers who want to support a range of hardware while taking advantage of advanced SIMD instructions on capable processors.

*Tags: `build`, `cpu-optimization`, `cross-platform`, `performance`*

---

## How can you implement debouncing for callback functions in C++?

Alex Bizeau from maxon shared a template function that implements debouncing using std::chrono for timing and std::thread for delayed execution. The debounce function wraps a callback and returns a new function that only executes the original callback after a specified delay has passed without additional calls. It uses a shared_ptr to track the last call time and spawns a detached thread that checks if enough time has elapsed before invoking the original function.

```cpp
#include <chrono> #include <functional> #include <thread>  template <typename Func, typename... Args> std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {     std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =         std::make_shared<std::chrono::steady_clock::time_point>();      return [=](Args... args) {         *lastCallTime = std::chrono::steady_clock::now();         std::thread([=]() {             std::this_thread::sleep_for(delay);             if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {                 func(args...);             }         }).detach();     }; }
```

*Tags: `threading`, `performance`, `reference`*

---

## What GPU API should I use for After Effects plugins and how do they compare?

Vulkan is highly recommended for maximum control over performance and memory management. It's explicit and detailed, allowing fine-tuning of memory transfers and format conversions. On macOS, MoltenVK (originally by Valve) automatically ports Vulkan features to Metal, with Khronos developing KosmicKrisp as its successor. WebGPU is easier to use and cross-platform but may not provide access to platform-specific optimizations. OpenGL is mature but deprecated—it's now typically implemented via Metal, DirectX, or Vulkan under the hood on all platforms. Vulkan also supports interoperability with CUDA, OpenCL, DirectX, Metal, and OpenGL, allowing you to import textures from After Effects' native GPU features into Vulkan.

*Tags: `vulkan`, `gpu`, `metal`, `opengl`, `cross-platform`, `performance`*

---

## How much faster is GPU rendering compared to CPU path for plugin operations?

According to benchmarks, there is a significant difference: GPU processing can complete operations in milliseconds while the CPU path takes several seconds. However, performance depends on the actual operations being offloaded—if critical work like UV unwrapping still falls back to CPU, the GPU advantage is negated.

*Tags: `gpu`, `performance`, `cpu`, `optimization`, `benchmark`*

---

## Is it possible to achieve progressive rendering in an After Effects plugin where samples render incrementally and display results to the user in real-time?

gabgren mentioned porting functionality from outside AE where progressive sampling over time with continuous result display is possible, making the interaction feel more responsive. They noted that the current AE plugin workflow requires waiting for all samples to render before seeing results on screen, which feels slow. They're exploring whether AE plugins can support this type of progressive rendering approach.

*Tags: `smart-render`, `render-loop`, `ui`, `mfr`, `performance`*

---

## Is it better to apply multiple transform_world operations separately or combine them into a single matrix multiplication?

Always combine transformations into a single matrix multiplication rather than applying multiple transform_world renders. Matrix multiplication of 3x3 matrices is extremely cheap (~20 operations), while each transform_world render performs millions of pixel operations. Additionally, multiple sequential transforms cause image degradation (blur/jaggedness) because pixels that don't land on exact whole pixels get blended, compounding with each render. A single combined transformation applied once produces superior quality and performance.

*Tags: `transform_world`, `performance`, `optimization`, `render-loop`*

---

## Can iterate_generic be used to parallelize multiple transform_world calls across threads?

No, iterate_generic should not be used to call transform_world from utility threads. The iterate_generic suite uses utility threads that are separate from the main render threads, and AE API interaction is only safe from main threads. transform_world is internally multithreaded but is not designed to be called from non-main utility threads, which can result in bugs or crashes. Additionally, acquiring suites repeatedly in the iteration callback causes significant overhead. Instead, consider writing a custom transform implementation optimized for multi-threaded execution.

```cpp
suites.Iterate8Suite1()->iterate_generic(
  numThreads,
  &xform_data,
  MT_Xform
);

PF_Err MT_Xform(void* refcon, A_long threadInd, A_long iterNum, A_long iterTotal) {
  // Avoid calling transform_world or other AE suite functions from here
  // Pre-acquire suites in main thread instead
}
```

*Tags: `threading`, `render-loop`, `aegp`, `smartfx`, `performance`*

---

## Why is there no performance improvement when calling suite functions repeatedly inside iterate_generic callbacks?

Repeatedly acquiring suites inside iteration callbacks creates significant overhead that negates parallel execution benefits. Suite acquisition should be done once from the main thread before the iteration, and then pointers to the callbacks should be passed to the iteration function. Pre-acquiring suites can improve performance by orders of magnitude compared to acquiring them for every iteration.

*Tags: `threading`, `performance`, `caching`, `smartfx`, `optimization`*

---

## What is the most performant way to iterate through pixels when converting 8-bit to 16-bit data in After Effects plugins?

Use IterateGeneric to iterate through each line instead of each pixel, which provides much better performance compared to iterating pixel-by-pixel. Additionally, consider using PF_COPY for the actual pixel conversion operation, and refer to methods from the Transformer example that convert PF_Pixel to PF_Pixel16 if needed.

*Tags: `memory`, `caching`, `performance`, `aegp`, `sdk`*

---

## How do you disable a layer's video switch from a C++ plugin effect?

Use AEGP_GetEffectLayer to get the layer handle from the effect reference, then use AEGP_SetLayerFlag with AEGP_LayerFlag_VIDEO_ACTIVE set to FALSE to disable the layer's video output. This is more efficient for render performance than setting opacity to 0.

```cpp
AEGP_LayerH layerH;
suites.AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
suites.AEGP_SetLayerFlag(layerH, AEGP_LayerFlag_VIDEO_ACTIVE, FALSE);
```

*Tags: `aegp`, `layer-checkout`, `performance`, `sdk`*

---

## Should I use the Iterate Suite with many iterations containing inner loops, or fewer iterations with larger inner loops?

When using the Iterate Suite with multithreading, the best approach depends on how well tasks can be split between cores rather than mathematical efficiency. If you have 4-75 outer iterations with 10,000 inner loop iterations, fewer large iterations would have less overhead. However, the optimal strategy is to break the work into smaller tasks that can run in parallel across available threads. With variable inner loop sizes (10-10,000), consider distributing tasks evenly across cores to avoid situations where some threads finish early and wait for others. Testing different configurations is recommended to find the optimal thread count for your specific workstation.

*Tags: `threading`, `iterate-suite`, `performance`, `multithreading`, `optimization`*

---

## How can I efficiently access neighboring pixel values during CCU iteration without using sampleIntegral32 on every pixel?

You can use pointer math to access adjacent pixels directly from the input buffer pointer. If inP is a pointer to the current pixel, you can get the next pixel on the same line with `PF_Pixel *next = inP + 1;`. However, for better performance when needing multiple integer samples of other pixels, use iterate_generic instead of the regular iteration suites, as it allows much more optimization potential than acquiring and releasing the sample suite on every pixel.

```cpp
PF_Pixel *next = inP + 1; // gives you the next pixel on the same line
```

*Tags: `ccu`, `mfr`, `memory`, `performance`, `iteration`, `aegp`*

---

## What is the performance impact of acquiring and releasing the sample suite on every pixel during iteration?

Acquiring and releasing the sample suite on every pixel significantly weighs down performance. For optimal results when you need to sample many adjacent pixels, it's recommended to access the input and output buffers directly and use iterate_generic instead, which allows each thread to handle a single buffer line and eliminates the overhead of repeated suite acquisition/release operations.

*Tags: `ccu`, `performance`, `memory`, `threading`, `optimization`*

---

## What is the performance difference between using a JavaScript scheduled task versus idle_hook for detecting changes in After Effects?

While a direct performance comparison was not formally tested, idle_hook is intuitively less resource-intensive than a scheduled task. For AEGPs, the update menu hook may be called when selections change, though it's uncertain whether it fires when the selection changes or only when the relevant menu is exposed.

*Tags: `aegp`, `scripting`, `performance`, `render-loop`*

---

## How can I optimize subpixel sampling performance in After Effects plugins?

To optimize subpixel sampling, acquire the sampling suite once before your rendering loop using AEFX_AcquireSuite, get a direct pointer to it, and release it afterwards. Avoid acquiring and releasing the suite for each sample operation. Alternatively, define the AEGP_SuitesHandler before the loop and pass a pointer to it to the sampling function, though acquiring the suite directly is likely faster since it reduces function call overhead.

*Tags: `mfr`, `aegp`, `performance`, `sampling`, `render-loop`*

---

## How can I efficiently update many parameters without causing UI slowness from repeated PF_ChangeFlag_CHANGED_VALUE flags?

Instead of setting PF_ChangeFlag_CHANGED_VALUE for each of many parameters, use AEGP_SetStreamValue() from the AEGP Suite to update parameter values more efficiently. Alternatively, if the parameters don't need to be directly exposed as individual sliders in the UI, store them in sequence data or arbitrary data, which are automatically saved with the project without requiring change flags for each value.

```cpp
params[paramId]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `ui`, `performance`*

---
