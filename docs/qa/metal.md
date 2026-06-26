# Q&A: metal

**23 entries** tagged with `metal`.

---

## How do you pass custom data structs to GPU kernels in AE plugins?

Define the struct in both the kernel code and your C++ header, then use void pointers to transfer the data from AE to the kernel. Pointers work with CUDA. For Metal, the approach may differ. For OpenCL it works with a few caveats.

*Contributors: [**tlafo**](../contributors/tlafo/), [**James Whiffin**](../contributors/james-whiffin/) · Source: adobe-plugin-devs · 2023-07-29 · Tags: `gpu`, `cuda`, `metal`, `opencl`, `kernel`, `custom-struct`*

---

## What is the recommended approach for supporting GPU rendering across Windows and macOS?

In the future, maintaining two completely separate code bases may be necessary: one using Metal for macOS and one using OpenGL or another framework for Windows, due to significant platform-specific GPU compatibility issues.

*Tags: `gpu`, `metal`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## Has anyone used metal cpp bindings instead of Objective-C for After Effects plugin development?

No direct answer was provided in the conversation. One participant acknowledged the question but did not provide experience or recommendations.

*Tags: `metal`, `macos`, `build`*

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

## How to fix OpenGL loader initialization failure on M1 Mac with ImGui?

The issue is likely related to M1 chip compatibility. ImGui can run over Metal instead of OpenGL, which would require conversion from OpenGL matrices but would be the proper approach for Apple Silicon. The error may be similar to previous issues with GLSL version differences between Mac and Windows, but specific to ARM64 architecture on M1.

*Tags: `opengl`, `metal`, `macos`, `apple-silicon`, `debugging`*

---

## How difficult would it be to match After Effects' lighting calculations in a custom 3D plugin?

This is a challenging task because it requires matching AE's specific lighting parameters and their meanings, such as spotlight falloff softness and the balance between ambient and direct lighting. Many plugins have implemented this, though the effort is significant.

*Tags: `gpu`, `opengl`, `metal`, `smartfx`*

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

## What are the main challenges when updating GPU plugins for MFR compatibility?

GPU plugins face significant challenges with MFR that CPU plugins don't encounter. Issues include crashes that occur only after long rendering periods (making them hard to debug), GPU caching complications, and sequence data handling. OpenGL-based plugins particularly struggle with Mac-specific issues. James Whiffin reported that even using Adobe's GLator sample, Mac OpenGL compatibility remains problematic. The recommendation is to consider maintaining completely separate codebases: one for Metal on Mac and one for Windows, as OpenGL performance on Mac can be severely degraded compared to Windows.

*Tags: `mfr`, `gpu`, `opengl`, `metal`, `macos`, `windows`, `caching`, `sequence-data`*

---

## Can you use Metal C++ bindings instead of Objective-C for After Effects plugin development on macOS?

James Whiffin asked about using Metal C++ bindings as an alternative to Objective-C for AE plugin development, noting that C++ would be more familiar for developers without Objective-C experience. This suggests that Metal C++ bindings are a viable option for macOS plugin development, offering a more accessible path for C++-focused developers.

*Tags: `metal`, `macos`, `cross-platform`, `gpu`*

---

## Should you use Metal directly or MoltenVK for cross-platform GPU plugin development on macOS?

Using Vulkan with MoltenVK (a Vulkan layer over Metal) reduces code duplication between Windows and macOS compared to maintaining separate Metal and DirectX implementations. However, there are some macOS-specific limitations to account for. Alternatively, keeping OpenGL on Windows and Metal on macOS is viable, but carries the risk that Apple may eventually remove OpenGL support entirely, as they have deprecated it in favor of Metal.

*Tags: `gpu`, `metal`, `vulkan`, `opengl`, `cross-platform`, `macos`, `windows`*

---

## What are the limitations of MoltenVK for After Effects plugin development?

MoltenVK before version 1.3 had several limitations including unsupported texture swizzle and uniforms limitations. However, these issues have been resolved in MoltenVK 1.3 and later versions.

*Tags: `vulkan`, `metal`, `moltenvk`, `macos`, `gpu`*

---

## Why does ImGui fail to initialize OpenGL on macOS with Apple Silicon (M1)?

OpenGL initialization failures on M1 Macs are often related to Apple Silicon compatibility issues. ImGui can run over Metal instead of OpenGL, though converting from OpenGL matrices to Metal requires additional work. The issue may be related to GLSL version differences between macOS and Windows, or Metal API requirements on Apple Silicon.

*Tags: `macos`, `apple-silicon`, `opengl`, `metal`, `debugging`*

---

## Is there an example of integrating Vulkan with After Effects on Apple Silicon?

Wunk is updating the Vulkanator sample project to demonstrate VulkanAPI (via MoltenVK) integration with After Effects on Apple Silicon, showing how to leverage Vulkan for GPU-accelerated rendering on macOS ARM64 systems.

*Tags: `vulkan`, `gpu`, `apple-silicon`, `metal`, `open-source`*

---

## What should you assume about the data pointer in pF_effectworld when using GPU mode in After Effects plugins?

According to the SDK documentation, when using GPU mode (CUDA, Metal, or OpenCL), you must assume that the data pointer is NULL in the pF_effectworld structure. This means the pixel data is not available in system memory and you need to work with GPU-resident data instead.

*Tags: `gpu`, `cuda`, `metal`, `opencl`, `mfr`, `reference`*

---

## What are alternative GPU APIs for communicating with After Effects to access GPU data?

Vulkan, CUDA, and Metal are viable options for GPU communication with After Effects. Developers have successfully implemented CUDA and Metal to get GPU data from AE in GPU mode. Vulkan is recommended as a cross-platform alternative to DirectX, though Adobe has not officially supported Vulkan exposure to plugins at this time.

*Tags: `gpu`, `vulkan`, `cuda`, `metal`, `cross-platform`*

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
