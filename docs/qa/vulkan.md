# Q&A: vulkan

**45 entries** tagged with `vulkan`.

---

## How do I copy pixel data row by row between Premiere buffers and a 3D API (Vulkan/OpenGL), and why does memcpy crash when copying from the 3D API back to Premiere?

For copying from Premiere to the 3D API, iterate row by row using the row bytes from the Premiere world definition. The crash when copying back likely relates to incorrect buffer mapping or row byte calculations. In Premiere, row bytes can be negative, and image buffers are 16-byte aligned with possible padding at the end of each line. For Vulkan specifically, check the actual VkSubresourceLayout via vkGetImageSubresourceLayout to ensure correct copy offsets. An access violation usually means a calculation is wrong or one of the buffers is not mapped correctly.

```cpp
// Premiere to 3D API (works):
Void PrPtr = (Char)PRData + y * PrRowByte;
Void ApiPtr = (Char)ApiData + y * APIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);

// APIRowByte = width * 4 (channels) * sizeOfPixel (1 for 8-bit, 4 for 32-bit)
// PrRowByte comes from Premiere world def, often negative
```

*Contributors: [**wunk**](../contributors/wunk/) · Source: adobe-plugin-devs · 2023-03-04 · Tags: `premiere`, `memcpy`, `row-bytes`, `vulkan`, `opengl`, `pixel-buffer`, `negative-row-bytes`*

---

## Is there a skeleton/starter project for building an AE plugin with Vulkan GPU rendering?

Yes — AE_Skeleton_Vulkan by Eric CPFX (emlcpfx) is a complete CMake-based skeleton that incorporates the Vulkan SDK into an AE plugin. It also includes build scripts and a sample script for signing, notarizing, and stapling on Mac. Repository: https://github.com/emlcpfx/AE_Skeleton_Vulkan

*Source: curated · Tags: `vulkan`, `gpu`, `cmake`, `build`, `macos`, `code-signing`, `skeleton`*

---

## How do you correctly extract and convert After Effects camera and light matrix data for use in a Vulkan shader?

The user describes a 4-step process: (1) get camera matrix from AE, (2) convert it to column-major format, (3) normalize position in clip space using range (-1,1) with zoom value as z factor, (4) get light matrix and normalize it. However, they report that using model and projection matrix operations hasn't produced expected results. The answer suggests this is a known challenge when working with AE camera data in non-OpenGL contexts like Vulkan, and the correct transformation depends on properly handling the coordinate system conversion between AE's internal representation and the target graphics API.

*Tags: `camera`, `matrix`, `vulkan`, `shader`, `coordinates`, `lighting`*

---

## How do you extract and use camera and light matrix data from After Effects in a Vulkan shader to match AE's rendering?

The process involves: (1) retrieving the camera matrix from AE, (2) converting it to column-major format, (3) normalizing the position in clip space using the range (-1,1) with zoom value as a z factor, and (4) extracting and normalizing the light matrix. The challenge is ensuring proper matrix operations and coordinate space conversions match AE's internal rendering pipeline. Camera data is sent directly from AE to the shader. When working with Vulkan (instead of OpenGL), standard OpenGL operations like glPerspective or glMatrixMode are not available, so manual matrix calculations and transformations must be implemented in the shader code.

*Tags: `vulkan`, `gpu`, `camera`, `matrix-math`, `shader`, `lighting`*

---

## Did you try Metal first before using MoltenVK for cross-platform GPU development?

Yes, Metal was tried first, but MoltenVK with Vulkan was chosen because it reduces code duplication between Windows and Mac platforms. However, there are some Mac-specific limitations to handle. Using Metal directly alongside DirectX/Vulkan would be like writing two separate plugins, and the developer encountered inconsistent results with SPIR-V to MSL versus SPIR-V to Vulkan without MoltenVK.

*Tags: `gpu`, `metal`, `vulkan`, `cross-platform`, `macos`, `windows`*

---

## What GPU APIs are being used for Windows and Mac support?

For cross-platform development, MoltenVK (a Vulkan layer over Metal) is used, which allows shared code between Windows and Mac. Alternatively, some developers keep OpenGL on Windows and Metal on Mac. OpenGL 4.x and 3.3 are still in use by major plugins like Element 3D and Helium, though there are concerns about Apple potentially removing OpenGL support entirely.

*Tags: `gpu`, `opengl`, `metal`, `vulkan`, `cross-platform`, `macos`, `windows`*

---

## What are the limitations of MoltenVK before version 1.3?

MoltenVK before VK 1.3 had some limitations including unsupported texture swizzle and uniforms limitations, but these issues are now resolved in version 1.3 and later.

*Tags: `vulkan`, `metal`, `moltenvk`, `macos`*

---

## How does the VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension avoids the need to allocate a staging buffer entirely. It allows you to turn a CPU-side void* directly into a VkDeviceMemory object, enabling direct data copying from an EffectWorld onto the GPU and allowing the GPU to write into it as well. This provides significant gains in both memory and speed, especially for large 4K frames. However, MoltenVK does not currently support this extension, though it is worth conditionally taking advantage of on Windows.

*Tags: `vulkan`, `memory`, `gpu`, `windows`, `cross-platform`*

---

## How can I embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications using MoltenVK?

You can embed MoltenVK in the plugin's framework folder as a dynamic library with the JSON configuration in resources/vulkan. However, this path approach doesn't work reliably for plugin bundles. An alternative is to use xcframework to embed it as a static library inside the plugin binary, similar to what the scaleUp plugin does. If Vulkan fails to create instances when using the static library approach, verify that the Vulkan layer configuration and initialization is correct for the statically-linked library within the plugin bundle context.

*Tags: `vulkan`, `macos`, `deployment`, `build`, `cross-platform`*

---

## How can MoltenVK be embedded inside an After Effects plugin bundle to avoid conflicts with other MoltenVK applications?

MoltenVK can be embedded as a static library using xcframework within the plugin binary, similar to how the scaleUp plugin does it. However, this approach requires careful configuration of the Vulkan loader and MoltenVK ICD to ensure proper function pointer resolution and layer/extension loading. The main challenge is ensuring the Vulkan loader correctly locates the MoltenVK ICD when statically linked, as opposed to using the standard /etc/vulkan or /usr/local installation paths. Testing should verify that the Vulkan loader is correctly configured for the static linking approach.

*Tags: `macos`, `vulkan`, `build`, `deployment`, `cross-platform`*

---

## What causes VK_ERROR_LAYER_NOT_PRESENT and VK_ERROR_EXTENSION_NOT_PRESENT errors when creating Vulkan instances with embedded MoltenVK in a plugin?

These errors typically indicate that the MoltenVK ICD is not being located correctly by the Vulkan loader, or there are environment configuration issues. When MoltenVK is statically embedded rather than installed system-wide, the Vulkan loader may not find the necessary validation layers and extensions. This is particularly challenging in sandboxed or non-standard installations where the LunarG Vulkan SDK is not in the default /usr/local path. The solution involves ensuring the Vulkan loader can properly locate the MoltenVK ICD configuration.

*Tags: `macos`, `vulkan`, `debugging`, `deployment`*

---

## How do you set up MoltenVK compatibility with Vulkan 1.2 on macOS for After Effects plugins?

Get at least the October update to get a MoltenVK compatible with Vulkan 1.2. For debug builds, create a target using ICD in environment with dylib linking, add "VK_KHR_portability_subset" to instance extensions (debug only for now until MoltenVK uses it), and include debugging layers. For release builds, link to frameworks only as static libraries to work within Vulkan 1.2 limitations.

*Tags: `vulkan`, `macos`, `apple-silicon`, `build`, `debugging`*

---

## Why does device-to-device buffer transfer in Vulkan have the same speed as host-to-device transfer when using external CUDA memory?

The transfer likely implicitly bounces off host memory or the CUDA buffer may actually be on host memory rather than device memory, even though the transfer is performed device-to-device. This can happen when Vulkan cannot reliably determine how to handle the caching of imported CUDA memory handles.

*Tags: `vulkan`, `cuda`, `gpu`, `memory`, `compute-cache`*

---

## Why do third-party GPU debuggers crash or fail when used with After Effects?

Third-party GPU debuggers are designed for games with permanent instances, making them incompatible with plugins running inside host applications like After Effects or Premiere. Even modern API debuggers struggle with this architecture. RenderDoc, for example, tends to capture the host application's DirectX instead of the plugin's OpenGL, and gDEBugger crashes after tracing.

*Tags: `gpu`, `debugging`, `opengl`, `vulkan`, `premiere`*

---

## Why not create a custom GPU context in OpenGL or Vulkan instead of using After Effects' native GPU API?

Instead of relying on After Effects' limited native GPU rendering API, you can create your own GPU context using OpenGL, Vulkan, or WebGPU. This approach allows you to maintain your CPU logic while selectively sending only what's needed to the GPU, avoiding the limitations of AE's GPU world which is less developed and has fewer available suites. gabgren and Tim Constantinov successfully used this approach, eventually settling on WebGPU after trying native AE and OpenGL.

*Tags: `gpu`, `opengl`, `vulkan`, `render-loop`, `cross-platform`*

---

## What GPU APIs are recommended for After Effects plugins?

For GPU work in After Effects plugins, several options are viable: Vulkan is recommended for maximum control over performance and memory transfers, with MoltenVK providing Metal support on macOS and KosmicKrisp coming as a successor. WebGPU is easier to use and cross-platform but may not allow platform-specific optimizations. OpenGL is mature but deprecated and implemented via Metal/DirectX/Vulkan under the hood. A practical approach is to implement a CPU plugin that does GPU rendering underneath. Vulkan interoperability features allow importing textures from CUDA, OpenCL, DirectX, Metal, or OpenGL.

*Tags: `gpu`, `vulkan`, `metal`, `opengl`, `cuda`, `cross-platform`, `memory`*

---

## How can After Effects GPU features be utilized with Vulkan?

Vulkan has interoperability features that allow you to import textures from native After Effects GPU features (which may use CUDA, OpenCL, DirectX, or Metal) into Vulkan for processing, providing a way to leverage both native AE GPU capabilities and Vulkan's explicit control.

*Tags: `gpu`, `vulkan`, `metal`, `cuda`, `interop`*

---

## Do you use Vulkan/WebGPU for everything instead of the After Effects GPU API?

The approach varies by developer. One developer started with AE GPU, moved to OpenGL, then to WebGPU. WebGPU is preferred because the code is closer to GLSL, it supports compute shaders, and handles OS compatibility automatically (using Metal on Mac and Vulkan on PC). However, WebGPU has performance trade-offs - WGPU library adds bounds checking in shaders without a way to disable it (unless loading SPIRV shaders directly), while Google's Dawn is better performance-wise but difficult to build.

*Tags: `gpu`, `webgpu`, `vulkan`, `metal`, `opengl`, `cross-platform`*

---

## What are the performance characteristics and trade-offs of WebGPU libraries like WGPU versus alternatives?

WGPU library has good compatibility but not great performance, with unavoidable bounds checking in shaders (unless using SPIRV shaders directly). Google's Dawn offers much better performance but is difficult to build. In practice, WebGPU-based plugins are usable but not super fast - performance depends on the effect logic, such as iterating over multiple frames.

*Tags: `gpu`, `webgpu`, `wgpu`, `performance`, `vulkan`*

---

## How do you efficiently copy image data row by row between Premiere and Vulkan/OpenGL using memcpy?

For copying from Premiere to 3D APIs (Vulkan/OpenGL), use row-by-row memcpy by calculating pointers for each row: cast the data pointers to char, offset by row byte counts, and copy one row at a time. The formula is: Void PrPtr = (Char)PRData + yPrRowByte; Void ApiPtr = (Char)ApiData + yAPIRowByte; Memcpy(ApiPtr, PrPtr, APIRowByte). The APIRowByte is calculated as width * 4 (for number of channels) * sizeOfPixel (1 for 8-bit, 4 for 32-bit). The PrRowByte comes from Premiere's definitions and can be negative. This approach works across all bitdepths. However, when copying from 3D APIs back to Premiere, crashes occur during memcpy because Premiere image buffers are 16-byte aligned with potential padding at the end of each line, so the reverse copy requires accounting for this buffer layout.

```cpp
Void PrPtr = (Char)PRData + yPrRowByte;
Void ApiPtr = (Char)ApiData + yAPIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `premiere`, `vulkan`, `opengl`, `memory`, `gpu`*

---

## How do you efficiently copy pixel data row by row when converting between Premiere and Vulkan/OpenGL?

In Premiere, use memcpy with row-based pointers: cast both Premiere and API data pointers to char, add the row offset (yPrRowByte for Premiere, yAPIRowByte for API), then memcpy the row. The APIRowByte calculation is width × NUM_channels × sizeOfPixel (1 for 8-bit, 4 for 32-bit). The PrRowByte comes from Premiere definitions and may be negative. Note that image buffers are 16-byte aligned with potential padding at line ends. In After Effects, the row-byte should already include 16-byte SIMD padding. When copying from 3D API back to Premiere causes crashes, verify the API pointer is correctly mapped and use vkGetImageSubresourceLayout in Vulkan to ensure proper source/destination image layout before copying.

```cpp
Void PrPtr = (Char)PRData + yPrRowByte;
Void ApiPtr = (Char)ApiData + yAPIRowByte;
Memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `vulkan`, `opengl`, `premiere`, `memory`, `gpu`*

---

## How do you correctly extract and transform camera and light matrices from After Effects to use in a Vulkan shader?

The user is working with After Effects camera/light matrix data for a Vulkan-based plugin. Their approach involves: (1) getting the camera matrix from AE, (2) converting it to column-major format, (3) normalizing position in clip space (-1,1) using zoom as a z factor, and (4) getting and normalizing the light matrix. They note they are not using OpenGL operations like glPerspective or glMatrixMode since they are targeting Vulkan. The user indicates they've tried model and projection matrix operations but results don't match AE's render. This suggests the camera transformation pipeline in AE may require specific handling of coordinate systems, matrix conventions (row vs column major), and potentially AE-specific camera parameters that differ from standard graphics API conventions.

*Tags: `camera`, `vulkan`, `matrix`, `shader`, `render-loop`, `gpu`*

---

## How do you correctly extract and transform camera and light matrix data from After Effects for use in a Vulkan shader?

The process involves several steps: (1) Get the camera matrix from After Effects, (2) Convert it to column-major format, (3) Normalize the position in clip space using the range (-1, 1) and apply the zoom value as a z-factor, (4) Get the light matrix and normalize it similarly. The user was attempting this without using OpenGL operations like glPerspective or glMatrixMode (since they're using Vulkan). They encountered issues where the rendered light position didn't match the AE position even after trying various model and projection matrix operations. The key challenge is ensuring proper coordinate space conversion between After Effects' internal representation and Vulkan's expected format.

*Tags: `vulkan`, `camera`, `matrix`, `shader`, `lighting`, `coordinate-space`*

---

## Is GLM compatible with Vulkan and can it be used for matrix operations in graphics pipelines?

Yes, GLM is compatible with Vulkan and is recommended for use with it. GLM provides matrix functions like glm::perspective and glm::lookAt that can be used for graphics transformations, though note that GLM's perspective function uses gluPerspective conventions which may differ from other implementations.

*Tags: `vulkan`, `gpu`, `compute-cache`, `reference`*

---

## Is there an open-source example of a Vulkan-based After Effects plugin?

Yes, Wunkolo has open-sourced Vulkanator, a Vulkan-based After Effects plugin project with sample code. This project represents an early push toward more open-source resources in the After Effects plugin development space. Repository: https://github.com/Wunkolo/Vulkanator

*Tags: `vulkan`, `open-source`, `reference`, `gpu`*

---

## Should you use Metal directly or MoltenVK for cross-platform GPU plugin development on macOS?

Using Vulkan with MoltenVK (a Vulkan layer over Metal) reduces code duplication between Windows and macOS compared to maintaining separate Metal and DirectX implementations. However, there are some macOS-specific limitations to account for. Alternatively, keeping OpenGL on Windows and Metal on macOS is viable, but carries the risk that Apple may eventually remove OpenGL support entirely, as they have deprecated it in favor of Metal.

*Tags: `gpu`, `metal`, `vulkan`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## What are the limitations of MoltenVK for After Effects plugin development?

MoltenVK before version 1.3 had several limitations including unsupported texture swizzle and uniforms limitations. However, these issues have been resolved in MoltenVK 1.3 and later versions.

*Tags: `vulkan`, `metal`, `moltenvk`, `macos`, `gpu`*

---

## How can VK_EXT_external_memory_host extension improve GPU data transfer performance?

The VK_EXT_external_memory_host extension (available on Windows) avoids the need to allocate a staging buffer by allowing a CPU-side void* to be turned directly into a VkDeviceMemory object. This enables copying data directly from an EffectWorld onto the GPU and allows the GPU to write back into it without redundant upload/download copies, resulting in significant gains in both memory and speed, especially for large 4K frames.

*Tags: `vulkan`, `gpu`, `memory`, `windows`, `optimization`*

---

## What is an example of an After Effects plugin using Vulkan?

ScaleUp is an AE script/plugin that uses Vulkan and MoltenVK for rendering. It can be found at https://aescripts.com/scaleup/

*Tags: `vulkan`, `moltenvk`, `open-source`, `reference`, `gpu`*

---

## What should I verify when using external libraries like Vulkan in an After Effects plugin?

Ensure that external libraries are properly installed on the user's system (e.g., Vulkan drivers install libraries to System32), check for any transitive dependencies from licensing tools or other libraries, and verify that all linked DLLs match the architecture of your plugin build.

*Tags: `windows`, `vulkan`, `deployment`, `build`*

---

## How can you embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications?

When embedding MoltenVK in an AE plugin, the standard approach of placing it in the framework folder as a dynamic library with JSON in resources/vulkan doesn't work for plugin bundles. The scaleUp plugin demonstrates using xcframework to embed it as a static library inside the plugin binary, though this approach requires careful configuration as Vulkan instance creation may fail if not set up correctly. The key challenge is ensuring proper linking and resource resolution within the plugin bundle rather than relying on system-wide installation in /etc/vulkan.

*Tags: `vulkan`, `macos`, `deployment`, `plugin`, `reference`*

---

## How do you embed MoltenVK as a static library in an After Effects plugin bundle to avoid conflicts with system installations?

When embedding MoltenVK as a static library in an After Effects plugin (xcframework), you may encounter Vulkan instance creation issues. The errors typically relate to missing Vulkan layers (VK_LAYER_KHRONOS_validation) and extensions (VK_KHR_portability_enumeration). This often indicates the MoltenVK_icd is not being correctly located or loaded. Ensure the Vulkan loader is properly configured for static linking. Reference the MoltenVK documentation for macOS development: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos. The scaleUp plugin demonstrates a working approach with embedded MoltenVK and Vulkan as static libraries.

*Tags: `vulkan`, `macos`, `moltenvk`, `static-linking`, `plugin-deployment`, `apple-silicon`*

---

## What is the official MoltenVK documentation for developing Vulkan applications on macOS?

The KhronosGroup MoltenVK GitHub repository contains detailed documentation on developing Vulkan applications for macOS, iOS, and tvOS, with careful guidance on environment setup and integration. Available at: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos

*Tags: `vulkan`, `moltenvk`, `macos`, `reference`, `documentation`*

---

## How do you set up Vulkan development on macOS with MoltenVK compatibility?

To develop Vulkan plugins on macOS with MoltenVK: (1) Update to at least October release to get MoltenVK compatible with Vulkan 1.2. (2) For debug builds, create a target using ICD (Installable Client Driver) in environment variables and dylib linking, then add the "VK_KHR_portability_subset" instance extension (debug-only until MoltenVK uses it natively) along with debugging layers. (3) For release builds, link only to frameworks as static libraries to comply with Vulkan 1.2 limitations.

*Tags: `vulkan`, `macos`, `cross-platform`, `debugging`, `build`*

---

## Is there an example of integrating Vulkan with After Effects on Apple Silicon?

Wunk is updating the Vulkanator sample project to demonstrate VulkanAPI (via MoltenVK) integration with After Effects on Apple Silicon, showing how to leverage Vulkan for GPU-accelerated rendering on macOS ARM64 systems.

*Tags: `vulkan`, `gpu`, `apple-silicon`, `metal`, `open-source`*

---

## Why is device-to-device buffer transfer between Vulkan and CUDA no faster than host-to-device transfer?

When importing a CUDA handle into Vulkan for external memory, the transfer may implicitly bounce through host memory if Vulkan cannot reliably determine how to handle the memory's caching. Additionally, the AE_cuda buffer may be allocated on the host rather than the device, which would explain why device-to-device transfer speeds match host-to-device speeds. Check whether the buffer is actually allocated on the device or if it's a host-side allocation.

*Tags: `vulkan`, `cuda`, `gpu`, `memory`, `cross-platform`*

---

## Is there an example After Effects plugin that uses Vulkan for GPU acceleration?

The Vulkanator project by Wunkolo demonstrates Vulkan integration with After Effects plugins for GPU-accelerated rendering. Repository: https://github.com/Wunkolo/Vulkanator

*Tags: `vulkan`, `gpu`, `open-source`, `reference`*

---

## What are the challenges with using third-party GPU debuggers for After Effects plugins?

GPU debuggers like gDEBugger, RenderDoc, and others frequently crash or fail when tracing After Effects. The core issue is that these debuggers are designed for standalone games with permanent instances, whereas plugin architectures within host applications like After Effects or Premiere Pro create a more complex environment. Additionally, debuggers may capture the host application's graphics API (DirectX) instead of the plugin's rendering API (OpenGL, Vulkan), making them ineffective for plugin-specific debugging.

*Tags: `gpu`, `debugging`, `opengl`, `vulkan`, `plugin-architecture`*

---

## What are alternative GPU APIs for communicating with After Effects to access GPU data?

Vulkan, CUDA, and Metal are viable options for GPU communication with After Effects. Developers have successfully implemented CUDA and Metal to get GPU data from AE in GPU mode. Vulkan is recommended as a cross-platform alternative to DirectX, though Adobe has not officially supported Vulkan exposure to plugins at this time.

*Tags: `gpu`, `vulkan`, `cuda`, `metal`, `cross-platform`*

---

## Should you use After Effects' native GPU API or create a custom GPU context for complex effects that need full source access?

According to Gabgren and Tim Constantinov's experience, After Effects' native GPU world is less developed with not all features available. For effects requiring full control over source data and GPU acceleration, it's better to create your own GPU context using OpenGL, Vulkan, or WebGPU rather than relying on AE's built-in GPU pipeline. This approach allows you to keep CPU-side operations intact and selectively send only necessary data to your custom GPU context. Tim Constantinov's team experimented with native AE GPU and OpenGL before settling on WebGPU, and Wunk has had positive results with Vulkan.

*Tags: `gpu`, `opengl`, `vulkan`, `webgpu`, `aegp`, `cross-platform`*

---

## What GPU API should I use for After Effects plugins and how do they compare?

Vulkan is highly recommended for maximum control over performance and memory management. It's explicit and detailed, allowing fine-tuning of memory transfers and format conversions. On macOS, MoltenVK (originally by Valve) automatically ports Vulkan features to Metal, with Khronos developing KosmicKrisp as its successor. WebGPU is easier to use and cross-platform but may not provide access to platform-specific optimizations. OpenGL is mature but deprecated—it's now typically implemented via Metal, DirectX, or Vulkan under the hood on all platforms. Vulkan also supports interoperability with CUDA, OpenCL, DirectX, Metal, and OpenGL, allowing you to import textures from After Effects' native GPU features into Vulkan.

*Tags: `vulkan`, `gpu`, `metal`, `opengl`, `cross-platform`, `performance`*

---

## What approach works well for implementing GPU acceleration in After Effects plugins?

A practical approach is to present your plugin as a CPU plugin to After Effects while implementing GPU rendering (Vulkan, OpenGL, or WebGPU) under the hood. This avoids compatibility issues while still leveraging GPU acceleration. Ensure that critical operations are actually executed on the GPU—falling back to CPU paths defeats the purpose.

*Tags: `gpu`, `vulkan`, `webgpu`, `optimization`, `rendering`*

---

## What are the tradeoffs between different GPU frameworks for After Effects plugins?

According to developers in the community, the progression has been: AE GPU → OpenGL → WebGPU. WebGPU is preferred because the code is closer to GLSL, it supports compute shaders, and it handles OS compatibility automatically (Metal for Mac, Vulkan for PC under the hood). However, WebGPU implementations have performance considerations: WGPU library is nice for compatibility but not great for performance, while Google's Dawn is much better performance-wise but difficult to build. A notable limitation of WebGPU is that it adds bounds checking in shaders without a way to disable it unless you load SPIRV shaders directly.

*Tags: `gpu`, `webgpu`, `vulkan`, `metal`, `opengl`, `cross-platform`*

---

## Is there an open-source skeleton project for Vulkan SDK integration with After Effects plugins?

Eric CPFX shared a Skeleton project designed for using the Vulkan SDK with After Effects plugin development. This project serves as a reference implementation for developers looking to integrate Vulkan into their AE plugins.

*Tags: `vulkan`, `skeleton`, `open-source`, `reference`, `build`*

---

## How do you efficiently copy pixel data row-by-row between Premiere and Vulkan/OpenGL APIs?

In Premiere, use memcpy with row-based copying by calculating pointers for each row: `void* PrPtr = (char*)PRData + y*PrRowByte; void* ApiPtr = (char*)ApiData + y*APIRowByte; memcpy(ApiPtr, PrPtr, APIRowByte);` where APIRowByte is calculated as `width * 4 * sizeOfPixel` (1 for 8-bit, 4 for 32-bit). PrRowByte comes from Premiere and may be negative. This approach works for reading from Premiere, but copying back to Premiere can cause crashes. For Vulkan specifically, use `vkGetImageSubresourceLayout()` to get the actual `VkSubresourceLayout` of source/destination images to ensure correct copying, as the layout may not always be packed. In After Effects, row-byte already includes 16-byte SIMD padding, so verify buffer alignment and that API pointers are correctly mapped to avoid access violations.

```cpp
void* PrPtr = (char*)PRData + y*PrRowByte;
void* ApiPtr = (char*)ApiData + y*APIRowByte;
memcpy(ApiPtr, PrPtr, APIRowByte);
```

*Tags: `vulkan`, `opengl`, `premiere`, `memory`, `gpu`*

---
