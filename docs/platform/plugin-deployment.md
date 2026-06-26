# Plugin Deployment

> 1 Q&A · source: AE plugin dev community Discord

### How do you embed MoltenVK as a static library in an After Effects plugin bundle to avoid conflicts with system installations?

When embedding MoltenVK as a static library in an After Effects plugin (xcframework), you may encounter Vulkan instance creation issues. The errors typically relate to missing Vulkan layers (VK_LAYER_KHRONOS_validation) and extensions (VK_KHR_portability_enumeration). This often indicates the MoltenVK_icd is not being correctly located or loaded. Ensure the Vulkan loader is properly configured for static linking. Reference the MoltenVK documentation for macOS development: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos. The scaleUp plugin demonstrates a working approach with embedded MoltenVK and Vulkan as static libraries.

*Tags: `apple-silicon`, `macos`, `moltenvk`, `plugin-deployment`, `static-linking`, `vulkan`*

---
