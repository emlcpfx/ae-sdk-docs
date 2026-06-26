# Moltenvk

> 5 Q&As · source: AE plugin dev community Discord

### What are the limitations of MoltenVK before version 1.3?

MoltenVK before VK 1.3 had some limitations including unsupported texture swizzle and uniforms limitations, but these issues are now resolved in version 1.3 and later.

*Tags: `macos`, `metal`, `moltenvk`, `vulkan`*

---

### What are the limitations of MoltenVK for After Effects plugin development?

MoltenVK before version 1.3 had several limitations including unsupported texture swizzle and uniforms limitations. However, these issues have been resolved in MoltenVK 1.3 and later versions.

*Tags: `gpu`, `macos`, `metal`, `moltenvk`, `vulkan`*

---

### What is an example of an After Effects plugin using Vulkan?

ScaleUp is an AE script/plugin that uses Vulkan and MoltenVK for rendering. It can be found at https://aescripts.com/scaleup/

*Tags: `gpu`, `moltenvk`, `open-source`, `reference`, `vulkan`*

---

### How do you embed MoltenVK as a static library in an After Effects plugin bundle to avoid conflicts with system installations?

When embedding MoltenVK as a static library in an After Effects plugin (xcframework), you may encounter Vulkan instance creation issues. The errors typically relate to missing Vulkan layers (VK_LAYER_KHRONOS_validation) and extensions (VK_KHR_portability_enumeration). This often indicates the MoltenVK_icd is not being correctly located or loaded. Ensure the Vulkan loader is properly configured for static linking. Reference the MoltenVK documentation for macOS development: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos. The scaleUp plugin demonstrates a working approach with embedded MoltenVK and Vulkan as static libraries.

*Tags: `apple-silicon`, `macos`, `moltenvk`, `plugin-deployment`, `static-linking`, `vulkan`*

---

### What is the official MoltenVK documentation for developing Vulkan applications on macOS?

The KhronosGroup MoltenVK GitHub repository contains detailed documentation on developing Vulkan applications for macOS, iOS, and tvOS, with careful guidance on environment setup and integration. Available at: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos

*Tags: `documentation`, `macos`, `moltenvk`, `reference`, `vulkan`*

---
