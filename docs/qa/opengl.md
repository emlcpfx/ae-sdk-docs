# Q&A: opengl

**40 entries** tagged with `opengl`.

---

## How do you use RenderDoc to debug GPU plugins in After Effects?

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

*Contributors: [**fad**](../contributors/fad/) · Source: adobe-plugin-devs · 2024-02-29 · Tags: `gpu`, `renderdoc`, `debugging`, `opengl`, `process-injection`*

---

## How does GLator (deprecated OpenGL sample) have memory leaks?

The GLator sample wraps GL rendering in try/catch. The suites.IterateFloatSuite1()->iterate call can throw PF_Interrupt_CANCEL in the download texture function, causing bufferH to never be deallocated. This leaks an entire output buffer of 32bpc pixels per frame from CPU, plus GPU FBO resources. Using scope guards (C++ RAII cleanup on scope exit) prevents this pattern - they handle cleanup regardless of how the scope is exited (exception, return, etc.).

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**tlafo**](../contributors/tlafo/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2025-04-18 · Tags: `glator`, `opengl`, `memory-leak`, `exception-handling`, `scope-guard`*

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

## What are the main challenges with GPU plugins and MFR compatibility?

GPU plugins have significant stability issues with MFR compared to CPU plugins. The issues are hard to track down because crashes occur only after extended rendering periods. OpenGL-based plugins experience particular difficulties, especially on macOS where platform-specific issues arise.

*Tags: `mfr`, `gpu`, `opengl`, `debugging`*

---

## What is the recommended approach for supporting GPU rendering across Windows and macOS?

In the future, maintaining two completely separate code bases may be necessary: one using Metal for macOS and one using OpenGL or another framework for Windows, due to significant platform-specific GPU compatibility issues.

*Tags: `gpu`, `metal`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## What GPU APIs are being used for Windows and Mac support?

For cross-platform development, MoltenVK (a Vulkan layer over Metal) is used, which allows shared code between Windows and Mac. Alternatively, some developers keep OpenGL on Windows and Metal on Mac. OpenGL 4.x and 3.3 are still in use by major plugins like Element 3D and Helium, though there are concerns about Apple potentially removing OpenGL support entirely.

*Tags: `gpu`, `opengl`, `metal`, `vulkan`, `cross-platform`, `macos`, `windows`*

---

## How to fix OpenGL loader initialization failure on M1 Mac with ImGui?

The issue is likely related to M1 chip compatibility. ImGui can run over Metal instead of OpenGL, which would require conversion from OpenGL matrices but would be the proper approach for Apple Silicon. The error may be similar to previous issues with GLSL version differences between Mac and Windows, but specific to ARM64 architecture on M1.

*Tags: `opengl`, `metal`, `macos`, `apple-silicon`, `debugging`*

---

## How do you convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL matrix format?

After Effects uses row-major matrix layout while OpenGL uses column-major layout. You need to transpose the matrix during conversion. For a 4x4 matrix, map AE matrix elements (row-major: 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15) to OpenGL format (column-major: 0,4,8,12,1,5,9,13,2,6,10,14,3,7,11,15). This can be done by iterating through the matrix and reassigning: glMatrix[x][0] = matrix.mat[0][x], glMatrix[x][1] = matrix.mat[1][x], glMatrix[x][2] = matrix.mat[2][x], glMatrix[x][3] = matrix.mat[3][x].

```cpp
for (x = 0; x < 4; x++)
{
    glMatrix[x][0] = matrix.mat[0][x];
    glMatrix[x][1] = matrix.mat[1][x];
    glMatrix[x][2] = matrix.mat[2][x];
    glMatrix[x][3] = matrix.mat[3][x];
}
```

*Tags: `aegp`, `opengl`, `matrix-conversion`, `camera`*

---

## What tools should be used for matrix operations and inversion when working with camera matrices in After Effects plugins?

The GLM (OpenGL Mathematics) library is recommended for performing matrix operations and inversions. It provides robust mathematical functions for handling matrix transformations needed when converting and manipulating camera coordinate systems.

*Tags: `aegp`, `opengl`, `matrix-conversion`, `camera`, `build`*

---

## Were other assignment operators like += working properly in GLSL?

The conversation indicates that /= was the problematic operator in GLSL, with the implication that other assignment operators like += were functioning correctly, though this wasn't explicitly confirmed in the replies.

*Tags: `opengl`, `debugging`, `gpu`*

---

## Why do third-party GPU debuggers crash or fail when used with After Effects?

Third-party GPU debuggers are designed for games with permanent instances, making them incompatible with plugins running inside host applications like After Effects or Premiere. Even modern API debuggers struggle with this architecture. RenderDoc, for example, tends to capture the host application's DirectX instead of the plugin's OpenGL, and gDEBugger crashes after tracing.

*Tags: `gpu`, `debugging`, `opengl`, `vulkan`, `premiere`*

---

## Why doesn't an Artisan-like OpenGL renderer plugin exist for After Effects?

The question was posed rhetorically but not answered directly in the conversation. The discussion suggests it would be theoretically possible to create a GPU plugin with 10k layers as sprites for real-time rendering, but no explanation was provided for why such a universal renderer hasn't been made.

*Tags: `gpu`, `opengl`, `render-loop`*

---

## How difficult would it be to match After Effects' lighting calculations in a custom 3D plugin?

This is a challenging task because it requires matching AE's specific lighting parameters and their meanings, such as spotlight falloff softness and the balance between ambient and direct lighting. Many plugins have implemented this, though the effort is significant.

*Tags: `gpu`, `opengl`, `metal`, `smartfx`*

---

## Does the old GLator sample have memory leaks in its GL rendering and texture download functions?

Yes, the GLator sample has major memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, meaning bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBO's.

*Tags: `memory`, `gpu`, `opengl`, `debugging`*

---

## Why not create a custom GPU context in OpenGL or Vulkan instead of using After Effects' native GPU API?

Instead of relying on After Effects' limited native GPU rendering API, you can create your own GPU context using OpenGL, Vulkan, or WebGPU. This approach allows you to maintain your CPU logic while selectively sending only what's needed to the GPU, avoiding the limitations of AE's GPU world which is less developed and has fewer available suites. gabgren and Tim Constantinov successfully used this approach, eventually settling on WebGPU after trying native AE and OpenGL.

*Tags: `gpu`, `opengl`, `vulkan`, `render-loop`, `cross-platform`*

---

## What GPU APIs are recommended for After Effects plugins?

For GPU work in After Effects plugins, several options are viable: Vulkan is recommended for maximum control over performance and memory transfers, with MoltenVK providing Metal support on macOS and KosmicKrisp coming as a successor. WebGPU is easier to use and cross-platform but may not allow platform-specific optimizations. OpenGL is mature but deprecated and implemented via Metal/DirectX/Vulkan under the hood. A practical approach is to implement a CPU plugin that does GPU rendering underneath. Vulkan interoperability features allow importing textures from CUDA, OpenCL, DirectX, Metal, or OpenGL.

*Tags: `gpu`, `vulkan`, `metal`, `opengl`, `cuda`, `cross-platform`, `memory`*

---

## Do you use Vulkan/WebGPU for everything instead of the After Effects GPU API?

The approach varies by developer. One developer started with AE GPU, moved to OpenGL, then to WebGPU. WebGPU is preferred because the code is closer to GLSL, it supports compute shaders, and handles OS compatibility automatically (using Metal on Mac and Vulkan on PC). However, WebGPU has performance trade-offs - WGPU library adds bounds checking in shaders without a way to disable it (unless loading SPIRV shaders directly), while Google's Dawn is better performance-wise but difficult to build.

*Tags: `gpu`, `webgpu`, `vulkan`, `metal`, `opengl`, `cross-platform`*

---

## How do you properly initialize ImGui with OpenGL on macOS?

When initializing ImGui with OpenGL on macOS, window flags must be set before window creation, not after. This differs from Windows behavior and is a common mistake. The OpenGL loader initialization fails if flags are set in the wrong order during the setup process.

*Tags: `macos`, `opengl`, `ui`, `debugging`*

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

## What are the main challenges when updating GPU plugins for MFR compatibility?

GPU plugins face significant challenges with MFR that CPU plugins don't encounter. Issues include crashes that occur only after long rendering periods (making them hard to debug), GPU caching complications, and sequence data handling. OpenGL-based plugins particularly struggle with Mac-specific issues. James Whiffin reported that even using Adobe's GLator sample, Mac OpenGL compatibility remains problematic. The recommendation is to consider maintaining completely separate codebases: one for Metal on Mac and one for Windows, as OpenGL performance on Mac can be severely degraded compared to Windows.

*Tags: `mfr`, `gpu`, `opengl`, `metal`, `macos`, `windows`, `caching`, `sequence-data`*

---

## Should you use Metal directly or MoltenVK for cross-platform GPU plugin development on macOS?

Using Vulkan with MoltenVK (a Vulkan layer over Metal) reduces code duplication between Windows and macOS compared to maintaining separate Metal and DirectX implementations. However, there are some macOS-specific limitations to account for. Alternatively, keeping OpenGL on Windows and Metal on macOS is viable, but carries the risk that Apple may eventually remove OpenGL support entirely, as they have deprecated it in favor of Metal.

*Tags: `gpu`, `metal`, `vulkan`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## What GPU APIs are commonly used by commercial After Effects plugins?

Popular commercial AE plugins like Element 3D and Helium use OpenGL (Element 3D uses OpenGL 4.x and Helium uses OpenGL 3.3). Many plugin developers continue to rely on OpenGL support on macOS despite its deprecation, hoping that Apple will prepare alternatives before removing it entirely.

*Tags: `gpu`, `opengl`, `macos`, `reference`*

---

## What OpenGL versions do popular After Effects effects use, and which ones work well?

Based on developer feedback, popular AE effects use varying OpenGL versions. Element 3D and Trapcode use OpenGL 4.x, while other effects like Helium maintain OpenGL 3.3. The questioner notes they previously used OpenGL 3.3 before migrating to Vulkan. OpenGL 3.3 appears to be a stable choice used by multiple working effects.

*Tags: `opengl`, `gpu`, `reference`, `cross-platform`*

---

## Why does ImGui fail to initialize OpenGL on macOS with Apple Silicon (M1)?

OpenGL initialization failures on M1 Macs are often related to Apple Silicon compatibility issues. ImGui can run over Metal instead of OpenGL, though converting from OpenGL matrices to Metal requires additional work. The issue may be related to GLSL version differences between macOS and Windows, or Metal API requirements on Apple Silicon.

*Tags: `macos`, `apple-silicon`, `opengl`, `metal`, `debugging`*

---

## How do I convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL coordinate system?

After Effects uses a row-major matrix layout (0 1 2 3 / 4 5 6 7 / 8 9 10 11 / 12 13 14 15) while OpenGL uses column-major (0 4 8 12 / 1 5 9 13 / 2 6 10 14 / 3 7 11 15). To convert, transpose the matrix by iterating through rows and columns: for each x from 0 to 3, set glMatrix[x][0] = matrix.mat[0][x], glMatrix[x][1] = matrix.mat[1][x], glMatrix[x][2] = matrix.mat[2][x], glMatrix[x][3] = matrix.mat[3][x]. You may also need matrix operations like inversion, for which the GLM library is recommended.

```cpp
for (x = 0; x < 4; x++)
{
    glMatrix[x][0] = matrix.mat[0][x];
    glMatrix[x][1] = matrix.mat[1][x];
    glMatrix[x][2] = matrix.mat[2][x];
    glMatrix[x][3] = matrix.mat[3][x];
}
```

*Tags: `aegp`, `opengl`, `matrix`, `coordinate-conversion`, `camera`*

---

## What resources explain how to convert After Effects camera matrices to OpenGL format?

Two archived Adobe Community threads provide detailed guidance on matrix coordinate system conversion for AE camera data. The first thread at https://community.adobe.com/t5/after-effects-discussions/glator-for-dummies-and-from-dummy/m-p/6930311 discusses camera matrix conversion generally. The second thread (archived at https://web.archive.org/web/20111223234233/http://forums.adobe.com/thread/570135) provides specific code examples for transposing AE matrices to OpenGL format, explaining the row-major vs column-major difference and matrix operations needed.

*Tags: `aegp`, `opengl`, `camera`, `reference`, `matrix`*

---

## What GLSL assignment operators are supported across different GPU platforms in After Effects plugins?

James Whiffin encountered an issue where a specific GLSL operator caused compilation failures. The error was identified through shader compilation feedback. The conversation indicates that while some assignment operators like += work correctly, the /= operator caused issues on at least one platform (Windows or Mac, GPU reference unknown). This suggests developers need to be cautious about which GLSL assignment operators are universally supported across different GPU architectures when developing AE plugins.

*Tags: `gpu`, `opengl`, `glsl`, `debugging`, `cross-platform`, `macos`, `windows`*

---

## What are the challenges with using third-party GPU debuggers for After Effects plugins?

GPU debuggers like gDEBugger, RenderDoc, and others frequently crash or fail when tracing After Effects. The core issue is that these debuggers are designed for standalone games with permanent instances, whereas plugin architectures within host applications like After Effects or Premiere Pro create a more complex environment. Additionally, debuggers may capture the host application's graphics API (DirectX) instead of the plugin's rendering API (OpenGL, Vulkan), making them ineffective for plugin-specific debugging.

*Tags: `gpu`, `debugging`, `opengl`, `vulkan`, `plugin-architecture`*

---

## Why hasn't someone created an OpenGL-based realtime GPU renderer plugin for After Effects that can handle thousands of layers as sprites?

This is a theoretical discussion about a potential feature gap in After Effects plugin ecosystem. The concept would involve creating a custom 3D renderer plugin that could leverage GPU acceleration to render 10,000+ layers in realtime by treating them as sprites, similar to what exists in other software like Artisan. No concrete answer or existing solution was provided in the conversation.

*Tags: `gpu`, `opengl`, `render-loop`, `reference`*

---

## Does the old GLator sample have memory leaks in its GL rendering code?

Yes, the GLator sample has significant memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, which means bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBOs.

*Tags: `opengl`, `memory`, `gpu`, `deprecated`, `reference`*

---

## Should you use After Effects' native GPU API or create a custom GPU context for complex effects that need full source access?

According to Gabgren and Tim Constantinov's experience, After Effects' native GPU world is less developed with not all features available. For effects requiring full control over source data and GPU acceleration, it's better to create your own GPU context using OpenGL, Vulkan, or WebGPU rather than relying on AE's built-in GPU pipeline. This approach allows you to keep CPU-side operations intact and selectively send only necessary data to your custom GPU context. Tim Constantinov's team experimented with native AE GPU and OpenGL before settling on WebGPU, and Wunk has had positive results with Vulkan.

*Tags: `gpu`, `opengl`, `vulkan`, `webgpu`, `aegp`, `cross-platform`*

---

## What GPU API should I use for After Effects plugins and how do they compare?

Vulkan is highly recommended for maximum control over performance and memory management. It's explicit and detailed, allowing fine-tuning of memory transfers and format conversions. On macOS, MoltenVK (originally by Valve) automatically ports Vulkan features to Metal, with Khronos developing KosmicKrisp as its successor. WebGPU is easier to use and cross-platform but may not provide access to platform-specific optimizations. OpenGL is mature but deprecated—it's now typically implemented via Metal, DirectX, or Vulkan under the hood on all platforms. Vulkan also supports interoperability with CUDA, OpenCL, DirectX, Metal, and OpenGL, allowing you to import textures from After Effects' native GPU features into Vulkan.

*Tags: `vulkan`, `gpu`, `metal`, `opengl`, `cross-platform`, `performance`*

---

## What are the tradeoffs between different GPU frameworks for After Effects plugins?

According to developers in the community, the progression has been: AE GPU → OpenGL → WebGPU. WebGPU is preferred because the code is closer to GLSL, it supports compute shaders, and it handles OS compatibility automatically (Metal for Mac, Vulkan for PC under the hood). However, WebGPU implementations have performance considerations: WGPU library is nice for compatibility but not great for performance, while Google's Dawn is much better performance-wise but difficult to build. A notable limitation of WebGPU is that it adds bounds checking in shaders without a way to disable it unless you load SPIRV shaders directly.

*Tags: `gpu`, `webgpu`, `vulkan`, `metal`, `opengl`, `cross-platform`*

---

## What GPU rendering approach did Red Giant Universe migrate to from CPU-only plugins?

Red Giant Universe transitioned from CPU-only plugins with OpenGL in the background to a custom GPU renderer backend that supports Metal, DirectX, and OpenGL. They now provide GPU buffer interop for every GPU buffer type available, allowing plugins to leverage native GPU rendering across different platforms.

*Tags: `gpu`, `metal`, `opengl`, `cross-platform`, `render-loop`*

---

## Should developers use the After Effects GPU Suite for plugin development?

No, the AE GPU Suite should be avoided for plugin development. Instead, developers should implement custom GPU renderer backends that support Metal, DirectX, and OpenGL with proper GPU buffer interoperability, as demonstrated by modern plugin architectures like Red Giant Universe's approach.

*Tags: `gpu`, `debugging`, `metal`, `opengl`, `best-practices`*

---

## Why does ImGui fail to initialize OpenGL loader on macOS?

The issue was related to window flag initialization order. On macOS, window flags need to be set before window creation, not after. This differs from Windows behavior where setting flags after creation may work. Ensure all window configuration flags are applied before creating the window.

*Tags: `macos`, `opengl`, `ui`, `debugging`*

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

## How do you prevent video memory leaks when using OpenGL in After Effects plugins, especially during UI parameter changes?

The issue is typically caused by exceptions thrown within OpenGL calls during an interrupt, which causes the code to skip glDeleteTextures calls and jump to catch blocks. To fix this, ensure that glDeleteTextures is called before checking for interrupts using PF_ABORT(), and put breakpoints in catch blocks to verify if exceptions are being thrown during user interaction. The proper order should be: perform GPU operations, delete textures, unbind framebuffers and textures, then check for abort conditions and release suites.

```cpp
glBindFramebuffer(GL_FRAMEBUFFER, 0);
glBindTexture(GL_TEXTURE_2D, 0);
glDeleteTextures(1, &inputFrameTexture);
glDeleteTextures(1, &inputFrameTexture2);
// Then check for abort after cleanup
ERR(PF_ABORT(in_data));
```

*Tags: `gpu`, `opengl`, `memory`, `debugging`, `plugin-development`*

---

## Is there a sample demonstrating OpenGL usage in After Effects plugins?

Yes, Adobe provides the GLator sample as a reference for using OpenGL in After Effects plugins. This sample can help developers understand how to integrate OpenGL drawing capabilities into their plugins.

*Tags: `opengl`, `gpu`, `reference`, `sample`*

---
