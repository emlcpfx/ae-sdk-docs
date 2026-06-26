# Q&A: moltenvk

**5 entries** tagged with `moltenvk`.

---

## What are the limitations of MoltenVK before version 1.3?

MoltenVK before VK 1.3 had some limitations including unsupported texture swizzle and uniforms limitations, but these issues are now resolved in version 1.3 and later.

*Tags: `vulkan`, `metal`, `moltenvk`, `macos`*

---

## What are the limitations of MoltenVK for After Effects plugin development?

MoltenVK before version 1.3 had several limitations including unsupported texture swizzle and uniforms limitations. However, these issues have been resolved in MoltenVK 1.3 and later versions.

*Tags: `vulkan`, `metal`, `moltenvk`, `macos`, `gpu`*

---

## What is an example of an After Effects plugin using Vulkan?

ScaleUp is an AE script/plugin that uses Vulkan and MoltenVK for rendering. It can be found at https://aescripts.com/scaleup/

*Tags: `vulkan`, `moltenvk`, `open-source`, `reference`, `gpu`*

---

## How do you embed MoltenVK as a static library in an After Effects plugin bundle to avoid conflicts with system installations?

When embedding MoltenVK as a static library in an After Effects plugin (xcframework), you may encounter Vulkan instance creation issues. The errors typically relate to missing Vulkan layers (VK_LAYER_KHRONOS_validation) and extensions (VK_KHR_portability_enumeration). This often indicates the MoltenVK_icd is not being correctly located or loaded. Ensure the Vulkan loader is properly configured for static linking. Reference the MoltenVK documentation for macOS development: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos. The scaleUp plugin demonstrates a working approach with embedded MoltenVK and Vulkan as static libraries.

*Tags: `vulkan`, `macos`, `moltenvk`, `static-linking`, `plugin-deployment`, `apple-silicon`*

---

## What is the official MoltenVK documentation for developing Vulkan applications on macOS?

The KhronosGroup MoltenVK GitHub repository contains detailed documentation on developing Vulkan applications for macOS, iOS, and tvOS, with careful guidance on environment setup and integration. Available at: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos

*Tags: `vulkan`, `moltenvk`, `macos`, `reference`, `documentation`*

---
