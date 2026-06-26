# Cmake

> 7 Q&As · source: AE plugin dev community Discord

### Are there CMake-based build systems for AE plugins?

Yes, there are several examples on GitHub: Vulkanator (https://github.com/Wunkolo/Vulkanator) uses CMake, and after_effects_cmake (https://github.com/mobile-bungalow/after_effects_cmake) is based on Vulkanator's setup. There's also a Rust-based project (https://github.com/virtualritz/after-effects) with its own PiPL compiler written in Rust. On Mac, you can use the Rez tool for .rsrc files and potentially use Ninja for faster incremental compilation.

*Tags: `build-system`, `cmake`, `cross-platform`, `ninja`, `xcode`*

---

### Is there a skeleton/starter project for building an AE plugin with Vulkan GPU rendering?

Yes — AE_Skeleton_Vulkan by Eric CPFX (emlcpfx) is a complete CMake-based skeleton that incorporates the Vulkan SDK into an AE plugin. It also includes build scripts and a sample script for signing, notarizing, and stapling on Mac. Repository: https://github.com/emlcpfx/AE_Skeleton_Vulkan

*Tags: `build`, `cmake`, `code-signing`, `gpu`, `macos`, `skeleton`, `vulkan`*

---

### Has anyone successfully compiled an After Effects plugin using cmake/ninja?

Yes, there are working examples available. The mobile-bungalow/after_effects_cmake repository provides a cmake setup based on vulkanator's work. Additionally, the virtualritz/after-effects project offers a cross-platform build system with its own PiPL compiler written in Rust as an alternative to using pipltool.exe.

*Tags: `build`, `cmake`, `cross-platform`, `pipl`*

---

### How can you achieve incremental compilation for After Effects plugins to improve build times?

Using the Ninja generator with cmake can provide faster incremental compilation. The xcodebuild CLI tool does not appear to support incremental compilation effectively, resulting in full rebuilds taking around 50 seconds. Switching from Xcode project generation to Ninja as the cmake generator should enable proper incremental compilation.

*Tags: `build`, `cmake`, `macos`, `performance`*

---

### Has anyone ported the After Effects plugin build system to use CMake instead of Visual Studio and Xcode?

The question was asked but no one in the conversation provided a definitive answer or shared an existing CMake-based build system for AE plugins. This remains an open challenge in the community.

*Tags: `build`, `cmake`, `cross-platform`, `windows`, `xcode`*

---

### Is there an open-source After Effects plugin example that uses CMake for build system setup?

The aftereffects_spatial_media_plugins repository by Gorialis uses CMake for build configuration, allowing developers to set up the build system themselves rather than relying on platform-specific templates. This approach is appealing for creating new plugins with a more understandable build process. Repository: https://github.com/Gorialis/aftereffects_spatial_media_plugins/

*Tags: `build`, `cmake`, `open-source`, `reference`*

---

### Is there a CMake-based build system available for After Effects plugins?

Yes, there is a CMake/Ninja setup available at https://github.com/mobile-bungalow/after_effects_cmake which is based on vulkanator's cmake setup. This uses pipltool.exe for PiPL compilation.

*Tags: `build`, `cmake`, `open-source`, `pipl`, `reference`*

---
