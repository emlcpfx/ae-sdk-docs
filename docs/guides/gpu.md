# GPU Rendering

> 93 Q&As · source: AE plugin dev community Discord

### How do you pass custom data structs to GPU kernels in AE plugins?

Define the struct in both the kernel code and your C++ header, then use void pointers to transfer the data from AE to the kernel. Pointers work with CUDA. For Metal, the approach may differ. For OpenCL it works with a few caveats.

*Tags: `cuda`, `custom-struct`, `gpu`, `kernel`, `metal`, `opencl`*

---

### Why is checkout_layer_pixels returning data == NULL for a layer parameter in SmartRender?

In GPU mode (CUDA, Metal, OpenCL), the SDK documentation states that you must assume the data pointer is NULL in PF_EffectWorld. The pixel data lives on the GPU, not in CPU memory. This is expected behavior when GPU rendering is active.

*Tags: `checkout-layer`, `effect-world`, `gpu`, `null-data`, `smart-render`*

---

### What is the best practice when a plugin runs out of GPU memory during rendering?

Returning PF_Err_OUT_OF_MEMORY may cause AE to show a black frame without warning the user. Options: (1) Use a C++ text library to render 'Out of GPU memory' text directly onto the frame. (2) Set a static global boolean (protected by mutex) in the render thread, then display the warning in another thread like PF_UpdateParamUI. Avoid using out_data warning messages during renders as they would fail render queue operations and prevent MFR from retrying with fewer threads.

*Tags: `error-handling`, `gpu`, `mfr`, `out-of-memory`, `render-queue`*

---

### How do you use RenderDoc to debug GPU plugins in After Effects?

Use RenderDoc's in-application API with process injection. Steps: (1) Enable 'Enable process injection' in RenderDoc's Tools > Settings. (2) Compile your plugin using RenderDoc's API - in GlobalSetup, check for renderdoc.dll with GetModuleHandleA and get the API pointer. (3) Launch AE first, then inject RenderDoc via File > Inject into Process. (4) RenderDoc must be injected before applying your plugin. (5) Use rdoc_api in your render function to manually begin/end frame capture.

```cpp
// In PF_Cmd_GLOBAL_SETUP:
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll")) {
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
// Then use rdoc_api->StartFrameCapture() / EndFrameCapture() in render
```

*Tags: `debugging`, `gpu`, `opengl`, `process-injection`, `renderdoc`*

---

### Is there a skeleton/starter project for building an AE plugin with Vulkan GPU rendering?

Yes — AE_Skeleton_Vulkan by Eric CPFX (emlcpfx) is a complete CMake-based skeleton that incorporates the Vulkan SDK into an AE plugin. It also includes build scripts and a sample script for signing, notarizing, and stapling on Mac. Repository: https://github.com/emlcpfx/AE_Skeleton_Vulkan

*Tags: `build`, `cmake`, `code-signing`, `gpu`, `macos`, `skeleton`, `vulkan`*

---

### What are the main challenges with GPU plugins and MFR compatibility?

GPU plugins have significant stability issues with MFR compared to CPU plugins. The issues are hard to track down because crashes occur only after extended rendering periods. OpenGL-based plugins experience particular difficulties, especially on macOS where platform-specific issues arise.

*Tags: `debugging`, `gpu`, `mfr`, `opengl`*

---

### What cross-platform GPU technology issues exist between Windows and macOS?

OpenCL-based plugins perform well on Windows but are significantly slower on macOS. This suggests that different GPU technologies have vastly different performance characteristics across platforms, requiring careful consideration when choosing GPU frameworks.

*Tags: `cross-platform`, `gpu`, `macos`, `windows`*

---

### What is the recommended approach for supporting GPU rendering across Windows and macOS?

In the future, maintaining two completely separate code bases may be necessary: one using Metal for macOS and one using OpenGL or another framework for Windows, due to significant platform-specific GPU compatibility issues.

*Tags: `cross-platform`, `gpu`, `macos`, `metal`, `opengl`, `windows`*

---

### How do you extract and use camera and light matrix data from After Effects in a Vulkan shader to match AE's rendering?

The process involves: (1) retrieving the camera matrix from AE, (2) converting it to column-major format, (3) normalizing the position in clip space using the range (-1,1) with zoom value as a z factor, and (4) extracting and normalizing the light matrix. The challenge is ensuring proper matrix operations and coordinate space conversions match AE's internal rendering pipeline. Camera data is sent directly from AE to the shader. When working with Vulkan (instead of OpenGL), standard OpenGL operations like glPerspective or glMatrixMode are not available, so manual matrix calculations and transformations must be implemented in the shader code.

*Tags: `camera`, `gpu`, `lighting`, `matrix-math`, `shader`, `vulkan`*

---

### Why must X and Y position arguments be cast to FIX when using subpixel_sample_float, and is there a better sampling method for smoother displacement?

The subpixel_sample_float function requires FIX format for coordinate arguments even though it returns float samples. This is part of AE's API design. For smoother displacement results comparable to vanilla AE or other displacement plugins, you should review sampling best practices in the Warbler plugin tutorial, which covers proper displacement sampling techniques.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
         INT2FIX(newX),
         INT2FIX(newY),
         &giP->inSamp_pb,
         &samplePixel));
```

*Tags: `debugging`, `displacement`, `gpu`, `mfr`, `sampling`*

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

### Did you try Metal first before using MoltenVK for cross-platform GPU development?

Yes, Metal was tried first, but MoltenVK with Vulkan was chosen because it reduces code duplication between Windows and Mac platforms. However, there are some Mac-specific limitations to handle. Using Metal directly alongside DirectX/Vulkan would be like writing two separate plugins, and the developer encountered inconsistent results with SPIR-V to MSL versus SPIR-V to Vulkan without MoltenVK.

*Tags: `cross-platform`, `gpu`, `macos`, `metal`, `vulkan`, `windows`*

---

### What GPU APIs are being used for Windows and Mac support?

For cross-platform development, MoltenVK (a Vulkan layer over Metal) is used, which allows shared code between Windows and Mac. Alternatively, some developers keep OpenGL on Windows and Metal on Mac. OpenGL 4.x and 3.3 are still in use by major plugins like Element 3D and Helium, though there are concerns about Apple potentially removing OpenGL support entirely.

*Tags: `cross-platform`, `gpu`, `macos`, `metal`, `opengl`, `vulkan`, `windows`*

---

### How does the VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension avoids the need to allocate a staging buffer entirely. It allows you to turn a CPU-side void* directly into a VkDeviceMemory object, enabling direct data copying from an EffectWorld onto the GPU and allowing the GPU to write into it as well. This provides significant gains in both memory and speed, especially for large 4K frames. However, MoltenVK does not currently support this extension, though it is worth conditionally taking advantage of on Windows.

*Tags: `cross-platform`, `gpu`, `memory`, `vulkan`, `windows`*

---

### Why does device-to-device buffer transfer in Vulkan have the same speed as host-to-device transfer when using external CUDA memory?

The transfer likely implicitly bounces off host memory or the CUDA buffer may actually be on host memory rather than device memory, even though the transfer is performed device-to-device. This can happen when Vulkan cannot reliably determine how to handle the caching of imported CUDA memory handles.

*Tags: `compute-cache`, `cuda`, `gpu`, `memory`, `vulkan`*

---

### Why does checkout_layer_pixels return NULL data for a layer parameter after checkout_layer succeeds?

This appears to be a layer checkout issue where the layer parameter has valid layer data after PreRender checkout_layer calls, but checkout_layer_pixels in SmartRender returns a structure with NULL data. The issue may be related to how the layer parameter is defined or how Adobe's GPU template handles layer checkouts. Ensure the layer parameter is properly defined with PF_ADD_LAYER and that the same layer index is used consistently between PreRender and SmartRender callbacks.

```cpp
// ParamsSetup
AEFX_CLR_STRUCT(def);
PF_ADD_LAYER(LAYER_STR, PF_LayerDefault_NONE, GPU_SKELETON_LAYER_Disk_ID);

// SmartRender
ERR(extraP->cb->checkout_layer_pixels(in_data->effect_ref, GPU_SKELETON_LAYER, &env_worldP));
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `aegp`, `gpu`, `layer-checkout`, `params`, `smart-render`*

---

### How should a plugin handle GPU out-of-memory errors differently for queue renders versus interactive scrubbing?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects may retry with lower thread counts during MFR queue renders, allowing the frame to succeed. However, during interactive scrubbing or non-queue renders, After Effects may not warn the user and simply display a black frame. The plugin should only show an out_data warning message about resolution/bitdepth requirements if the user is not rendering from the queue. Unfortunately, there is no documented way to distinguish between a render request from the queue versus interactive timeline scrubbing in the plugin API.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `render-loop`*

---

### How should a plugin handle GPU out-of-memory errors and warn users appropriately?

When a plugin runs out of GPU memory, it should return PF_Err_OUT_OF_MEMORY. During MFR (Multi-Frame Rendering) queue renders, After Effects may retry with lower thread counts. However, for interactive scrubbing in the composition, After Effects may not warn users and they see black frames instead. The challenge is that there's no standard way to distinguish between queue renders and interactive scrubbing to conditionally show warning messages in out_data without failing the render. A workaround would be to return the error code and accept that After Effects' handling may be limited, or document the resolution and bitdepth requirements for users to manually adjust.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `ui`*

---

### How should a plugin handle GPU memory exhaustion errors during render versus interactive scrubbing?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects may not warn the user if they're scrubbing the timeline interactively (only black frames appear). During MFR queue renders, AE may retry with lower thread counts. One approach is to use a C++ text library to render an "Out of GPU memory" message directly onto the frame output, making the error visible to the user regardless of render context. However, there is currently no known way to distinguish between queue renders and interactive scrubbing to conditionally show warnings in out_data.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `render-loop`*

---

### Were other assignment operators like += working properly in GLSL?

The conversation indicates that /= was the problematic operator in GLSL, with the implication that other assignment operators like += were functioning correctly, though this wasn't explicitly confirmed in the replies.

*Tags: `debugging`, `gpu`, `opengl`*

---

### Why do third-party GPU debuggers crash or fail when used with After Effects?

Third-party GPU debuggers are designed for games with permanent instances, making them incompatible with plugins running inside host applications like After Effects or Premiere. Even modern API debuggers struggle with this architecture. RenderDoc, for example, tends to capture the host application's DirectX instead of the plugin's OpenGL, and gDEBugger crashes after tracing.

*Tags: `debugging`, `gpu`, `opengl`, `premiere`, `vulkan`*

---

### How can I use RenderDoc to debug GPU code in After Effects without crashing?

Use RenderDoc's In-application API by injecting it into AE after it's already loaded. First, enable process injection in Tools > Settings > "Enable process injection". Launch AE fully, then inject RenderDoc via File > Inject into Process before applying your plugin. In your plugin code, compile with the RenderDoc API by loading renderdoc.dll in PF_Cmd_GLOBAL_SETUP and retrieving the API function pointer. Then use the rdoc_api pointer in your render function to manually begin and end frame captures.

```cpp
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll"))
{
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
```

*Tags: `debugging`, `gpu`, `windows`*

---

### Can cv::mixChannels be used to convert ARGB back to BGRA to enable memcpy of entire rows?

Yes, cv::mixChannels can be used to convert between channel formats. However, a more efficient approach is to use OpenCV's Mat constructor with the step parameter (rowbytes) to create a Mat that points directly to the layer's pixel data without copying. Use cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes). Additionally, you should benchmark different approaches - mixChannels followed by memcpy might be slower than using IterateSuite (one thread per row) and doing manual channel swizzling when copying from the matrix to the layer.

```cpp
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Tags: `gpu`, `layer-checkout`, `memory`, `render-loop`*

---

### Why doesn't an Artisan-like OpenGL renderer plugin exist for After Effects?

The question was posed rhetorically but not answered directly in the conversation. The discussion suggests it would be theoretically possible to create a GPU plugin with 10k layers as sprites for real-time rendering, but no explanation was provided for why such a universal renderer hasn't been made.

*Tags: `gpu`, `opengl`, `render-loop`*

---

### How difficult would it be to match After Effects' lighting calculations in a custom 3D plugin?

This is a challenging task because it requires matching AE's specific lighting parameters and their meanings, such as spotlight falloff softness and the balance between ambient and direct lighting. Many plugins have implemented this, though the effort is significant.

*Tags: `gpu`, `metal`, `opengl`, `smartfx`*

---

### Is using f32 (32-bit float) data type safe and well-supported in After Effects plugin development?

Yes, f32 support is safe and essential for After Effects plugins, particularly those integrating 3D renderers. This has been validated through practical experience with production plugins like AtomKraft for AE and other Rust-based AE integrations that have run in production for extended periods.

*Tags: `build`, `deployment`, `gpu`, `threading`*

---

### Why might rowbytes differ in an 8-bit EffectWorld when the layer is cropped?

Rowbytes can differ based on cropping and alignment. For example, in an 8bpc EffectWorld that is cropped 25% horizontally, rowbytes could be calculated as 32*4*width, reflecting the stride needed for the allocated buffer dimensions rather than just the visible dimensions.

*Tags: `gpu`, `memory`, `output-rect`*

---

### Does the old GLator sample have memory leaks in its GL rendering and texture download functions?

Yes, the GLator sample has major memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, meaning bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBO's.

*Tags: `debugging`, `gpu`, `memory`, `opengl`*

---

### Is DirectX rendering already implemented in After Effects production, and has anyone made a plugin using it?

DirectX rendering was not widely available for plugins as of the conversation date. Adobe replaced the UI from OpenGL to DX12 in 2021, and the latest SDK has an official flag for it, but it's likely not yet ready to expose to plugins. It may be added for Windows on ARM device support, where DirectX is the first-party option. Adobe has not been supporting Vulkan, possibly because the version of AFX that passes DX12 handles to plugins hasn't been released yet.

*Tags: `apple-silicon`, `directx`, `gpu`, `sdk`, `windows`*

---

### How can DirectX compute be enabled for debugging in After Effects?

DirectX compute can be enabled by setting the 'forcedirectxcompute' flag in the console panel within AE settings. However, a full computer restart is required for the setting to take effect; restarting AE alone is not sufficient. Once properly configured, it becomes possible to debug DirectX for future productions.

*Tags: `debugging`, `directx`, `gpu`, `windows`*

---

### How slow is CPU to GPU upload and has it been benchmarked?

The question was asked but not directly answered in the conversation. Jonah asked for benchmarking data on CPU→GPU upload performance, but no specific benchmark results were provided.

*Tags: `gpu`, `memory`, `performance`*

---

### Why not create a custom GPU context in OpenGL or Vulkan instead of using After Effects' native GPU API?

Instead of relying on After Effects' limited native GPU rendering API, you can create your own GPU context using OpenGL, Vulkan, or WebGPU. This approach allows you to maintain your CPU logic while selectively sending only what's needed to the GPU, avoiding the limitations of AE's GPU world which is less developed and has fewer available suites. gabgren and Tim Constantinov successfully used this approach, eventually settling on WebGPU after trying native AE and OpenGL.

*Tags: `cross-platform`, `gpu`, `opengl`, `render-loop`, `vulkan`*

---

### What GPU APIs are recommended for After Effects plugins?

For GPU work in After Effects plugins, several options are viable: Vulkan is recommended for maximum control over performance and memory transfers, with MoltenVK providing Metal support on macOS and KosmicKrisp coming as a successor. WebGPU is easier to use and cross-platform but may not allow platform-specific optimizations. OpenGL is mature but deprecated and implemented via Metal/DirectX/Vulkan under the hood. A practical approach is to implement a CPU plugin that does GPU rendering underneath. Vulkan interoperability features allow importing textures from CUDA, OpenCL, DirectX, Metal, or OpenGL.

*Tags: `cross-platform`, `cuda`, `gpu`, `memory`, `metal`, `opengl`, `vulkan`*

---

### How can After Effects GPU features be utilized with Vulkan?

Vulkan has interoperability features that allow you to import textures from native After Effects GPU features (which may use CUDA, OpenCL, DirectX, or Metal) into Vulkan for processing, providing a way to leverage both native AE GPU capabilities and Vulkan's explicit control.

*Tags: `cuda`, `gpu`, `interop`, `metal`, `vulkan`*

---

### Why does GPU mode still run slow when the work is being done on GPU?

The performance issue occurs when critical operations fall back to CPU processing. For example, in UV unwrapping for 3D models, if a mask is present in GPU mode, the data passed is post-mask which breaks the mathematical requirements, forcing the hard work back to the CPU path, negating GPU performance benefits.

*Tags: `gpu`, `memory`, `performance`, `render-loop`*

---

### Do you use Vulkan/WebGPU for everything instead of the After Effects GPU API?

The approach varies by developer. One developer started with AE GPU, moved to OpenGL, then to WebGPU. WebGPU is preferred because the code is closer to GLSL, it supports compute shaders, and handles OS compatibility automatically (using Metal on Mac and Vulkan on PC). However, WebGPU has performance trade-offs - WGPU library adds bounds checking in shaders without a way to disable it (unless loading SPIRV shaders directly), while Google's Dawn is better performance-wise but difficult to build.

*Tags: `cross-platform`, `gpu`, `metal`, `opengl`, `vulkan`, `webgpu`*

---

### What are the performance characteristics and trade-offs of WebGPU libraries like WGPU versus alternatives?

WGPU library has good compatibility but not great performance, with unavoidable bounds checking in shaders (unless using SPIRV shaders directly). Google's Dawn offers much better performance but is difficult to build. In practice, WebGPU-based plugins are usable but not super fast - performance depends on the effect logic, such as iterating over multiple frames.

*Tags: `gpu`, `performance`, `vulkan`, `webgpu`, `wgpu`*

---

### What are the risks of encoding data into dead pixels and retrieving them with sampleImage?

The solution is dependent on render format which can be tricky. CPU and GPU have different byte orders in After Effects (ARGB vs RGBA), and GPU formats like CUDA may use BGRA. Any swizzeling or color format conversion could mangle the data. Additionally, after threading changes in AE, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering yet.

*Tags: `cuda`, `gpu`, `memory`, `render-loop`, `threading`*

---

### How can you ensure byte order doesn't affect encoding data into pixel values?

sampleImage always returns RGBA. As long as you encode each color separately and limit it to one byte per color, byte order or even bit depth won't matter.

*Tags: `gpu`, `memory`, `output-rect`*

---

### Did you test the GI rendering performance in quarter resolution?

The conversation suggests testing in quarter resolution as a next step to evaluate performance improvements, though the specific test results are not explicitly stated.

*Tags: `gpu`, `performance`, `render-loop`*

---

### What are the causes of plugins not loading in After Effects 2026?

There is a known issue on Mac where plugins from the Mediacore folder are not recognized and only load from AE's plugin folder. Additionally, AE2026 has GPU compatibility detection issues where it may only detect onboard GPU and refuse installation if it deems the GPU incompatible.

*Tags: `deployment`, `gpu`, `macos`*

---

### How do you efficiently copy image data row by row between Premiere and Vulkan/OpenGL using memcpy?

For copying from Premiere to 3D APIs (Vulkan/OpenGL), use row-by-row memcpy by calculating pointers for each row: cast the data pointers to char, offset by row byte counts, and copy one row at a time. The formula is: Void PrPtr = (Char)PRData + yPrRowByte; Void ApiPtr = (Char)ApiData + yAPIRowByte; Memcpy(ApiPtr, PrPtr, APIRowByte). The APIRowByte is calculated as width * 4 (for number of channels) * sizeOfPixel (1 for 8-bit, 4 for 32-bit). The PrRowByte comes from Premiere's definitions and can be negative. This approach works across all bitdepths. However, when copying from 3D APIs back to Premiere, crashes occur during memcpy because Premiere image buffers are 16-byte aligned with potential padding at the end of each line, so the reverse copy requires accounting for this buffer layout.

```cpp
Void PrPtr = (Char)PRData + yPrRowByte;
Void ApiPtr = (Char)ApiData + yAPIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `gpu`, `memory`, `opengl`, `premiere`, `vulkan`*

---

### How do you efficiently copy pixel data row by row when converting between Premiere and Vulkan/OpenGL?

In Premiere, use memcpy with row-based pointers: cast both Premiere and API data pointers to char, add the row offset (yPrRowByte for Premiere, yAPIRowByte for API), then memcpy the row. The APIRowByte calculation is width × NUM_channels × sizeOfPixel (1 for 8-bit, 4 for 32-bit). The PrRowByte comes from Premiere definitions and may be negative. Note that image buffers are 16-byte aligned with potential padding at line ends. In After Effects, the row-byte should already include 16-byte SIMD padding. When copying from 3D API back to Premiere causes crashes, verify the API pointer is correctly mapped and use vkGetImageSubresourceLayout in Vulkan to ensure proper source/destination image layout before copying.

```cpp
Void PrPtr = (Char)PRData + yPrRowByte;
Void ApiPtr = (Char)ApiData + yAPIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `gpu`, `memory`, `opengl`, `premiere`, `vulkan`*

---

### What are the main challenges when updating GPU plugins for MFR compatibility?

GPU plugins face significant challenges with MFR that CPU plugins don't encounter. Issues include crashes that occur only after long rendering periods (making them hard to debug), GPU caching complications, and sequence data handling. OpenGL-based plugins particularly struggle with Mac-specific issues. James Whiffin reported that even using Adobe's GLator sample, Mac OpenGL compatibility remains problematic. The recommendation is to consider maintaining completely separate codebases: one for Metal on Mac and one for Windows, as OpenGL performance on Mac can be severely degraded compared to Windows.

*Tags: `caching`, `gpu`, `macos`, `metal`, `mfr`, `opengl`, `sequence-data`, `windows`*

---

### How should you store GPU and CPU data across multiple frames in After Effects plugins?

For GPU work, store data in global data or sequence data and access it later. For CPU work, cache the PF_EffectWorld in the global or sequence data. This approach may need validation against the new 3-way-checkout mechanism.

*Tags: `caching`, `gpu`, `memory`, `sequence-data`, `smartfx`*

---

### How do you correctly extract and transform camera and light matrices from After Effects to use in a Vulkan shader?

The user is working with After Effects camera/light matrix data for a Vulkan-based plugin. Their approach involves: (1) getting the camera matrix from AE, (2) converting it to column-major format, (3) normalizing position in clip space (-1,1) using zoom as a z factor, and (4) getting and normalizing the light matrix. They note they are not using OpenGL operations like glPerspective or glMatrixMode since they are targeting Vulkan. The user indicates they've tried model and projection matrix operations but results don't match AE's render. This suggests the camera transformation pipeline in AE may require specific handling of coordinate systems, matrix conventions (row vs column major), and potentially AE-specific camera parameters that differ from standard graphics API conventions.

*Tags: `camera`, `gpu`, `matrix`, `render-loop`, `shader`, `vulkan`*

---

### Is GLM compatible with Vulkan and can it be used for matrix operations in graphics pipelines?

Yes, GLM is compatible with Vulkan and is recommended for use with it. GLM provides matrix functions like glm::perspective and glm::lookAt that can be used for graphics transformations, though note that GLM's perspective function uses gluPerspective conventions which may differ from other implementations.

*Tags: `compute-cache`, `gpu`, `reference`, `vulkan`*

---

### Is there an open-source example of volumetric fractal noise implementation for After Effects?

Yes, there is a GitHub project by mes51 that implements volumetric fractal noise: https://github.com/mes51/VolumetricFractalNoise. This project serves as a reference implementation and workaround for volumetric fractal noise effects in After Effects plugins.

*Tags: `gpu`, `open-source`, `reference`, `rendering`*

---

### Is there an open-source example of a Vulkan-based After Effects plugin?

Yes, Wunkolo has open-sourced Vulkanator, a Vulkan-based After Effects plugin project with sample code. This project represents an early push toward more open-source resources in the After Effects plugin development space. Repository: https://github.com/Wunkolo/Vulkanator

*Tags: `gpu`, `open-source`, `reference`, `vulkan`*

---

### What is an efficient algorithm for creating edge outlines or choker mattes based on alpha channels?

Instead of checking neighbors for each transparent pixel (which results in O(n^2) complexity), use a distance field approach. Create an intermediate 2D buffer of signed distance values (e.g., int16_t) to exchange memory for speed. The Jump Flooding Algorithm (JFA) is a fast method for generating distance fields and is well-suited for this exact use case.

*Tags: `algorithm`, `gpu`, `matte`, `memory`, `performance`*

---

### How is it possible for a customer to have a newer GPU driver version than me on an older macOS version?

On Apple systems, drivers are typically bundled with the OS, making newer drivers on older OS versions unusual. One possibility is manual driver installation similar to Hackintosh setups, though this is not straightforward. Additionally, Apple may have issued driver downgrades in response to GPU issues in specific macOS versions (e.g., macOS 12.3 had known issues with AMD 5xxx and 6xxx GPUs). See: https://www.reddit.com/r/hackintosh/comments/ter3g2/psa_macos_monterey_123_and_amd_5xxx_and_6xxx_gpu/

*Tags: `debugging`, `gpu`, `macos`*

---

### Can you use Metal C++ bindings instead of Objective-C for After Effects plugin development on macOS?

James Whiffin asked about using Metal C++ bindings as an alternative to Objective-C for AE plugin development, noting that C++ would be more familiar for developers without Objective-C experience. This suggests that Metal C++ bindings are a viable option for macOS plugin development, offering a more accessible path for C++-focused developers.

*Tags: `cross-platform`, `gpu`, `macos`, `metal`*

---

### Should you use Metal directly or MoltenVK for cross-platform GPU plugin development on macOS?

Using Vulkan with MoltenVK (a Vulkan layer over Metal) reduces code duplication between Windows and macOS compared to maintaining separate Metal and DirectX implementations. However, there are some macOS-specific limitations to account for. Alternatively, keeping OpenGL on Windows and Metal on macOS is viable, but carries the risk that Apple may eventually remove OpenGL support entirely, as they have deprecated it in favor of Metal.

*Tags: `cross-platform`, `gpu`, `macos`, `metal`, `opengl`, `vulkan`, `windows`*

---

### What GPU APIs are commonly used by commercial After Effects plugins?

Popular commercial AE plugins like Element 3D and Helium use OpenGL (Element 3D uses OpenGL 4.x and Helium uses OpenGL 3.3). Many plugin developers continue to rely on OpenGL support on macOS despite its deprecation, hoping that Apple will prepare alternatives before removing it entirely.

*Tags: `gpu`, `macos`, `opengl`, `reference`*

---

### What are the limitations of MoltenVK for After Effects plugin development?

MoltenVK before version 1.3 had several limitations including unsupported texture swizzle and uniforms limitations. However, these issues have been resolved in MoltenVK 1.3 and later versions.

*Tags: `gpu`, `macos`, `metal`, `moltenvk`, `vulkan`*

---

### How can VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension (available on Windows) avoids the need to allocate a staging buffer by allowing a CPU-side void* to be turned directly into a VkDeviceMemory object. This enables copying data directly from an EffectWorld onto the GPU and allows the GPU to write back into it without redundant upload/download copies, resulting in significant gains in both memory and speed, especially for large 4K frames.

*Tags: `gpu`, `memory`, `optimization`, `vulkan`, `windows`*

---

### What is an example of an After Effects plugin using Vulkan?

ScaleUp is an AE script/plugin that uses Vulkan and MoltenVK for rendering. It can be found at https://aescripts.com/scaleup/

*Tags: `gpu`, `moltenvk`, `open-source`, `reference`, `vulkan`*

---

### What OpenGL versions do popular After Effects effects use, and which ones work well?

Based on developer feedback, popular AE effects use varying OpenGL versions. Element 3D and Trapcode use OpenGL 4.x, while other effects like Helium maintain OpenGL 3.3. The questioner notes they previously used OpenGL 3.3 before migrating to Vulkan. OpenGL 3.3 appears to be a stable choice used by multiple working effects.

*Tags: `cross-platform`, `gpu`, `opengl`, `reference`*

---

### Is there an example of integrating Vulkan with After Effects on Apple Silicon?

Wunk is updating the Vulkanator sample project to demonstrate VulkanAPI (via MoltenVK) integration with After Effects on Apple Silicon, showing how to leverage Vulkan for GPU-accelerated rendering on macOS ARM64 systems.

*Tags: `apple-silicon`, `gpu`, `metal`, `open-source`, `vulkan`*

---

### Why is device-to-device buffer transfer between Vulkan and CUDA no faster than host-to-device transfer?

When importing a CUDA handle into Vulkan for external memory, the transfer may implicitly bounce through host memory if Vulkan cannot reliably determine how to handle the memory's caching. Additionally, the AE_cuda buffer may be allocated on the host rather than the device, which would explain why device-to-device transfer speeds match host-to-device speeds. Check whether the buffer is actually allocated on the device or if it's a host-side allocation.

*Tags: `cross-platform`, `cuda`, `gpu`, `memory`, `vulkan`*

---

### Is there an example After Effects plugin that uses Vulkan for GPU acceleration?

The Vulkanator project by Wunkolo demonstrates Vulkan integration with After Effects plugins for GPU-accelerated rendering. Repository: https://github.com/Wunkolo/Vulkanator

*Tags: `gpu`, `open-source`, `reference`, `vulkan`*

---

### Why is checkout_layer_pixels returning NULL data for a layer parameter even though the layer is selected?

Tim Constantinov reported an issue where after calling checkout_layer_pixels on a layer selected in a layer parameter, the data field equals NULL. The problem occurs in SmartRender after checkout_layer_pixels is called on GPU_SKELETON_LAYER. The issue appears to be intermittent and may be related to the Adobe-supplied GPU template. The exact cause wasn't definitively identified in the conversation, but Tim noted this issue resolved itself spontaneously in the past.

```cpp
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `debugging`, `gpu`, `layer-checkout`, `params`, `smartfx`*

---

### What should you assume about the data pointer in pF_effectworld when using GPU mode in After Effects plugins?

According to the SDK documentation, when using GPU mode (CUDA, Metal, or OpenCL), you must assume that the data pointer is NULL in the pF_effectworld structure. This means the pixel data is not available in system memory and you need to work with GPU-resident data instead.

*Tags: `cuda`, `gpu`, `metal`, `mfr`, `opencl`, `reference`*

---

### How can a plugin distinguish between a render queue render and a user scrubbing the timeline in the composition to handle GPU memory errors appropriately?

James Whiffin raised this as an unsolved problem in AE plugin development. When a plugin returns PF_Err_OUT_OF_MEMORY, AE may not warn the user if they're scrubbing the timeline (resulting in a black frame), but the plugin cannot reliably distinguish between queue renders and interactive scrubbing. While returning the error code prevents render queue failures (allowing AE to retry with lower thread counts), there's currently no documented method to detect the render context and conditionally display user warnings via out_data. This remains a limitation when plugins need multiple 32bpc GPU buffers at high resolutions.

*Tags: `debugging`, `gpu`, `memory`, `mfr`*

---

### How should a plugin handle GPU out-of-memory errors differently for interactive scrubbing versus render queue operations?

When a plugin returns PF_Err_OUT_OF_MEMORY, After Effects handles it differently depending on context. During MFR (Multi-Frame Rendering) queue renders, AE may retry with a lower thread count. However, during interactive timeline scrubbing, AE doesn't warn the user and may display a black frame instead. The challenge is that PF_Err_OUT_OF_MEMORY warnings in out_data will fail the render, and there is currently no documented method to distinguish between a render queue request versus interactive scrubbing. Plugins must accept that they cannot reliably warn users about resolution/bitdepth being too high for available GPU memory in interactive contexts without failing the frame.

*Tags: `debugging`, `gpu`, `memory`, `mfr`, `smartfx`*

---

### What is the best practice for handling GPU out-of-memory errors in After Effects plugins?

When a plugin runs out of GPU memory, it should return PF_Err_OUT_OF_MEMORY. During MFR (multi-frame rendering), After Effects may retry the frame with a lower thread count. For interactive scrubbing where the user isn't rendering from the queue, one approach is to use a C++ text library to render an error message directly onto the frame (e.g., "Out of GPU memory") so the user understands why a black frame appeared. However, there is currently no reliable way to distinguish between a render queue request and interactive timeline scrubbing to conditionally show warnings in out_data.

*Tags: `gpu`, `memory`, `mfr`, `output-rect`, `render-loop`*

---

### What operator caused a GPU driver issue in After Effects plugin development?

Using the /= operator instead of the longhand division assignment can cause GPU driver issues. James Whiffin reported a ticket where users' GPU drivers failed due to this operator choice, suggesting that drivers may have inconsistent support for compound assignment operators.

*Tags: `cross-platform`, `debugging`, `gpu`*

---

### What GLSL assignment operators are supported across different GPU platforms in After Effects plugins?

James Whiffin encountered an issue where a specific GLSL operator caused compilation failures. The error was identified through shader compilation feedback. The conversation indicates that while some assignment operators like += work correctly, the /= operator caused issues on at least one platform (Windows or Mac, GPU reference unknown). This suggests developers need to be cautious about which GLSL assignment operators are universally supported across different GPU architectures when developing AE plugins.

*Tags: `cross-platform`, `debugging`, `glsl`, `gpu`, `macos`, `opengl`, `windows`*

---

### What are the challenges with using third-party GPU debuggers for After Effects plugins?

GPU debuggers like gDEBugger, RenderDoc, and others frequently crash or fail when tracing After Effects. The core issue is that these debuggers are designed for standalone games with permanent instances, whereas plugin architectures within host applications like After Effects or Premiere Pro create a more complex environment. Additionally, debuggers may capture the host application's graphics API (DirectX) instead of the plugin's rendering API (OpenGL, Vulkan), making them ineffective for plugin-specific debugging.

*Tags: `debugging`, `gpu`, `opengl`, `plugin-architecture`, `vulkan`*

---

### How can I use RenderDoc to debug GPU code in After Effects plugins without crashes?

Use RenderDoc's in-application API with process injection. Enable process injection in Tools > Settings > "Enable process injection" (hidden by default). Launch AE fully, then inject RenderDoc via File > Inject into Process before applying your plugin. In your plugin's PF_Cmd_GLOBAL_SETUP, load the RenderDoc API and use the rdoc_api pointer to manually begin and end frame captures in your render function. This approach avoids the crashes that occur when launching RenderDoc normally with AE.

```cpp
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll"))
{
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
```

*Tags: `debugging`, `gpu`, `render-loop`, `windows`*

---

### How can you create an OpenCV Mat that points to existing pixel data with custom row stride?

Use the cv::Mat constructor with the step parameter to specify custom row bytes. Example: cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes); This allows OpenCV to work directly with After Effects pixel buffers without copying data.

```cpp
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Tags: `caching`, `gpu`, `memory`, `opencv`*

---

### What are the limitations of working with thousands of layers in After Effects, and how do plugins like Newton handle this?

After Effects struggles with performance even at 512 layers. The user is asking whether Newton (a physics simulation plugin) encounters the same scaling issues when simulating many layers, and considering whether switching to GPU-based sprite rendering would be necessary to maintain flexibility while handling 10,000 layer instances.

*Tags: `ae-limits`, `gpu`, `memory`, `performance`, `render-loop`*

---

### Why hasn't someone created an OpenGL-based realtime GPU renderer plugin for After Effects that can handle thousands of layers as sprites?

This is a theoretical discussion about a potential feature gap in After Effects plugin ecosystem. The concept would involve creating a custom 3D renderer plugin that could leverage GPU acceleration to render 10,000+ layers in realtime by treating them as sprites, similar to what exists in other software like Artisan. No concrete answer or existing solution was provided in the conversation.

*Tags: `gpu`, `opengl`, `reference`, `render-loop`*

---

### How can you leverage After Effects' rendering engine (3D lights, effects) while maintaining control over individual layers in a plugin?

One approach is to use the GPU route with sprites while keeping flexibility for individual layers. This allows you to take advantage of AE's built-in rendering engine capabilities like 3D lights and effects on each layer, rather than trying to replicate these features manually in your own 3D plugin.

*Tags: `3d`, `effects`, `gpu`, `layer-checkout`, `render-loop`*

---

### What is the challenge in matching After Effects' lighting calculations in a custom 3D plugin?

The main challenge is making all parameters mean the same thing with the same units as AE's native lighting system. This includes matching properties like spotlight falloff softness and the balance between ambient and direct lighting. While many plugins attempt this, it is complex to replicate precisely.

*Tags: `3d`, `effects`, `gpu`, `parameters`*

---

### How can you detect the After Effects version from within a plugin?

You can use ExtendScript with 'app.version' via the AEGP execute script. Alternatively, if your plugin supports both After Effects and Premiere Pro with GPU acceleration, you have access to a Premiere Pro PICA Suite AppInfo that provides the version. The version data is in format like 24.3, and you need to add 2000 to get the actual version number (e.g., 24.3 becomes 2024.3). Some versions may include additional components like 24.3.x which require parsing.

*Tags: `aegp`, `gpu`, `premiere`, `scripting`, `version-detection`*

---

### Does the old GLator sample have memory leaks in its GL rendering code?

Yes, the GLator sample has significant memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, which means bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBOs.

*Tags: `deprecated`, `gpu`, `memory`, `opengl`, `reference`*

---

### Is DirectX rendering currently implemented in After Effects production, and how can it be enabled for plugin development?

DirectX rendering support was added to the After Effects SDK with an official flag, though it appears to be in early stages. Adobe replaced the UI from OpenGL to DX12 in 2021. To enable DirectX compute for debugging and testing, use the 'forcedxcompute' setting in the After Effects console panel under settings, but note that a full computer restart (not just AE restart) is required for the setting to take effect. The feature is likely being developed primarily for Windows on ARM device support, as those devices have first-party support for DX12. Adobe has not yet officially supported Vulkan exposure to plugins.

*Tags: `apple-silicon`, `debugging`, `directx`, `gpu`, `windows`*

---

### What are alternative GPU APIs for communicating with After Effects to access GPU data?

Vulkan, CUDA, and Metal are viable options for GPU communication with After Effects. Developers have successfully implemented CUDA and Metal to get GPU data from AE in GPU mode. Vulkan is recommended as a cross-platform alternative to DirectX, though Adobe has not officially supported Vulkan exposure to plugins at this time.

*Tags: `cross-platform`, `cuda`, `gpu`, `metal`, `vulkan`*

---

### How can you apply masks to GPU-rendered output in After Effects when the GPU API only provides masked/cropped buffers?

Eric encountered this limitation while developing the Wrap It 3D camera projection plugin. After Effects' GPU rendering API (checkout_layer_pixels) only returns post-mask-applied, cropped buffers with no GPU equivalent of PF_CHECKOUT_PARAM to access the raw unmasked source. Attempted workarounds included using PF_OutFlag2_REVEALS_ZERO_ALPHA with expanded PreRender requests, but this didn't provide the full unmasked layer to GPU. A manual CPU→GPU upload of the unmasked source defeats GPU acceleration benefits. The core limitation is that AE's native GPU world has no mechanism to provide raw unmasked source data on GPU. As a workaround, users may need to use track mattes in GPU mode, or consider building a custom GPU context (OpenGL, Vulkan, or WebGPU) outside of After Effects' native GPU pipeline to maintain full control over the source data.

*Tags: `aegp`, `gpu`, `layer-checkout`, `masks`, `render-loop`*

---

### Should you use After Effects' native GPU API or create a custom GPU context for complex effects that need full source access?

According to Gabgren and Tim Constantinov's experience, After Effects' native GPU world is less developed with not all features available. For effects requiring full control over source data and GPU acceleration, it's better to create your own GPU context using OpenGL, Vulkan, or WebGPU rather than relying on AE's built-in GPU pipeline. This approach allows you to keep CPU-side operations intact and selectively send only necessary data to your custom GPU context. Tim Constantinov's team experimented with native AE GPU and OpenGL before settling on WebGPU, and Wunk has had positive results with Vulkan.

*Tags: `aegp`, `cross-platform`, `gpu`, `opengl`, `vulkan`, `webgpu`*

---

### What GPU API should I use for After Effects plugins and how do they compare?

Vulkan is highly recommended for maximum control over performance and memory management. It's explicit and detailed, allowing fine-tuning of memory transfers and format conversions. On macOS, MoltenVK (originally by Valve) automatically ports Vulkan features to Metal, with Khronos developing KosmicKrisp as its successor. WebGPU is easier to use and cross-platform but may not provide access to platform-specific optimizations. OpenGL is mature but deprecated—it's now typically implemented via Metal, DirectX, or Vulkan under the hood on all platforms. Vulkan also supports interoperability with CUDA, OpenCL, DirectX, Metal, and OpenGL, allowing you to import textures from After Effects' native GPU features into Vulkan.

*Tags: `cross-platform`, `gpu`, `metal`, `opengl`, `performance`, `vulkan`*

---

### How much faster is GPU rendering compared to CPU path for plugin operations?

According to benchmarks, there is a significant difference: GPU processing can complete operations in milliseconds while the CPU path takes several seconds. However, performance depends on the actual operations being offloaded—if critical work like UV unwrapping still falls back to CPU, the GPU advantage is negated.

*Tags: `benchmark`, `cpu`, `gpu`, `optimization`, `performance`*

---

### What approach works well for implementing GPU acceleration in After Effects plugins?

A practical approach is to present your plugin as a CPU plugin to After Effects while implementing GPU rendering (Vulkan, OpenGL, or WebGPU) under the hood. This avoids compatibility issues while still leveraging GPU acceleration. Ensure that critical operations are actually executed on the GPU—falling back to CPU paths defeats the purpose.

*Tags: `gpu`, `optimization`, `rendering`, `vulkan`, `webgpu`*

---

### What are the tradeoffs between different GPU frameworks for After Effects plugins?

According to developers in the community, the progression has been: AE GPU → OpenGL → WebGPU. WebGPU is preferred because the code is closer to GLSL, it supports compute shaders, and it handles OS compatibility automatically (Metal for Mac, Vulkan for PC under the hood). However, WebGPU implementations have performance considerations: WGPU library is nice for compatibility but not great for performance, while Google's Dawn is much better performance-wise but difficult to build. A notable limitation of WebGPU is that it adds bounds checking in shaders without a way to disable it unless you load SPIRV shaders directly.

*Tags: `cross-platform`, `gpu`, `metal`, `opengl`, `vulkan`, `webgpu`*

---

### Is there a recommended WebGPU library for building After Effects GPU plugins?

The WGPU library is commonly used for WebGPU support in AE plugins due to its good cross-platform compatibility (automatically using Metal on Mac and Vulkan on PC). However, Google's Dawn is noted as having better performance characteristics, though it is more difficult to build and integrate.

*Tags: `cross-platform`, `gpu`, `tool`, `webgpu`*

---

### What GPU rendering approach did Red Giant Universe migrate to from CPU-only plugins?

Red Giant Universe transitioned from CPU-only plugins with OpenGL in the background to a custom GPU renderer backend that supports Metal, DirectX, and OpenGL. They now provide GPU buffer interop for every GPU buffer type available, allowing plugins to leverage native GPU rendering across different platforms.

*Tags: `cross-platform`, `gpu`, `metal`, `opengl`, `render-loop`*

---

### Should developers use the After Effects GPU Suite for plugin development?

No, the AE GPU Suite should be avoided for plugin development. Instead, developers should implement custom GPU renderer backends that support Metal, DirectX, and OpenGL with proper GPU buffer interoperability, as demonstrated by modern plugin architectures like Red Giant Universe's approach.

*Tags: `best-practices`, `debugging`, `gpu`, `metal`, `opengl`*

---

### How do you efficiently copy pixel data row-by-row between Premiere and Vulkan/OpenGL APIs?

In Premiere, use memcpy with row-based copying by calculating pointers for each row: `void* PrPtr = (char*)PRData + y*PrRowByte; void* ApiPtr = (char*)ApiData + y*APIRowByte; memcpy(ApiPtr, PrPtr, APIRowByte);` where APIRowByte is calculated as `width * 4 * sizeOfPixel` (1 for 8-bit, 4 for 32-bit). PrRowByte comes from Premiere and may be negative. This approach works for reading from Premiere, but copying back to Premiere can cause crashes. For Vulkan specifically, use `vkGetImageSubresourceLayout()` to get the actual `VkSubresourceLayout` of source/destination images to ensure correct copying, as the layout may not always be packed. In After Effects, row-byte already includes 16-byte SIMD padding, so verify buffer alignment and that API pointers are correctly mapped to avoid access violations.

```cpp
void* PrPtr = (char*)PRData + y*PrRowByte;
void* ApiPtr = (char*)ApiData + y*APIRowByte;
memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `gpu`, `memory`, `opengl`, `premiere`, `vulkan`*

---

### How do you prevent video memory leaks when using OpenGL in After Effects plugins, especially during UI parameter changes?

The issue is typically caused by exceptions thrown within OpenGL calls during an interrupt, which causes the code to skip glDeleteTextures calls and jump to catch blocks. To fix this, ensure that glDeleteTextures is called before checking for interrupts using PF_ABORT(), and put breakpoints in catch blocks to verify if exceptions are being thrown during user interaction. The proper order should be: perform GPU operations, delete textures, unbind framebuffers and textures, then check for abort conditions and release suites.

```cpp
glBindFramebuffer(GL_FRAMEBUFFER, 0);
glBindTexture(GL_TEXTURE_2D, 0);
glDeleteTextures(1, &inputFrameTexture);
glDeleteTextures(1, &inputFrameTexture2);
// Then check for abort after cleanup
ERR(PF_ABORT(in_data));
```

*Tags: `debugging`, `gpu`, `memory`, `opengl`, `plugin-development`*

---

### Is there a sample demonstrating OpenGL usage in After Effects plugins?

Yes, Adobe provides the GLator sample as a reference for using OpenGL in After Effects plugins. This sample can help developers understand how to integrate OpenGL drawing capabilities into their plugins.

*Tags: `gpu`, `opengl`, `reference`, `sample`*

---
