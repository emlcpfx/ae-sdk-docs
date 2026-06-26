# Skeleton

> 7 Q&As · source: AE plugin dev community Discord

### Is there a skeleton/starter project for building an AE plugin with Vulkan GPU rendering?

Yes — AE_Skeleton_Vulkan by Eric CPFX (emlcpfx) is a complete CMake-based skeleton that incorporates the Vulkan SDK into an AE plugin. It also includes build scripts and a sample script for signing, notarizing, and stapling on Mac. Repository: https://github.com/emlcpfx/AE_Skeleton_Vulkan

*Tags: `build`, `cmake`, `code-signing`, `gpu`, `macos`, `skeleton`, `vulkan`*

---

### Is there an open-source skeleton project for Vulkan SDK integration with After Effects plugins?

Eric CPFX shared a Skeleton project designed for using the Vulkan SDK with After Effects plugin development. This project serves as a reference implementation for developers looking to integrate Vulkan into their AE plugins.

*Tags: `build`, `open-source`, `reference`, `skeleton`, `vulkan`*

---

### How should arbitrary data reallocation be handled when it depends on non-animatable parameters like 'rows' and 'cols' in a C++ effect plugin?

Arbitrary parameter handlers in After Effects are stateless functions that operate without access to a specific effect instance, effect reference, or sequence data. Instead of trying to fetch parameter values during HandleArbitrary() calls (where effect_ref is usually null and checkout/checkin rarely work), you should reallocate and store allocation-related metadata like 'rows' and 'cols' in your arbitrary data structure itself during UserChangedParam() or the render function. This way, HandleArbitrary() can process the data independently without querying system or instance state. The render function should fetch raw arbitrary data and other relevant pieces available during render time, then process them there rather than during arbitrary parameter handling calls.

*Tags: `arb-data`, `memory`, `params`, `skeleton`*

---

### What are good sample projects to learn how to pull and push pixel values in After Effects plugins?

The Adobe After Effects SDK includes two key sample projects: the Shifter sample demonstrates how to iterate through output samples and pull input values from arbitrary locations (useful for coordinate transformation effects), and the CCU sample project shows how to access buffer pixels directly in RAM for pushing values to specific pixel locations.

*Tags: `open-source`, `reference`, `sdk`, `skeleton`*

---

### How do you fix a Skeleton project that won't build with a web deployment error?

If the Skeleton project shows a conversion warning about "Web deployment to the local IIS server is no longer supported" and won't build, this error disables your build settings. The simplest solution is to duplicate a working project from the SDK and replace only the .cpp and .h files with the Skeleton files, rather than trying to manually fix the corrupted build settings.

*Tags: `after_effects_sdk`, `build`, `sdk`, `skeleton`, `visual_studio`*

---

### What is the correct Mac OS X SDK version to use when porting an After Effects plugin from Windows to Mac?

When porting a plugin to Mac using Xcode, if you encounter an error about a missing SDK (such as 'There is no SDK with the name or path /Developer/SDKs/MacOSX10.4u.sdk'), you should update the project's Base SDK setting to a newer version like Mac OS X 10.6. The available choices typically include versions like 10.5, 10.6, and "Current Mac OS", and 10.6 was confirmed as a working choice for successful compilation.

*Tags: `build`, `cross-platform`, `macos`, `skeleton`, `xcode`*

---

### How do I fix the 'project file is not writeable' error when modifying After Effects SDK skeleton projects in Xcode?

The issue is typically that the project files have write-protection enabled. Check if files are locked by selecting them in Finder and opening Get Info to see if the Locked checkbox is checked. If so, uncheck it for each file. This write-locking is common when files are transferred from Windows due to read-only media assumptions. If files aren't locked, verify project settings don't have SCM (Source Control Management) enabled, as this can also cause writeability issues with .pbxproj files.

*Tags: `build`, `debugging`, `macos`, `skeleton`, `xcode`*

---
