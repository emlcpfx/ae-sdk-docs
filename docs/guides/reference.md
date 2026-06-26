# Reference

> 240 Q&As · source: AE plugin dev community Discord

### Where should After Effects SDK developers share and discover code snippets and solutions?

Stack Overflow with the 'After Effects SDK' tag is recommended as a good system for searching and hosting code snippets. GitHub can also be used for sharing code examples, though the AESDK community is niche. Creating dedicated channels or communities helps organize and grow the knowledge base.

*Tags: `deployment`, `open-source`, `reference`, `scripting`, `tool`*

---

### Where can I find discussions about transform_world and motion blur issues in After Effects?

Rich shared a question posted on the Adobe After Effects community forum discussing transform_world and motion blur. The forum thread can be found at https://community.adobe.com/t5/after-effects-discussions/transform-world-amp-motion-blur/td-p/12918545

*Tags: `debugging`, `motion-blur`, `reference`, `transform`*

---

### Where can I find an example After Effects plugin implementation?

dvb metareal shared their After Effects plugin at https://omino.com/pixelblog, currently at version 0.9. This serves as a working reference implementation for plugin development.

*Tags: `open-source`, `reference`, `tool`*

---

### What resources exist to understand sequence data in After Effects plugin development?

Tobias Fleischer (reduxFX) has created comprehensive documentation about sequence data that has helped developers understand the concept clearly. This resource is considered essential reference material for working with sequence data in After Effects plugins.

*Tags: `aegp`, `documentation`, `reference`, `sequence-data`*

---

### What alignment guarantees does PF_LayerDef::rowbytes have in After Effects?

rowbytes will always be a multiple of sizeof(PF_Pixel8), which is 4 bytes, but 16-byte alignment is not guaranteed. The minimum guarantee is that rowbytes must be a multiple of the alignment requirement of the pixel type being used. Since After Effects effects don't have pixel types smaller than ARGB8, rowbytes will always be at least 4-byte aligned in practice. Misaligned pointers would result in undefined behavior according to the C specification.

*Tags: `aegp`, `memory`, `params`, `reference`*

---

### What does the C specification say about pointer alignment and undefined behavior?

According to the C specification (section 6.3.2.3, paragraph 7 of the C11 standard), a pointer to an object type may be converted to a pointer to a different object type, but if the resulting pointer is not correctly aligned for the referenced type, the behavior is undefined. Apple documents this issue in their Xcode documentation on misaligned pointers: https://developer.apple.com/documentation/xcode/misaligned-pointer

*Tags: `cross-platform`, `debugging`, `memory`, `reference`*

---

### How can you install and activate Adobe After Effects CS6 on Windows 11?

CS6 may have compatibility issues with Windows 11. Try running the installer in compatibility mode, or consider using Windows 10 instead if Windows 11 proves problematic. Adobe's activation servers for CS6 may no longer be functional, which can prevent license key activation on new machines.

*Tags: `cross-platform`, `deployment`, `reference`, `windows`*

---

### How should popup menu items be formatted in After Effects plugin parameters?

Popup menu items should be separated by pipe characters (|) with the separator placed at the end of each line, including the last item. For example: "choice1|" "choice2|" "Choice3". When using PF_ADD_POPUPX, ensure the parameter definition is properly initialized with AEFX_CLR_STRUCT(def) before adding the popup, and that there are no missing parameter definitions between the popup and any transition function that follows it.

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_POPUPX("Color", 3, 2,
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Tags: `params`, `reference`, `ui`*

---

### Is there a reference implementation for handling BGRA and YUVA colorspaces in Premiere plugins?

Yes, the SDK noise example in the After Effects SDK demonstrates proper handling of both BGRA and YUVA colorspaces for Premiere. This example shows how to use the special BGRA suite for 8-bit processing and serves as a reference for implementing colorspace support in plugins targeting both AE and Premiere.

*Tags: `bgra`, `open-source`, `premiere`, `reference`, `sdk-example`, `yuva`*

---

### Where can I find information about Premiere UXP Beta?

Alex Bizeau from Maxon published a blog post with information on Premiere UXP Beta. The blog post was shared as a resource for developers interested in learning more about the Premiere UXP Beta.

*Tags: `premiere`, `reference`, `resource`, `uxp`*

---

### What is the best documentation to get started with UXP for Premiere Pro?

According to the discussion, there is a comprehensive article that contains all the links for UXP documentation, forums, and other resources for Premiere Pro development. UXP in Premiere Pro is currently limited but will be expanded soon.

*Tags: `premiere`, `reference`, `ui`, `uxp`*

---

### What is the Transfusion plugin and how does it demonstrate sequence data caching?

Transfusion is an After Effects plugin by gabgren that demonstrates caching the EffectWorld in sequence data across frames. The plugin stores the result of each render call in sequence_data, then uses that cached result on the next frame for additive effects. This approach works for 8-bit rendering without MFR (Multi-Frame Rendering), though MFR compatibility may require adjustments to this workflow.

*Tags: `caching`, `open-source`, `reference`, `sequence-data`*

---

### What is the Compute Cache useful for in After Effects plugin development?

The Compute Cache is a useful learning resource for understanding how to work with sequence data and plugin state management in After Effects plugins.

*Tags: `compute-cache`, `reference`, `sequence-data`*

---

### What is the current behavior of the After Effects Memory Suite functions?

According to discussions on the aescripts Slack channel, the Memory Suite functions originally worked as intended in earlier versions of After Effects. However, in current versions, these functions essentially just call new and delete, suggesting they may no longer provide the specialized memory management they once did. This is important context when deciding how to handle memory allocation and deallocation in plugins.

*Tags: `aegp`, `memory`, `reference`*

---

### Is GLM compatible with Vulkan and can it be used for matrix operations in graphics pipelines?

Yes, GLM is compatible with Vulkan and is recommended for use with it. GLM provides matrix functions like glm::perspective and glm::lookAt that can be used for graphics transformations, though note that GLM's perspective function uses gluPerspective conventions which may differ from other implementations.

*Tags: `compute-cache`, `gpu`, `reference`, `vulkan`*

---

### Is there an open-source example of volumetric fractal noise implementation for After Effects?

Yes, there is a GitHub project by mes51 that implements volumetric fractal noise: https://github.com/mes51/VolumetricFractalNoise. This project serves as a reference implementation and workaround for volumetric fractal noise effects in After Effects plugins.

*Tags: `gpu`, `open-source`, `reference`, `rendering`*

---

### Why must X and Y positions be cast to FIX when using subpixel_sample_float for displacement sampling?

When using the subpixel_sample_float function from the Sampling Float Suite, coordinates must be converted to FIX format (fixed-point notation) as required by the AE API, even though you're working with float values. This is a requirement of the sampling function signature, not a rounding operation that degrades quality. The issue with displacement quality may relate to the sampling method chosen rather than the FIX conversion itself. Consulting displacement plugin implementations and tutorials can help identify if a different sampling approach would yield better results.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref, INT2FIX(newX), INT2FIX(newY), &giP->inSamp_pb, &samplePixel));
```

*Tags: `displacement`, `params`, `reference`, `sampling`*

---

### Is there a tutorial or reference for building displacement plugins in After Effects?

The Warbler plugin tutorial from MacTech Magazine (Vol. 15, Issue 9) provides guidance on building displacement effects in After Effects plugins. It covers the fundamentals of displacement plugin development and can help understand proper sampling techniques and implementation practices.

*Tags: `displacement`, `reference`, `tutorial`*

---

### Is there an open-source example of a Vulkan-based After Effects plugin?

Yes, Wunkolo has open-sourced Vulkanator, a Vulkan-based After Effects plugin project with sample code. This project represents an early push toward more open-source resources in the After Effects plugin development space. Repository: https://github.com/Wunkolo/Vulkanator

*Tags: `gpu`, `open-source`, `reference`, `vulkan`*

---

### Where can I find an example solution for color grid implementation in After Effects plugins?

The colorgrid example in the After Effects SDK contains the solution. This is a reference implementation that demonstrates how to work with color grids in plugin development.

*Tags: `open-source`, `reference`, `sdk`*

---

### What is a good resource for understanding the Jump Flooding Algorithm for distance field generation?

The blog post at https://itscai.us/blog/post/jfa/ provides an intuitive description of the Jump Flooding Algorithm and demonstrates how it applies directly to creating outlines and choker effects from alpha channels.

*Tags: `algorithm`, `distance-field`, `reference`, `tool`*

---

### What GPU APIs are commonly used by commercial After Effects plugins?

Popular commercial AE plugins like Element 3D and Helium use OpenGL (Element 3D uses OpenGL 4.x and Helium uses OpenGL 3.3). Many plugin developers continue to rely on OpenGL support on macOS despite its deprecation, hoping that Apple will prepare alternatives before removing it entirely.

*Tags: `gpu`, `macos`, `opengl`, `reference`*

---

### What is an example of an After Effects plugin using Vulkan?

ScaleUp is an AE script/plugin that uses Vulkan and MoltenVK for rendering. It can be found at https://aescripts.com/scaleup/

*Tags: `gpu`, `moltenvk`, `open-source`, `reference`, `vulkan`*

---

### How can you embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications?

When embedding MoltenVK in an AE plugin, the standard approach of placing it in the framework folder as a dynamic library with JSON in resources/vulkan doesn't work for plugin bundles. The scaleUp plugin demonstrates using xcframework to embed it as a static library inside the plugin binary, though this approach requires careful configuration as Vulkan instance creation may fail if not set up correctly. The key challenge is ensuring proper linking and resource resolution within the plugin bundle rather than relying on system-wide installation in /etc/vulkan.

*Tags: `deployment`, `macos`, `plugin`, `reference`, `vulkan`*

---

### What OpenGL versions do popular After Effects effects use, and which ones work well?

Based on developer feedback, popular AE effects use varying OpenGL versions. Element 3D and Trapcode use OpenGL 4.x, while other effects like Helium maintain OpenGL 3.3. The questioner notes they previously used OpenGL 3.3 before migrating to Vulkan. OpenGL 3.3 appears to be a stable choice used by multiple working effects.

*Tags: `cross-platform`, `gpu`, `opengl`, `reference`*

---

### What is the official MoltenVK documentation for developing Vulkan applications on macOS?

The KhronosGroup MoltenVK GitHub repository contains detailed documentation on developing Vulkan applications for macOS, iOS, and tvOS, with careful guidance on environment setup and integration. Available at: https://github.com/KhronosGroup/MoltenVK/blob/0fe5ffecc5ae8a1ad072d3c95ea22b99f2cdc6be/README.md#developing-vulkan-applications-for-macos-ios-and-tvos

*Tags: `documentation`, `macos`, `moltenvk`, `reference`, `vulkan`*

---

### How can I get all of the active camera's properties in an After Effects plugin?

The Resizer example in the After Effects SDK demonstrates how to work with camera layers and access their properties. This example is a good reference for interfacing with camera layers and retrieving their active properties.

*Tags: `aegp`, `reference`, `sdk`*

---

### What resources explain how to convert After Effects camera matrices to OpenGL format?

Two archived Adobe Community threads provide detailed guidance on matrix coordinate system conversion for AE camera data. The first thread at https://community.adobe.com/t5/after-effects-discussions/glator-for-dummies-and-from-dummy/m-p/6930311 discusses camera matrix conversion generally. The second thread (archived at https://web.archive.org/web/20111223234233/http://forums.adobe.com/thread/570135) provides specific code examples for transposing AE matrices to OpenGL format, explaining the row-major vs column-major difference and matrix operations needed.

*Tags: `aegp`, `camera`, `matrix`, `opengl`, `reference`*

---

### What drawing libraries or tools are recommended for After Effects plugin development?

Cairo and DrawBot are recommended options for drawing in After Effects plugins. The canvas suite should not be confused with a drawing toolkit—it actually refers to the rendering pipeline rather than a suite for drawing on the viewport.

*Tags: `reference`, `tool`, `ui`*

---

### Where can I find discussion and solutions about PF_PROGRESS bar not showing in After Effects plugins?

Adobe Community has a discussion thread about PF_PROGRESS bar issues: https://community.adobe.com/t5/after-effects-discussions/progress-bar-won-t-show-using-pf-progress/m-p/10474224. The thread discusses limitations of progress reporting in the render function, particularly when the main thread is blocked during computation.

*Tags: `debugging`, `reference`, `ui`*

---

### Can arbitrary parameters with PF_PUI_INVISIBLE flag be used to store custom data in After Effects plugins?

Yes, arbitrary (Arb) parameters can be used to store custom data invisibly in plugin parameters. However, there are important constraints: the data structure must be serializable and cannot contain C++ std::string types. Instead, use fixed-size char arrays (char[size]) for string data, as std::string cannot be flattened or saved to disk. Refer to the ColorGrid example in the SDK for proper Arb parameter setup including special functions for initialization and updates.

*Tags: `arb-data`, `params`, `reference`, `ui`*

---

### What is the Adobe ae-plugin-thread-safety repository?

Adobe's ae-plugin-thread-safety repository at https://github.com/adobe/ae-plugin-thread-safety contains examples and best practices for handling thread-safe operations in After Effects plugins, including atomic variable usage and mutex patterns.

*Tags: `open-source`, `reference`, `threading`, `tool`*

---

### Is there an open-source After Effects plugin example that uses CMake for build system setup?

The aftereffects_spatial_media_plugins repository by Gorialis uses CMake for build configuration, allowing developers to set up the build system themselves rather than relying on platform-specific templates. This approach is appealing for creating new plugins with a more understandable build process. Repository: https://github.com/Gorialis/aftereffects_spatial_media_plugins/

*Tags: `build`, `cmake`, `open-source`, `reference`*

---

### Is there an example After Effects plugin that uses Vulkan for GPU acceleration?

The Vulkanator project by Wunkolo demonstrates Vulkan integration with After Effects plugins for GPU-accelerated rendering. Repository: https://github.com/Wunkolo/Vulkanator

*Tags: `gpu`, `open-source`, `reference`, `vulkan`*

---

### Is the PF_Arbitrary_NEW_FUNC selector called in After Effects post-CS6 architecture?

No, PF_Arbitrary_NEW_FUNC is not called in the post-CS6 architecture. Instead, arbitrary data is initialized by allocating a default Arb handle during ParamSetup by calling a custom function to create the default arb data. For every new instance, After Effects sends a PF_Arbitrary_COPY_FUNC selector with your default arb handle. While the NEW_FUNC code can be left in place just in case, it will not be triggered in practice.

*Tags: `aegp`, `arb-data`, `params`, `reference`*

---

### What should you assume about the data pointer in pF_effectworld when using GPU mode in After Effects plugins?

According to the SDK documentation, when using GPU mode (CUDA, Metal, or OpenCL), you must assume that the data pointer is NULL in the pF_effectworld structure. This means the pixel data is not available in system memory and you need to work with GPU-resident data instead.

*Tags: `cuda`, `gpu`, `metal`, `mfr`, `opencl`, `reference`*

---

### What is a good approach to manage memory safety for After Effects SDK objects?

Create C++ wrapper classes around AESDK objects to enable automatic memory management. This approach leverages C++ features like constructors and destructors to ensure proper allocation and deallocation of SDK objects. Additionally, there are already existing C++ wrappers for portions of the After Effects SDK available on GitHub that can be referenced or reused for this purpose.

*Tags: `aegp`, `build`, `memory`, `open-source`, `reference`*

---

### How can I get text layer path vertices in true layer space that match expression path.points() behavior, especially when the layer is parented?

PF_PathDataSuite does not return vertices in true layer space as documented in the SDK. Instead, it returns vertices in a hybrid space that is affected by 2D transforms (xy-translation, shear from parented transforms, Z Rotation) but not by z-translation, Orientation, or X/Y Rotation properties. To convert PF_PathDataSuite output to true layer space (matching what path.points() returns in expressions), you need to reverse the 2D transforms while preserving the unaffected 3D properties. Use AEGP_GetLayerToWorldXform() to convert to world space after obtaining layer-space vertices. The discrepancy exists because PF_PathDataSuite appears to return vertices in a space between composition and layer space, unlike expressions which correctly return layer-space coordinates independent of layer transform properties.

*Tags: `aegp`, `cross-platform`, `layer-checkout`, `params`, `reference`*

---

### What is the official Adobe documentation for working with path data in After Effects plugins?

The Adobe After Effects Plugin SDK documentation on working with paths is available at https://ae-plugins.docsforadobe.dev/effect-details/working-with-paths.html. This guide describes PF_PathDataSuite and claims that returned vertices are in the layer's coordinate space, though developers have found discrepancies between this documentation and actual behavior when dealing with transformed and parented layers.

*Tags: `aegp`, `documentation`, `reference`*

---

### What is the Adobe After Effects expressions documentation for layer space transformations?

The Adobe After Effects expressions documentation for layer space is available at https://ae-expressions.docsforadobe.dev/layer-space.html. This reference defines how layer space works in expressions and how path.points() correctly returns vertices in true layer space, independent of the layer's transform properties.

*Tags: `documentation`, `reference`, `scripting`*

---

### What is the difference between checkout_layer_pixels and PF_Checkout?

checkout_layer_pixels should be used instead of PF_Checkout for layer pixel operations in After Effects plugins.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

### What is an easy way to calculate and verify the correct bitmask values for PiPL outflags and outflags2 fields?

Tobias Fleischer created a quick online lookup/cheat sheet tool in JavaScript that allows developers to easily find and verify the correct bitmask values for PiPL outflags and outflags2 fields without needing to run the host application and receive warnings. The tool is available at: https://www.reduxfx.com/piplflags/

*Tags: `build`, `debugging`, `pipl`, `reference`, `tool`*

---

### How should output flags be defined in an After Effects plugin to ensure they auto-update during compilation?

Define output flags using a macro in your plugin header file, then reference that macro in both the PiPL AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 entries. This ensures the PiPL automatically updates when you recompile. However, be aware that using higher bits/newest flags with this approach can cause overflow in Adobe's PiPL compiler, so for newer flags you may need to revert to manually defining flags as individual bits.

```cpp
#define FX_OUT_FLAGS (  PF_OutFlag_FORCE_RERENDER + \
                         PF_OutFlag_USE_OUTPUT_EXTENT + \
                         PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                         PF_OutFlag_DEEP_COLOR_AWARE + \
                         PF_OutFlag_CUSTOM_UI + \
                         PF_OutFlag_I_DO_DIALOG )

// In PiPL:
AE_Effect_Global_OutFlags {
    FX_OUT_FLAGS
},
AE_Effect_Global_OutFlags_2 {
    FX_OUT_FLAGS2
}
```

*Tags: `build`, `params`, `pipl`, `reference`*

---

### What is a tool for calculating PiPL outflags and outflags2 bitmasks?

Tobias Fleischer (reduxFX) created an online lookup/cheat sheet for calculating and evaluating bitmasks for the outflags and outflags2 fields in PiPL files. Available at https://reduxfx.com/aeoutflags.htm

*Tags: `deployment`, `pipl`, `reference`, `tool`*

---

### What are the size limitations for storing custom metadata in After Effects projects using xmpPacket from ExtendScript?

Custom metadata stored via xmpPacket in ExtendScript has a practical size limit of less than 500kb. After Effects itself stores significantly more than 500kb of data in its XMP packet for reference. The metadata is not regulated by the undo manager and does not appear to be undoable.

*Tags: `arb-data`, `reference`, `scripting`*

---

### Are there tools available that can remove XMP packets from After Effects project files to reduce file size?

Yes, tools like AE Viewer are available that can remove the XMP packet from After Effects files, significantly reducing file sizes (e.g., from 50mb down to 50kb). This demonstrates that AE bloats project files with substantial XMP metadata that can be stripped if not needed.

*Tags: `deployment`, `reference`, `tool`*

---

### What is plugplug.DLL and how can it be used for inter-plugin communication?

plugplug.DLL is a DLL that allows After Effects plugins to communicate with each other. You can call the extern "C" functions exposed by plugplug.DLL from your C++ plugin to enable inter-plugin communication. This concept is related to loading plugins from within other plugins, as discussed in the Adobe community post: https://community.adobe.com/t5/after-effects-discussions/how-to-load-a-plugin-from-another-plugin/td-p/6738182

*Tags: `aegp`, `cross-platform`, `deployment`, `reference`, `windows`*

---

### How can you reverse-engineer and integrate a DLL into an After Effects plugin?

One approach is to use depends.exe to inspect the DLL's dependencies and exports, then reference existing samples (like the Illustrator sample) as a guide to understand the integration pattern. While not always a 1:1 match, samples can provide enough context to build your own wrapper. Modern approaches may involve directly linking to the DLL and providing a header file, but manual reverse-engineering through dependency analysis and sample-based implementation has proven effective.

*Tags: `build`, `debugging`, `reference`, `windows`*

---

### When is PF_Cmd_GLOBAL_SETUP called in After Effects versus Premiere Pro?

In After Effects, PF_Cmd_GLOBAL_SETUP is called the first time the user applies the effect plugin to a layer. In Premiere Pro, it is called when the application is loading. The timing differs between the two host applications.

*Tags: `aegp`, `pipl`, `premiere`, `reference`*

---

### What is the documentation for using sequence data in After Effects effect plugins?

Adobe's official documentation on Global Sequence Frame Data is available at https://ae-plugins.docsforadobe.dev/effect-details/global-sequence-frame-data.html#validating-sequence-data. This documentation provides guidance on sequence data validation and includes simulation plugins as a primary example use case.

*Tags: `aegp`, `documentation`, `reference`, `sequence-data`*

---

### Should I manually allocate memory in After Effects effect plugins, or follow the sample plugins' approach?

You should not manually allocate memory in effect plugins. Instead, follow what the sample plugins are doing. Manual allocation is problematic because: (1) After Effects won't know whether to deallocate using delete[] or free(), (2) AE won't be able to track memory usage for optimization and management, and (3) it can lead to memory leaks or corruption. Let AE manage memory allocation and deallocation through its own APIs.

*Tags: `best-practices`, `effect-plugin`, `memory`, `reference`*

---

### How do I properly sample and write pixels to a PF_LayerDef respecting rowbytes alignment?

Use pointer arithmetic to calculate pixel addresses based on rowbytes. The formula is: (char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)). This ensures proper memory alignment. Reference: https://ae-plugins.docsforadobe.dev/effect-details/tips-tricks.html?highlight=center%20of%20a%20pixel#sampling-pixels-at-x-y

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data +
    (y * def.rowbytes) +
    (x * sizeof(PF_Pixel)));
}
```

*Tags: `aegp`, `memory`, `output-rect`, `reference`*

---

### When do I need to use PF_Cmd_FRAME_SETUP to customize output size in AE plugins?

You only need to use PF_Cmd_FRAME_SETUP (https://ae-plugins.docsforadobe.dev/effect-basics/command-selectors.html) if your effect expands the output buffer size, such as glow or drop shadow effects. For effects that maintain the same output dimensions as input, this step is not necessary.

*Tags: `aegp`, `output-rect`, `reference`*

---

### What is the official Adobe documentation for pixel sampling and layer manipulation in After Effects plugins?

Adobe provides comprehensive documentation at https://ae-plugins.docsforadobe.dev/effect-details/tips-tricks.html covering sampling pixels at X,Y coordinates and related plugin development techniques.

*Tags: `aegp`, `debugging`, `pipl`, `reference`*

---

### What is the OpenCV Mat step parameter and how does it work?

The step parameter in OpenCV's Mat constructor specifies the number of bytes between consecutive rows in the matrix. This is useful when working with data that has custom row alignment or stride, such as pixel buffers from After Effects. See: https://docs.opencv.org/4.x/d3/d63/classcv_1_1Mat.html#a51615ebf17a64c968df0bf49b4de6a3a

*Tags: `memory`, `opencv`, `reference`*

---

### Why hasn't someone created an OpenGL-based realtime GPU renderer plugin for After Effects that can handle thousands of layers as sprites?

This is a theoretical discussion about a potential feature gap in After Effects plugin ecosystem. The concept would involve creating a custom 3D renderer plugin that could leverage GPU acceleration to render 10,000+ layers in realtime by treating them as sprites, similar to what exists in other software like Artisan. No concrete answer or existing solution was provided in the conversation.

*Tags: `gpu`, `opengl`, `reference`, `render-loop`*

---

### What is the purpose of the changedB parameter in PathOutline, and can paths be modified through the SDK?

The changedB parameter appears to signal whether path data has been modified, though the exact mechanism is unclear. The public SDK does not expose functions for modifying paths—they are read-only. It's theorized that After Effects internally uses an undocumented API for path modification that mirrors the PathOutline disposal API, but this functionality is not available to plugin developers.

*Tags: `aegp`, `params`, `reference`, `sdk`*

---

### Is there a CMake-based build system available for After Effects plugins?

Yes, there is a CMake/Ninja setup available at https://github.com/mobile-bungalow/after_effects_cmake which is based on vulkanator's cmake setup. This uses pipltool.exe for PiPL compilation.

*Tags: `build`, `cmake`, `open-source`, `pipl`, `reference`*

---

### What is an alternative cross-platform build system for After Effects plugins with a custom PiPL compiler?

The virtualritz/after-effects project at https://github.com/virtualritz/after-effects provides its own cross-platform build system with a custom PiPL compiler written in Rust, offering an alternative to traditional build approaches.

*Tags: `build`, `cross-platform`, `open-source`, `pipl`, `reference`, `rust`*

---

### Is there an open-source reference for After Effects plugin development and layer handling?

The virtualritz/after-effects GitHub repository (https://github.com/virtualritz/after-effects) contains useful reference code for plugin development, including examples of working with PF_LayerDef structures, bit depth detection, and other AEGP-related functionality.

*Tags: `aegp`, `layer-checkout`, `open-source`, `reference`*

---

### Is there a reference repository documenting After Effects SDK structures and flags?

The virtualritz/after-effects repository on GitHub contains useful references for understanding PF_LayerDef structures and world_flags behavior, including documented examples of how to extract bit depth information from layer definitions.

*Tags: `mfr`, `open-source`, `reference`, `tool`*

---

### Is there an open-source reference for After Effects plugin development in Rust?

Yes, the virtualritz/after-effects repository (https://github.com/virtualritz/after-effects) provides Rust bindings and examples for After Effects plugin development. It includes practical code examples and API patterns learned from real plugin development, including handling 3D rendering integration and bit-depth detection. The crate was published as a result of production plugin development experience.

*Tags: `build`, `open-source`, `reference`, `tool`*

---

### Is there an open-source Rust crate for After Effects plugin development?

The virtualritz/after-effects repository (https://github.com/virtualritz/after-effects) is an open-source Rust crate for AE plugin development that includes f32 support. According to the repository owner, it was developed based on SDK examples, After Effects SDK mailing list discussions, and reverse-engineering from the AtomKraft for AE plugin. The crate was initially published after the author completed a year-long Rust plugin project integrating a 3D renderer, where f32 support proved essential.

*Tags: `build`, `open-source`, `reference`, `rust`*

---

### What is Maxon Studio and how does it relate to project templates?

Maxon Studio is a template tool that transforms compositions into capsules which can be reused as templates to regenerate projects. It is similar to the AEGP ProjDumper sample but with significantly more features. The tool has been internal since inception, with initial release delayed due to lacking features and UX. It is being prepared for wider studio release with plans to capture arbitrary plugin data into capsules for reuse across 3rd party and Adobe stock plugins.

*Tags: `aegp`, `arb-data`, `reference`, `tool`*

---

### Is there a reference implementation of dynamic dropdown list handling in After Effects plugins?

Diffusae (an AE plugin project) has a working implementation for updating dropdown parameters like a models list. It uses the AEGP StreamSuite approach with AEGP_GetNewEffectStreamByIndex and AEGP_SetStreamValue called from UpdateParamsUI. This approach is more reliable than directly manipulating param_union.pd.u.namesptr.

*Tags: `aegp`, `open-source`, `params`, `reference`, `ui`*

---

### Why do lock/unlock functions still exist in the After Effects SDK if they weren't ported to 64-bit?

According to Tobias Fleischer, the lock/unlock functions were not ported when the AE codebase moved to 64-bit in CS6 (2011), yet they still exist in the current SDK and sample code calls them. While rowbyte notes these functions are somewhat redundant in a 64-bit address space, they still provide a safe way to dereference handles (which are effectively double pointers in 64-bit) without confusion. The SDK provides a DH macro as an alternative for direct dereferencing.

*Tags: `64-bit`, `aegp`, `memory`, `reference`, `sdk`*

---

### Is there a reference Adobe forum post discussing plugin loading failures on macOS?

Yes, there is an Adobe Community forum post discussing plugin load failures in After Effects 25.2 and 25.3 on macOS: https://community.adobe.com/t5/after-effects-beta-discussions/my-plugins-fails-to-load-in-25-2-and-25-3-on-macos/m-p/15192946. This post documents the issue where plugins don't appear to be signed correctly, which is now a requirement on Apple Silicon Macs.

*Tags: `apple-silicon`, `debugging`, `deployment`, `macos`, `reference`*

---

### Is there an open-source Rust binding for After Effects plugin development?

The virtualritz/after-effects repository on GitHub provides Rust bindings for After Effects plugin development. This project enables developers to write AE plugins using Rust instead of C++.

*Tags: `build`, `cross-platform`, `open-source`, `reference`, `rust`*

---

### Is there an open-source Rust binding library for After Effects plugin development?

The virtualritz/after-effects repository on GitHub provides Rust bindings for After Effects plugin development. It enables developers to write AE plugins in Rust with support for debugging on macOS using standard debuggers.

*Tags: `macos`, `open-source`, `reference`, `rust`*

---

### Does the old GLator sample have memory leaks in its GL rendering code?

Yes, the GLator sample has significant memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, which means bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBOs.

*Tags: `deprecated`, `gpu`, `memory`, `opengl`, `reference`*

---

### What is a good resource for understanding scope guards and their implementation in C++?

Alex Bizeau from maxon mentioned that scope guards are smart pointers for anything that can use lambda functions. They're valuable memory safety tools because they handle deletion automatically on throwing and return statements without requiring explicit deletion handling at every return point. Maxon has their own implementation, with examples available in their codebase.

*Tags: `debugging`, `memory`, `open-source`, `reference`, `tool`*

---

### What is a good tool for managing resource cleanup and memory safety in C++ plugin code?

Scope guards are a useful pattern for memory safety in C++ code. They act like smart pointers for any resource, allowing you to use lambda functions to handle cleanup on scope exit. This is particularly valuable for handling exceptions and early returns without explicitly managing deletion at every exit point. Alex Bizeau from maxon recommended the scope_guard library as a reference implementation: https://github.com/ricab/scope_guard/blob/main/scope_guard.hpp. Scope guards work well with both smart pointers and even malloc/free patterns, making them a favorite tool for resource management.

*Tags: `cpp`, `memory`, `reference`, `resource-management`, `smart-pointer`, `tool`*

---

### What is a command ID tool for After Effects plugin development?

Justin shared a command ID tool that helps developers work with After Effects commands. The tool was highlighted in a recent LinkedIn post by Justin and is useful for plugin development workflows.

*Tags: `aegp`, `debugging`, `reference`, `tool`*

---

### How can you find and use the 'Reveal in Composition' command ID in After Effects scripting?

The 'Reveal in Composition' command ID can be found using app.findMenuCommandId('Reveal in Composition'), which returns 2775. However, this command may not work in all contexts and might return null if the menu is just a category menu rather than an executable command. An alternative approach is to mimic the behavior manually by opening the desired composition and selecting the desired layer, which often produces better results than relying on the command ID.

```cpp
app.findMenuCommandId('Reveal in Composition'); /// 2775
```

*Tags: `aegp`, `reference`, `scripting`, `ui`*

---

### What is the command ID for 'Reveal in Timeline' in After Effects?

The 'Reveal in Timeline' command ID is 2536, though it may require specific context to work properly or may not function as expected in all scenarios. Users attempting to use this command should be aware that it may need particular conditions to be met or may behave differently than anticipated.

*Tags: `aegp`, `reference`, `scripting`, `ui`*

---

### What should be considered when mapping After Effects command IDs to avoid conflicts?

When creating mappings of AE command IDs, avoid treating keys as unique since a single command name can map to multiple IDs with different meanings. For example, 'undo' maps to both ID 16 (the actual undo command) and ID 2371 (clear undo), but only one will display. Ensure the correct ID is shown for each command's actual function.

*Tags: `aegp`, `debugging`, `reference`, `scripting`*

---

### Are there duplicate command IDs in After Effects command ID mappings that developers should be aware of?

Yes, there are duplicate command IDs in After Effects command ID mappings. For example, 'undo' appears as both ID 16 and ID 2371 in JSON files, but they represent different commands—ID 16 is the actual 'undo' command while ID 2371 is 'clear undo'. Developers should avoid treating keys as unique and should verify which ID corresponds to the intended command functionality.

*Tags: `aegp`, `debugging`, `reference`*

---

### Are there alternatives to PiPL for describing plugin entrypoints in After Effects plugins?

Yes, PiPL is considered somewhat deprecated. There are alternative ways to describe plugin entrypoints, though the conversation does not specify the exact modern replacement method. Developers should investigate newer plugin descriptor formats beyond the traditional PiPL approach.

*Tags: `aegp`, `deprecated`, `pipl`, `reference`*

---

### What are the new entry points for After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. PIPL is no longer used by Premiere Pro, so developers only need to write PIPL for After Effects. Some investigation suggests it may be possible to remove PIPL entirely by using these new entry points, though the exact mechanism is not yet fully documented.

*Tags: `EffectMain`, `aegp`, `entry-point`, `pipl`, `reference`*

---

### When was the URL-based plugin registration method added to After Effects?

The URL-based plugin registration version was added two versions ago in After Effects 23, according to tlafo. This allows plugins to be registered without needing traditional PIPL resources.

*Tags: `aegp`, `deployment`, `pipl`, `reference`*

---

### How can you implement debouncing for callback functions in C++?

Alex Bizeau from maxon shared a template function that implements debouncing using std::chrono for timing and std::thread for delayed execution. The debounce function wraps a callback and returns a new function that only executes the original callback after a specified delay has passed without additional calls. It uses a shared_ptr to track the last call time and spawns a detached thread that checks if enough time has elapsed before invoking the original function.

```cpp
#include <chrono> #include <functional> #include <thread>  template <typename Func, typename... Args> std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {     std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =         std::make_shared<std::chrono::steady_clock::time_point>();      return [=](Args... args) {         *lastCallTime = std::chrono::steady_clock::now();         std::thread([=]() {             std::this_thread::sleep_for(delay);             if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {                 func(args...);             }         }).detach();     }; }
```

*Tags: `performance`, `reference`, `threading`*

---

### What are alternative approaches to accessing text layer properties when the standard AEGP text API is broken?

Alex Bizeau shared that the Kerning API in ExtendScript is broken, but works well in Expression Script. A workaround is to use ExtendScript to write an expression to a text layer that writes values to a separate text layer, then read that second layer's value back in ExtendScript. This bypasses the broken AEGP text kerning API. While hacky, this demonstrates using expressions as an intermediary to work around AE API limitations.

*Tags: `aegp`, `reference`, `scripting`, `ui`*

---

### How can I detect when a plugin loads an old project to change default UI visibility between versions?

You can use the After Effects documentation to detect project compatibility. When loading an old V1 project in V2, check if legacy parameters exist or have non-default values during initialization, then programmatically set your UI mode (Simple vs Advanced) accordingly. The actual implementation details are found in the official AE SDK documentation.

*Tags: `params`, `reference`, `scripting`, `ui`*

---

### Is there an open-source After Effects plugin project that demonstrates parameter manipulation?

ISF4AE is an open-source project on GitHub that demonstrates various parameter handling techniques for After Effects plugins, including dynamic parameter visibility and manipulation. It can be a useful reference for plugin developers working with parameters.

*Tags: `open-source`, `params`, `reference`*

---

### Is there an open-source example of hide/show functionality for After Effects plugins?

Alex Bizeau from Maxon mentioned the 36pix wrapper they created years ago as a working reference implementation for hide/show functionality in AE plugins. They also noted there is a special trick required for Premiere Pro but that hide/show is feasible to implement. The codebase is accessible to authorized developers.

*Tags: `open-source`, `premiere`, `reference`, `ui`*

---

### What are examples of After Effects plugins that were superseded by native Adobe features?

Several plugins have been made redundant by native Adobe implementations. James Whiffin's Thicc Stroke plugin was superseded when Adobe released native tapered strokes. Tobias Fleischer's VariFont plugin, which provided variable font support in After Effects, was eventually replaced when Adobe finally implemented native variable font support after years of the plugin being the only solution available.

*Tags: `open-source`, `reference`, `ui`*

---

### What happened to Transcriptive and how did Premiere's native transcription feature impact third-party plugins?

Transcriptive by Digital Anarchy was a popular plugin for years but Adobe introduced native transcription features in Premiere, making third-party solutions less necessary. Transcriptive's web services are ending in May 2026. Reference: https://digitalanarchy.com/blog/video-editing-plugins/transcriptive-end-of-life-web-services-will-be-ending-in-may-2026/

*Tags: `deployment`, `premiere`, `reference`*

---

### How did Adobe's variable font support in After Effects affect third-party VariFont plugins?

Tobias Fleischer (reduxFX) developed the VariFont plugin as the only way to use variable fonts in After Effects for an extended period. After years of waiting, Adobe finally implemented native variable font support in AE, making the third-party plugin less essential. Plugin developers anticipated this would eventually happen.

*Tags: `ae`, `deployment`, `reference`*

---

### Is there an open-source skeleton project for Vulkan SDK integration with After Effects plugins?

Eric CPFX shared a Skeleton project designed for using the Vulkan SDK with After Effects plugin development. This project serves as a reference implementation for developers looking to integrate Vulkan into their AE plugins.

*Tags: `build`, `open-source`, `reference`, `skeleton`, `vulkan`*

---

### What historical CPU architecture values were stored in the Gestalt API for After Effects plugins?

The Gestalt API values held gestalt values for CPU and FPU on 68x and PowerPC architectures. While the Gestalt API was deprecated on macOS since 10.8, Apple added values for Intel and Silicon ARM CPUs (values 10 and 20). However, Adobe no longer passes these values through to plugins.

*Tags: `apple-silicon`, `architecture`, `deprecated`, `macos`, `reference`*

---

### Where should After Effects plugin developers share and host code snippets?

Stack Overflow is recommended as a good platform for sharing code snippets and knowledge about After Effects SDK development. Using the tag 'After Effects SDK' makes the content searchable and discoverable. While GitHub could be used, Stack Overflow's forum structure is better suited for this niche community compared to chat systems which have limitations for code hosting.

*Tags: `deployment`, `open-source`, `reference`, `scripting`, `tool`*

---

### Is there an open-source After Effects plugin example available?

dvb metareal shared their After Effects plugin at https://omino.com/pixelblog, currently at version 0.9. This serves as a reference implementation for AE plugin development.

*Tags: `deployment`, `open-source`, `reference`*

---

### What is a good reference for understanding sequence data in After Effects plugin development?

Tobias Fleischer (reduxFX) has created comprehensive documentation about sequence data that has been very helpful for developers learning this concept. His guide clarifies how sequence data works in After Effects plugins.

*Tags: `aegp`, `documentation`, `reference`, `sequence-data`*

---

### Is PF_LayerDef::rowbytes always guaranteed to be a multiple of 4?

This question was asked but not answered in the conversation. No response or clarification was provided.

*Tags: `aegp`, `memory`, `reference`*

---

### Where should developers discuss OpenFX plugin development?

Alex Bizeau from Maxon recommends checking the official Slack channel of OpenFX instead of unofficial Discord servers for plugin development discussions.

*Tags: `community`, `openfx`, `reference`*

---

### Is there documentation available for debugging plugins in DaVinci Resolve?

Alex Bizeau from Maxon mentions there is official documentation available for debugging OpenFX plugins in DaVinci Resolve that can be helpful for developers working with that host.

*Tags: `debugging`, `openfx`, `reference`, `resolve`*

---

### Where can you find example code for handling Premiere Pro BGRA and YUVA colorspaces?

The sdknoise example in the After Effects SDK demonstrates how to properly handle BGRA and YUVA colorspaces for Premiere Pro plugins. This example shows the correct approach for colorspace handling without relying on problematic iterate suites.

*Tags: `bgra`, `colorspace`, `open-source`, `premiere`, `reference`, `yuva`*

---

### What is the best documentation to start learning UXP for Premiere Pro?

UXP in Premiere Pro is currently very limited but will change soon. There is a comprehensive article that contains all the relevant links to documentation, forums, and other resources to get started with UXP development for Premiere Pro.

*Tags: `documentation`, `premiere`, `reference`, `ui`, `uxp`*

---

### What is Bolt UXP?

Bolt UXP is a framework or resource for UXP development in Premiere Pro. It was shared as a relevant tool for developers working with UXP plugins.

*Tags: `premiere`, `reference`, `tool`, `ui`, `uxp`*

---

### How can I make a plugin support 8-bit, 16-bit, and 32-bit projects while sharing the same internal codebase without duplicating logic?

Use C++ templates to create multiple instances of your algorithm with different data types and range constants. This allows you to maintain a single shared codebase while handling different bit depths. Instead of letting After Effects automatically convert bit depths (which introduces color noise), you can manually handle the conversion within your plugin by instantiating template versions for 8-bit, 16-bit, and 32-bit processing.

*Tags: `build`, `mfr`, `params`, `reference`*

---

### Is there a working sample demonstrating buffer expansion in SmartFX?

Yes, shachar carmi created a working sample based on the 'smarty pants' sample that demonstrates buffer resizing in SmartFX. The sample is available at https://drive.google.com/file/d/1w4Fc9rtGXjU-YT4CC-6q01ft2MOvOLtt/view?usp=sharing. Additionally, an updated version that implements Gaussian blur with buffer expansion as the blur size grows is available at https://drive.google.com/file/d/1JWtvacj-jt5I8XfqxCzz3k_8lGi_nLaI/view?usp=sharing. These samples show how to properly structure the PreRender and SmartRender functions to handle expanded buffers correctly.

*Tags: `open-source`, `reference`, `smart-render`, `smartfx`*

---

### Where can you find an open-source directional blur plugin with buffer expansion implementation?

The AE-Motion repository on GitHub contains a Directional Blur plugin modified to use buffer expansion: https://github.com/NotYetEasy/AE-Motion/tree/main/DirectionalBlur. The repository includes all the plugin source code and can be compiled to test buffer expansion functionality in a real blur plugin implementation.

*Tags: `github`, `open-source`, `reference`, `smartfx`*

---

### How can I make an After Effects plugin save a new file when a UI button is clicked?

The After Effects SDK only handles importing files into projects or rendering to custom file types. To save arbitrary data to a file from a button click in a plugin, use standard C file I/O functions directly: fopen(), fwrite(), and fclose(). This is similar to how you would use .saveDlg(), .open(), .write(), and .close() in ExtendScript/JavaScript.

```cpp
fopen/fwrite/fclose
```

*Tags: `aegp`, `reference`, `sdk`, `ui`*

---

### Is there an example of passing strings from ExtendScript to a custom After Effects plugin via ExternalObject?

Yes, there is a working example provided by the community that demonstrates C external object communication with ExtendScript. The example shows how to structure a C external object for passing strings between ExtendScript and an After Effects plugin. Link: https://community.adobe.com/t5/after-effects-discussions/issue-passing-string-from-extendscript-to-custom-after-effects-plugin-via-externalobject/m-p/15019735#M259100

*Tags: `aegp`, `open-source`, `reference`, `scripting`*

---

### What is the correct first argument to pass to transform_world and other suite callbacks in After Effects plugins?

According to community expert Shachar Carmi, the Adobe documentation is incorrect about the first argument. Instead of passing in_data as documented, you should pass NULL or in_data->effect_ref. Passing NULL works because many suite callbacks accept null instead of effect_ref, and the PF_INTERRUPT macro internally uses effect_ref to make interrupt checks. The incorrect documentation may be legacy from before CC2015 when a separate rendering thread was introduced.

*Tags: `debugging`, `params`, `reference`, `sdk`, `transform_world`*

---

### What is the minimal code structure needed to implement smart render in an After Effects plugin?

The minimal smart render implementation requires a PreRender function that checks out the input layer with the correct channel mask and updates result rectangles, and a SmartRender function that checks out input and output buffers, handles parameter checkout, and processes the image according to bit depth (8, 16, or 32 bit). The code must properly union the result rectangles and always check in parameters even on error conditions.

```cpp
static PF_Err PreRender(PF_InData *in_data, PF_OutData *out_data, PF_PreRenderExtra *extra) {
  PF_Err err = PF_Err_NONE;
  PF_RenderRequest req = extra->input->output_request;
  PF_CheckoutResult in_result;
  req.channel_mask = PF_ChannelMask_ARGB;
  ERR(extra->cb->checkout_layer(in_data->effect_ref, EFFECT_INPUT, EFFECT_INPUT, &req, in_data->current_time, in_data->time_step, in_data->time_scale, &in_result));
  UnionLRect(&in_result.result_rect, &extra->output->result_rect);
  UnionLRect(&in_result.max_result_rect, &extra->output->max_result_rect);
  return err;
}

static PF_Err SmartRender(PF_InData *in_data, PF_OutData *out_data, PF_SmartRenderExtra *extra) {
  PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
  PF_EffectWorld *input_worldP = NULL, *output_worldP = NULL;
  ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, SHIFT_INPUT, &input_worldP));
  ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));
  if(input_worldP && output_worldP) {
    PF_ParamDef paramWhatever;
    AEFX_CLR_STRUCT(paramWhatever);
    ERR(PF_CHECKOUT_PARAM(in_data, EFFECT_PARAM, in_data->current_time, in_data->time_step, in_data->time_scale, &paramWhatever));
    switch (extra->input->bitdepth) {
      case 8: /* process 8 bit */ break;
      case 16: /* process 16 bit */ break;
      case 32: /* process 32 bit */ break;
    }
    ERR2(PF_CHECKIN_PARAM(in_data, &paramWhatever));
  }
  return err;
}
```

*Tags: `aegp`, `params`, `reference`, `smart-render`*

---

### How do you copy input to output without processing the image in a smart render effect?

Use the PF_COPY command to copy memory from input to output buffer. This command is independent of color depth and handles 8-bit, 16-bit, and 32-bit images automatically. The syntax is: ERR(PF_COPY(inputP, outputP, NULL, NULL));

```cpp
ERR(PF_COPY(inputP, outputP, NULL, NULL));
```

*Tags: `reference`, `smart-render`*

---

### How do you remove a proxy from a composition or footage in After Effects?

There are several ways to remove a proxy in After Effects. You can click the small red square icon next to your composition or footage in the Project window to toggle the proxy on and off. Alternatively, you can remove the proxy altogether from the Interpret Footage window. Another method is to select the Proxy object in the Project Panel, then go to File > Set Proxy... and choose None.

*Tags: `aegp`, `reference`, `ui`*

---

### Does After Effects provide built-in matrix multiplication functions?

After Effects does not offer native matrix handling functions. However, the Artie sample plugin contains code for multiplying 4x4 matrices. For 3x3 matrix multiplication, you can implement your own using standard matrix multiplication algorithms, such as the example from Stack Overflow: https://stackoverflow.com/questions/25250188/multiply-two-matrices-in-c

```cpp
for (int i = 0; i < a; i++)
  for (int j = 0; j < d; j++) {
    Mat3[i][j] = 0;
    for (int k = 0; k < c; k++)
      Mat3[i][j] += Mat1[i][k] * Mat2[k][j];
  }
```

*Tags: `reference`, `sdk`, `tool`*

---

### Can a custom 3D importer be written to behave like the .obj importer with 3D transforms and depth buffer support?

No, this is not possible with standard importer AEGPS (IO and FBIO plugins). Importers only parse a requested frame and pass it to After Effects as an image. True 3D behavior is handled by Artisan plugins, which perform full composition rendering. To achieve 3D import functionality, you would need to write an Artisan plugin, but this requires handling all rendering yourself, not just the 3D import.

*Tags: `3d`, `aegp`, `importer`, `reference`*

---

### How do you get the name of the selected layer in an After Effects plugin?

To get the name of the selected layer, first call AEGP_GetNewCollectionFromCompSelection() to get the current selection. Then traverse the collection using AEGP_GetCollectionNumItems() and AEGP_GetCollectionItemByIndex() to get individual collection items. Each item returns an AEGP_CollectionItemV2 structure. Check the 'type' field to identify if it's a layer. If it is a layer type, extract the AEGP_LayerH from the union field u.layer. Finally, pass this AEGP_LayerH handle to AEGP_GetLayerName() to retrieve the layer name.

```cpp
// Get selected layers
AEGP_CollectionH collectionH;
AEGP_GetNewCollectionFromCompSelection(NULL, &collectionH);

A_long num_items;
AEGP_GetCollectionNumItems(collectionH, &num_items);

for (A_long i = 0; i < num_items; i++) {
    AEGP_CollectionItemV2 item;
    AEGP_GetCollectionItemByIndex(collectionH, i, &item);
    
    if (item.type == AEGP_CollectionItemType_LAYER) {
        AEGP_LayerH layer_h = item.u.layer.layer_handle;
        A_UTF16Char layer_name[AEGP_MAX_LAYER_NAME_LEN];
        AEGP_GetLayerName(layer_h, layer_name);
    }
}
```

*Tags: `aegp`, `layer-checkout`, `reference`*

---

### How can I get the mouse position relative to the composition when the user clicks in the comp preview window?

Mouse coordinates are passed to the plugin through UI event callbacks. You can get the mouse click location in composition window coordinates and convert those into layer source coordinates. The CCU SDK sample project demonstrates how to implement this functionality with working code examples.

*Tags: `coords`, `mouse-input`, `reference`, `sdk`, `ui`*

---

### What is a good SDK sample project that demonstrates handling mouse clicks in the composition window?

The CCU SDK sample project is an official Adobe After Effects SDK example that shows how to get mouse click locations in comp window coordinates and convert them into layer source coordinates. It provides a complete working implementation for custom UI event handling.

*Tags: `open-source`, `reference`, `sample-project`, `sdk`, `ui`*

---

### What resources are available for implementing hot reloading in Xcode development?

For Xcode development, refer to the Stack Overflow discussion on instant run or hot reloading: https://stackoverflow.com/questions/42529081/instant-run-or-hot-reloading-for-xcode. Note that hot reloading was more common in 32-bit builds but became less practical with the transition to 64-bit architecture. Limitations exist when hot reloading is used with After Effects plugins, particularly around parameter setup.

*Tags: `build`, `debugging`, `macos`, `reference`, `tool`*

---

### Is there a sample plugin that demonstrates resizing the output frame?

The 'Resizer' SDK sample project is the recommended reference for learning how to resize output frames in After Effects plugins.

*Tags: `output-rect`, `reference`, `sdk`*

---

### How do you properly update UI parameter values using PF_Cmd_UPDATE_PARAMS_UI in After Effects plugins?

During UPDATE_PARAMS_UI, you should only change parameter appearance (hidden, disabled, etc.) and not values. To change values, set proper defaults during PARAM_SETUP instead. When updating parameters, do not modify values and flags directly on the original params array. Instead, make a copy of the param struct, modify the copy, and pass it back to PF_UpdateParamUI. Also ensure you set out_data->out_flags |= PF_OutFlag_REFRESH_UI before returning. Refer to the Supervisor SDK sample project for correct implementation details.

```cpp
static PF_Err UpdateUI(PF_InData* in_data, PF_OutData* out_data, PF_ParamDef* params[]) {
  PF_Err err = PF_Err_NONE;
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  // Make a copy, modify the copy, then use PF_UpdateParamUI
  ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref, SKELETON_COLOR, params[SKELETON_COLOR]));
  out_data->out_flags |= PF_OutFlag_REFRESH_UI;
  return err;
}
```

*Tags: `params`, `pipl`, `reference`, `ui`*

---

### What is the Supervisor SDK sample project and what does it demonstrate?

The Supervisor SDK sample project is an official Adobe After Effects SDK example that demonstrates proper parameter supervision and UI updating. While somewhat convoluted, the first few lines of its UpdateParameterUI() function show the correct implementation for using PF_UpdateParamUI with parameter copies and proper flag handling.

*Tags: `open-source`, `params`, `reference`, `sdk`, `ui`*

---

### How can I handle the save event in an After Effects plugin (AEGP)?

You can use the command_hook to detect save events, but it is not 100% reliable as sometimes saves occur without triggering a call. A more robust approach is to implement an idle_hook on your AEGP that checks if the project is dirty. Track the dirty flag state between hook calls—if the project transitions from dirty to clean between successive idle hook invocations, it indicates the user has saved the project.

*Tags: `aegp`, `reference`, `scripting`*

---

### How do you get the duration of a composition layer in the After Effects SDK for C++?

To get layer duration, use AEGP_GetLayerDuration(). To get composition duration, use AEGP_GetItemFromComp() followed by AEGP_GetItemDuration().

*Tags: `aegp`, `c++`, `reference`, `sdk`*

---

### Is there an official Adobe documentation reference for the compute cache API?

Yes, Adobe's official documentation for the compute cache API is available at https://ae-plugins.docsforadobe.dev/effect-details/compute-cache-api.html, which includes details about AEGP_ComputeCacheCallbacks, the generate_key function, and the AEGP_HashSuite1 for computing hashes.

*Tags: `aegp`, `compute-cache`, `documentation`, `reference`*

---

### How do you convert an A_Time value to a floating-point seconds value in the After Effects SDK?

A_Time is a rational type with two components: value and scale. To convert to seconds as an A_FpLong, divide the value by the scale: `A_FpLong currentTime = currT.value / currT.scale;`. Note that A_Time values are rationals and may not map exactly to floating point, potentially causing off-by-one frame issues, so for precision it's better to work directly with rational time operations.

```cpp
ERR(suites.ItemSuite6()->AEGP_GetItemCurrentTime(itemH, &currT));
A_FpLong currentTime = currT.value / currT.scale;
```

*Tags: `aegp`, `params`, `reference`*

---

### Where can you find documentation and definitions for After Effects SDK data types like A_Time and A_FpLong?

Most AE SDK types are not documented in the PDF reference. The best approach is to examine the SDK header files directly. In your IDE, right-click on any AE type and select 'Go to Definition' to jump to the header where the type is defined. For example, Pf_fp_long is defined as a double. The main documentation resource is https://ae-plugins.docsforadobe.dev/index.html, though it may have gaps.

*Tags: `aegp`, `debugging`, `reference`*

---

### Is there an official sample project demonstrating how to access and manipulate streams in the After Effects SDK?

Yes, Adobe provides the 'project dumper' sample project as part of the After Effects SDK. This sample demonstrates how to access streams and is useful for understanding how to affect parameters in a project using the StreamSuite.

*Tags: `aegp`, `open-source`, `reference`, `sdk`, `streaming`*

---

### How do I resolve 'Cannot open source file' errors when moving After Effects SDK example projects to a new location?

When copying AE SDK example projects like the supervisor folder to a new location, include directories only apply to .h and .hpp header files, not to .c and .cpp source files. Source files use relative paths from the Visual Studio project file location. To fix the error, either re-import the missing files or maintain the original SDK folder structure relative to where your .vcxproj file is located. Simply adjusting the include directories in project properties will not resolve paths for source files.

*Tags: `build`, `reference`, `sdk`, `windows`*

---

### What is a good SDK sample project to learn how to manage dynamic parameter visibility?

The 'Supervisor' SDK sample project is recommended as a reference for understanding how to use PF_UpdateParamUI for managing parameter visibility and behavior in After Effects plugins. It demonstrates techniques for controlling whether parameters appear in the Effect Control Window and timeline.

*Tags: `params`, `reference`, `sdk`, `ui`*

---

### What is a good library for saving EffectWorld pixel data to image files in AE plugins?

tinypng is a popular library for saving pixel buffer data to image files when working with AE plugin EffectWorld objects. You can access the pixel data via effect_worldP->data and then use tinypng to serialize it to disk.

*Tags: `memory`, `open-source`, `reference`, `tool`*

---

### How can I hide a custom arbitrary parameter from the timeline while keeping it visible in the Effect Controls Window?

If the custom arbitrary parameter is purely for UI purposes and doesn't need to vary over time, use PF_ADD_NULL instead of creating an arbitrary parameter. This approach avoids the <error> display in the timeline while still allowing UI controls in the ECW. The HistoGrid example demonstrates this pattern effectively.

```cpp
PF_ADD_NULL("parameter_name");
```

*Tags: `arb-data`, `params`, `reference`, `ui`*

---

### How do I retrieve a layer's position after checking it out in an After Effects plugin?

To get a layer's position, use AEGP_GetNewLayerStream with the AEGP_LayerStream_POSITION parameter. The origin_x and origin_y fields from a checked-out layer only give the 0,0 location within the grabbed buffer, not the layer's actual position in the composition. If you need the layer's source dimensions instead, use AEGP_GetNewStreamValue to get the AEGP_LayerIDVal, then AEGP_GetLayerFromLayerID, AEGP_GetLayerSourceItem, and finally AEGP_GetItemDimensions.

```cpp
AEFX_CLR_STRUCT(checkout);
ERR(PF_CHECKOUT_PARAM(in_data,
  PARENT_LAYER_ID,
  in_data->current_time,
  in_data->time_step,
  in_data->time_scale,
  &checkout));
// Use AEGP_GetNewLayerStream with AEGP_LayerStream_POSITION
// origin_x and origin_y only give buffer-relative coordinates, not composition position
```

*Tags: `aegp`, `layer-checkout`, `params`, `reference`*

---

### How do you get the composition width and height in the PF_Cmd_USER_CHANGED_PARAM function?

The in_data->width and height fields denote your effect's layer source item full resolution size (unmasked, unchanged by effects), not the composition size. If you need the actual composition dimensions, you need to get the effect's layer source item and retrieve the project item's dimensions from it. The output pointer will be NULL in PF_Cmd_USER_CHANGED_PARAM, unlike in SmartRender where you can use checkout_output.

*Tags: `aegp`, `params`, `reference`*

---

### Is there an open-source searchable database of After Effects command IDs?

Justin Taylor at Hyper Brew maintains an updated searchable Command ID list for After Effects 2022 at https://hyperbrew.co/blog/after-effects-command-ids/, which is useful for scripting and tool development. There was also a Bitbucket snippet repository at https://bitbucket.org/justin2taylor/workspace/snippets/aLjjBE with command ID resources.

*Tags: `extendscript`, `open-source`, `reference`, `scripting`, `tool`*

---

### What should the refconPV value be when creating arbitrary data parameters in After Effects plugins?

The refconPV is a pointer to whatever you'd like, and that pointer is passed back to the plugin during arbitrary data calls for that parameter. If you have different handler classes for each arbitrary parameter, you can pass a pointer to an instance of that class. If unused, you can pass null. The value 0xDEADBEEFDEADBEEF seen in examples is just programmer's leetspeak and not significant. This allows you to maintain separate state or handlers for multiple arbitrary data parameters in the same plugin.

```cpp
#define ARB_REFCON (void*)0xDEADBEEFDEADBEEF
PF_ADD_ARBITRARY2( "Color Grid",
UI_GRID_WIDTH,
UI_GRID_HEIGHT,
PF_PUI_CONTROL | PF_PUI_DONT_ERASE_CONTROL,
def.u.arb_d.dephault,
COLOR_GRID_UI,
ARB_REFCON);
```

*Tags: `arb-data`, `params`, `reference`, `ui`*

---

### Should I use PF_Param_FLOAT_SLIDER instead of the deprecated PF_Param_SLIDER for slider parameters in After Effects plugins?

According to the Adobe AE SDK documentation, PF_Param_SLIDER and PF_Param_FIX_SLIDER are deprecated in favor of PF_Param_FLOAT_SLIDER. However, PF_Param_SLIDER still works and hasn't been completely removed from the API. If you need to force integer-only sliders, PF_Param_SLIDER remains a viable option. For PF_Param_FLOAT_SLIDER, you can set precision to 0 and round values in param supervision, though this won't prevent keyframe interpolation from creating fractional values. Community experts suggest that PF_Param_SLIDER is still safe to use if integer sliders are a requirement.

*Tags: `params`, `reference`, `ui`*

---

### How can I localize parameter labels in After Effects plugins to display in different languages like Chinese?

Adobe provides official documentation on localization for After Effects plugins. The localization guide is available in the official After Effects SDK documentation at https://ae-plugins.docsforadobe.dev/intro/localization.html, which covers how to implement parameter label localization for different languages.

*Tags: `localization`, `params`, `reference`, `sdk`, `ui`*

---

### How can I access and modify layer styles like inner shadow using the AE SDK?

Use AEGP_DynamicStreamSuite4 to access layer styles. You need to explore the layer's dynamic stream groups to find the specific style properties you want to modify, such as inner shadow values.

*Tags: `aegp`, `layer-checkout`, `reference`, `sdk`*

---

### Is there a reference document explaining After Effects sequence data structure and behavior?

The Redux FX documentation provides comprehensive information about sequence data in After Effects plugins, including structure and calling conventions: https://reduxfx.com/ae_seqdata

*Tags: `documentation`, `reference`, `sequence-data`*

---

### How can I create a plugin that adds image layers and regenerates them when properties change?

This is possible by combining an AEGP panel plugin with an effect plugin. The AEGP panel (implemented similar to the SDK's 'Panelator' sample) creates and manages layers with an inspector window for user controls. The effect plugin handles rendering and stores configuration data using either hidden parameters (which can be read/written by the AEGP window) or sequence data (a memory chunk that stores arbitrary data without UI representation, undo stack entries, or reset behavior). The effect plugin can regenerate images whenever properties change, with the decisions made in the panel interface driving the effect's rendering behavior.

*Tags: `aegp`, `params`, `reference`, `sequence-data`, `ui`*

---

### What is the recommended sample project for learning how to create a dockable panel in After Effects?

The SDK sample project 'Panelator' demonstrates how to create a dockable panel in After Effects. This sample shows the fundamentals of building a panel interface that can be populated with custom controls and integrated into the AE workspace.

*Tags: `aegp`, `reference`, `sdk`, `ui`*

---

### How can an effect plugin determine what type of layer it is attached to?

Use AEGP_GetLayerObjectType() to determine the layer type. This function returns one of several values: AEGP_ObjectType_NONE (-1), AEGP_ObjectType_AV (includes all pre-AE 5.0 layer types including audio, video sources, and adjustment layers), AEGP_ObjectType_LIGHT, AEGP_ObjectType_CAMERA, AEGP_ObjectType_TEXT, or AEGP_ObjectType_VECTOR. To access the layer handle from within an effect, use AEGP_GetEffectLayer().

```cpp
AEGP_ObjectType_NONE = -1,
AEGP_ObjectType_AV, /* Includes all pre-AE 5.0 layer types (audio or video source, including adjustment layers) */
AEGP_ObjectType_LIGHT,
AEGP_ObjectType_CAMERA,
AEGP_ObjectType_TEXT,
AEGP_ObjectType_VECTOR,
```

*Tags: `aegp`, `params`, `reference`*

---

### How does the PF_FloatMatrix work and how do you use it with transform_world() to transform a PF_EffectWorld?

PF_FloatMatrix is a standard 3x3 transformation matrix used in graphics programming. It can perform translation, rotation, scaling, and skewing transformations all in one concatenated matrix. The matrix indices control different transformation properties: the first row typically contains x-translation and scaling/rotation components, while the third row contains y-translation and scaling/rotation components. When applying rotations using a matrix, you place sin and cos (and negative values) in specific positions following standard 2D/3D graphics conventions. Important note: After Effects uses row-based matrices, which is the opposite of the OpenGL standard, so you need to swap the axis accordingly. The transform_world() function applies the supplied matrix to transform the input image, and the result is transformed along with the layer. There is no "current transformation" state—the matrix is applied independently.

*Tags: `mfr`, `params`, `reference`, `transformation`*

---

### What is a good resource for understanding transformation matrices for use in After Effects plugins?

Wikipedia's article on Transformation Matrices (https://en.wikipedia.org/wiki/Transformation_matrix) is a good starting point for understanding how 3x3 matrices work in graphics. Be aware that After Effects uses row-based matrices, which differs from the OpenGL standard, so axis values need to be swapped accordingly. Standard transformation matrix tutorials and code samples from general graphics programming resources apply directly to AE plugin development.

*Tags: `open-source`, `reference`, `transformation`*

---

### What is the correct way to bypass rendering in an After Effects effect plugin?

To bypass rendering in an AE effect plugin, you have a few options: (1) Try setting the output rectangle during pre-render to a 0-sized rect or a rect where the right value is smaller than the left value to tell AE to skip the render, though this approach is not well-documented and may require experimentation. (2) The more reliable and practical approach is to manually copy the input buffer to the output buffer in your render function. According to community experts, the overhead of copying is negligible even when stacking multiple such effects, making this a pragmatic solution that works reliably on both audio and non-audio layers.

*Tags: `aegp`, `output-rect`, `reference`, `render-loop`*

---

### Is there a built-in After Effects flag to prevent an effect from rendering?

The PF_OutFlag_AUDIO_EFFECT_ONLY flag can be used to avoid rendering on non-audio layers, but it does not work for audio layers themselves. For a universal solution that works on all layer types, manually copying the input to the output during the render pass is the recommended approach rather than relying on flags or output rectangle tricks.

*Tags: `aegp`, `params`, `reference`*

---

### What is the recommended approach for changing parameter order in newer versions of an After Effects plugin while preserving saved project data?

Adobe's AE Plugin SDK Guide provides detailed documentation on changing parameter orders: https://ae-plugin-sdk-guide.readthedocs.io/effect-details/changing-parameter-orders.html. The guide explains how to properly handle parameter ID assignment and reordering to maintain backwards compatibility with projects saved using older plugin versions.

*Tags: `params`, `reference`, `sdk`, `versioning`*

---

### How can I load a file into a C++ plugin just once instead of every frame in the render function?

Use global_data or sequence_data to store the loaded dictionary outside the render function. global_data stores information common to all instances of an effect throughout an AE session and is not saved with the project, while sequence_data stores information specific to one applied instance of the effect and can be saved with the project. You can put any data you want in these memory handles. Refer to the "supervisor" sample project in the SDK for implementation details on how to use global_data.

*Tags: `aegp`, `caching`, `memory`, `reference`*

---

### Where can I find the official After Effects SDK guide?

The After Effects SDK guide is hosted at https://ae-plugin-sdk-guide.readthedocs.io/. This is the current location for the official documentation after previous links were moved.

*Tags: `aegp`, `documentation`, `reference`, `sdk`*

---

### How can you get the index of an effect on a layer in After Effects?

To find the effect index on a layer, get the stream reference of the first effect parameter using EffectSuite4, then use AEGP_GetNewParentStreamRef to get the effect's stream representation, and finally use AEGP_GetStreamIndexInParent to retrieve the effect index on the layer. This is the reverse operation of AEGP_GetLayerEffectByIndex.

*Tags: `aegp`, `params`, `reference`*

---

### How can I change the current time (CTI) in After Effects using an AEGP plugin?

To change the current time (CTI) with a plugin, first get the comp's item from the effect layer, then use AEGP_SetItemCurrentTime. The process involves: (1) using AEGP_GetEffectLayer to get the layer handle, (2) using AEGP_GetLayerParentComp to get the composition handle, (3) using AEGP_GetItemFromComp to get the item handle from the composition, and (4) finally calling AEGP_SetItemCurrentTime with the item handle and desired time value.

```cpp
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &m_layerH);
suites.LayerSuite8()->AEGP_GetLayerParentComp(m_layerH, &m_compH);
suites.CompSuite11()->AEGP_GetItemFromComp(m_compH, &compItemH);
suites.ItemSuite9()->AEGP_SetItemCurrentTime(compItemH, &compTime);
```

*Tags: `aegp`, `cti`, `reference`, `scripting`, `ui`*

---

### How should PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS be used correctly when adding new parameters to legacy plugin versions?

The flag PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS is meant to provide a default value for old projects that don't have the new parameter. However, the PF_ADD_CHECKBOX macro copies the passed default over the passed value, which can cause unexpected behavior. The solution is to either copy the macro code into your own files and fix it there, or manually define the parameter without using the macro, defining it the old-fashioned way. Avoid modifying the AE headers directly as SDK updates would undo the changes.

```cpp
def.u.bd.value = FALSE;        // value for legacy projects which did not have this param
PF_ADD_CHECKBOX(
    STR(StrID_Checkbox_Param_Name),
    STR(StrID_Checkbox_Description),
    TRUE, // value for new applications, and when reset
    PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS,
    DOWNSAMPLE_DISK_ID);
```

*Tags: `params`, `pipl`, `reference`, `sdk`*

---

### Is plugin_id always necessary when calling AEGP functions?

For the most part, you can pass NULL instead of the plugin_id to AEGP functions and they will work. However, there is one important exception: when registering command hooks, you should actually use the plugin_id, otherwise you might not get the hook/events you requested.

*Tags: `aegp`, `reference`, `sdk`*

---

### Why does an angle parameter show weird fixed-point values instead of the expected float value?

Angle parameters in After Effects use fixed-point representation internally. To convert a fixed-point angle value to float, use the FIX_2_FLOAT(X) macro. Alternatively, use the PF_AngleParamSuite1::PF_GetFloatingPointValueFromAngleDef() function from the SDK, which handles the conversion automatically.

```cpp
FIX_2_FLOAT(paramdef.u.ad)
```

*Tags: `aegp`, `params`, `reference`, `sdk`*

---

### How do you add static text labels to an After Effects effect UI?

You can use PF_ADD_TOPIC and PF_END_TOPIC macros without putting any params in between to create a text section. Alternatively, you can use PF_ADD_NULL, declare a minimal custom UI size, and not implement it.

*Tags: `params`, `pipl`, `reference`, `ui`*

---

### What does the PF_ prefix stand for in the After Effects SDK?

PF stands for 'Plug-in Filter', as opposed to AEGP which stands for 'After Effects General Plug-in'. The PF_ prefix is used throughout the SDK for parameters and classes related to filter plugins.

*Tags: `aegp`, `pipl`, `reference`, `sdk`*

---

### What functionality should be implemented in the Render function versus the 8-bit and 16-bit functions?

Your plugin's entry point function gets invoked with a RENDER or SMART_RENDER call. As long as your plugin responds to that call, you can implement whatever functionality you need. There is no strict separation required—you have flexibility in how you organize your plugin's main functionality across these functions.

*Tags: `aegp`, `pipl`, `reference`, `render-loop`*

---

### How can I access data from another plugin in After Effects?

Getting parameter data is relatively straightforward using both the C and JavaScript APIs, and the 'project dumper' sample project demonstrates this. However, there are two significant limitations: (1) accessing arbitrary parameter data (custom type params) is difficult—while you can retrieve the data chunk, deciphering its structure is challenging without knowing the format; (2) accessing sequence data (data stored internally by the plugin rather than in params) is only possible if the plugin provides specific custom infrastructure to expose it, otherwise only the plugin itself can access it.

*Tags: `arb-data`, `params`, `reference`, `scripting`, `sequence-data`*

---

### Is there a sample project that demonstrates how to extract parameter data from After Effects projects?

Yes, Adobe provides the 'project dumper' sample project as a reference implementation for getting parameter data from plugins using both the C and JavaScript APIs.

*Tags: `open-source`, `params`, `reference`, `scripting`*

---

### How can a plugin force an undo or redo action in After Effects?

Use AEGP_DoCommand() with command IDs: 16 for undo and 2035 for redo. Note that directly manipulating undo state is generally discouraged as a best practice.

```cpp
AEGP_DoCommand();
// undo = 16
// redo = 2035
```

*Tags: `aegp`, `reference`, `ui`*

---

### Can I implement a mirror effect using the transform_world function?

Yes, transform_world handles all 2D transformations including rotating, scaling, and moving. If 2D transformations are sufficient for your mirror effect, there is no reason not to use transform_world rather than manually calculating transformations.

*Tags: `mfr`, `params`, `reference`*

---

### How can I duplicate layers along a 3D path with auto-orientation in After Effects?

The After Effects API does not offer built-in tools for rendering 3D transformations. While transform_world() is available for 2D transformations, you need to implement your own 3D transform algorithm to render layer instances along a path. One approach is to calculate the transformation yourself and use transform_world() to fake a 3D look with scale and rotation. For a proof of concept, examine the "Artie" sample project which contains macros for getting XY coordinates of a texture using a 4x4 matrix. If antialiasing is not a concern, this code can provide a quick setup for a 3D transform function.

*Tags: `3d`, `aegp`, `path`, `reference`, `transform`*

---

### What is a good reference sample project for implementing 3D matrix transformations in After Effects plugins?

The "Artie" sample project included in the After Effects SDK contains macros for getting XY coordinates of a texture using a 4x4 matrix. This sample is useful as a proof of concept for implementing 3D transform functions when antialiasing is not a critical concern.

*Tags: `3d`, `matrix`, `open-source`, `reference`, `sample`, `sdk`*

---

### Is there an open-source example of using sequence_data and popup dialogs in an After Effects plugin?

The AE_tl_math plugin by crazylafo on GitHub demonstrates sequence data handling in an After Effects plugin. Available at: https://github.com/crazylafo/AE_tl_math (specifically the tl_math.cpp file). This example shows how to flatten sequences and work with data communication in plugins.

*Tags: `open-source`, `reference`, `sequence-data`, `ui`*

---

### Is there a sample project demonstrating how to implement arbitrary data parameters in After Effects?

The 'ColorGrid' sample project is a good reference for understanding how to properly handle arbitrary parameters. It demonstrates the correct implementation of ARB param flatten and unflatten operations.

*Tags: `arb-data`, `open-source`, `params`, `reference`*

---

### Where can I find a sample project demonstrating Drawbot usage?

The "CCU" sample project in the After Effects SDK demonstrates the use of Drawbot for custom UI drawing.

*Tags: `drawbot`, `reference`, `sample`, `ui`*

---

### How can I access suite functions from within a pixel iteration function?

Use the refcon (reference context) argument of the iterate() function to pass a struct containing pointers to the suites and any other data you need. Define a struct with your suite references and other variables, pass a pointer to an instance of this struct as the refcon argument to iterate(), and then cast it back to your struct type inside the pixel function to access the suites.

```cpp
typedef struct {
  PF_in_data *in_data;
  AEGP_Suites &suites;
  bool someVariable;
  float someOtherVariable;
} myIterationData;

// In calling function
myIterationData data;
data.suites = &suites;
data.someVariable = FALSE;
suites.Iterate16Suite1()->iterate(
  arg,
  arg,
  arg,
  (void*)&data
);

// In pixel function
myIterationData &data = *(myIterationData*)refcon;
if(data.someVariable) {
  data.suites.whateverSuite()->
}
```

*Tags: `aegp`, `mfr`, `reference`, `render-loop`*

---

### How can I access mask position and tangent information at a particular parameter for a stroke effect plugin?

Use the PF_PathEvalSegLengthDeriv1 function from the After Effects SDK. This function allows you to evaluate path segments and obtain derivative information, which provides the position and tangent data needed for stamping patterns along mask paths without having to write your own Bezier curve rasterizer.

*Tags: `aegp`, `mask`, `params`, `reference`*

---

### What sample projects and tutorials are available for learning After Effects plugin development?

There is a significant shortage of sample projects and tutorials for AE plugin development. Available resources include: (1) A very old MacTech tutorial on After Effects Plugins at http://www.mactech.com/articles/mactech/Vol.15/15.09/AfterEffectsPlugins/index.html, and (2) A paid course at https://www.fxphd.com/details/526/. Beyond these, the Adobe community forums contain extensive information ranging from basics to advanced topics. The SDK itself includes sample plugin codes such as Transformer and Skeleton that can be studied.

*Tags: `open-source`, `reference`, `sdk`, `tutorial`*

---

### Where can I find old but foundational tutorials on After Effects plugin development?

MacTech published a foundational tutorial on After Effects Plugins that, while older, provides essential learning material for plugin developers. URL: http://www.mactech.com/articles/mactech/Vol.15/15.09/AfterEffectsPlugins/index.html

*Tags: `aegp`, `reference`, `tutorial`*

---

### When should I use PreRender and SmartRender in After Effects plugins, and what are their benefits?

PreRender (PF_Cmd_SMART_PRE_RENDER) and SmartRender (PF_Cmd_SMART_RENDER) are supported only in After Effects and are mandatory if you wish to support 32bpc (32-bit per channel) rendering. They also offer rendering pipeline improvements. However, if your plugin needs to work in Premiere Pro as well, you must support the old rendering pipeline with FrameSetup() and regular Render() calls instead.

*Tags: `ae`, `mfr`, `reference`, `render-loop`, `smart-render`*

---

### Is there a sample plugin that demonstrates PreRender and SmartRender implementation?

The Shifter sample plugin included in the After Effects SDK demonstrates PreRender and SmartRender usage.

*Tags: `open-source`, `reference`, `sample`, `smart-render`*

---

### How can an After Effects plugin determine the index of the layer it is currently applied to?

Use the PFInterfaceSuite1 to get the effect layer, then use LayerSuite8 to get the layer index. First call AEGP_GetEffectLayer(in_data->effect_ref, &layerH) to get the layer handle, then call AEGP_GetLayerIndex(layerH, layer_indexPL) to retrieve the layer's index.

```cpp
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
suites.LayerSuite8()->AEGP_GetLayerIndex(layerH, layer_indexPL);
```

*Tags: `aegp`, `layer-checkout`, `reference`*

---

### How do you create a new composition from an imported image or footage item in After Effects?

There is no single command to create a composition directly from footage. Instead, you need to gather the item's data using AEGP_ItemSuite functions like AEGP_GetItemDimensions and AEGP_GetItemDuration to retrieve the necessary information, then use AEGP_CreateComp to create the new composition with that data.

*Tags: `aegp`, `reference`, `scripting`*

---

### Is there a reference in the SDK documentation about bundling AEGP plugins with effect plugins?

According to the SDK guide, if you want to add an AEGP plug-in to the same binary as an effect plugin, the PiPL of the AEGP must be the first one in the resource list. This allows you to avoid creating a separate installer while still gaining access to AEGP features like idle hooks for proper effect synchronization.

*Tags: `aegp`, `pipl`, `plugin-architecture`, `reference`, `sdk`*

---

### How can I trigger an event or function when the playhead crosses a marker in After Effects?

There is no direct marker event in the After Effects SDK. Instead, you can use an idle hook to monitor the composition's current time and detect marker crossings by comparing the previous time with the current time. Alternatively, you can listen to time change events and perform the same check. You may also investigate AEGP_RegisterCommandHook to see if a command is triggered on marker events.

*Tags: `aegp`, `marker-detection`, `reference`, `sdk`*

---

### How do modular plugins like Plexus work, where multiple plugins are stacked to produce a final result?

Modular plugins typically don't render anything themselves. Instead, the individual 'module' effects allow users to input parameters, while a main/master effect reads the parameter values from all the module effects and performs the actual rendering based on those combined parameter values. This is simpler and more elegant than trying to maintain off-screen buffers or create custom channels.

*Tags: `architecture`, `params`, `plugin-design`, `reference`*

---

### How do I access and manipulate individual RGB pixel values in an After Effects plugin?

To access individual pixels and their RGB values, examine the 'shifter' sample project included in the After Effects SDK. This sample demonstrates how to read and modify individual pixel data, which is fundamental for any plugin that needs to perform per-pixel color operations.

*Tags: `color`, `pixels`, `reference`, `sample`, `sdk`*

---

### How do I access pixel data from different frames in an After Effects plugin?

The 'checkout' sample project in the After Effects SDK demonstrates how to retrieve images from points in time other than the currently processed frame. This is useful for plugins that need to reference previous or future frames, such as temporal effects or frame-blending operations.

*Tags: `frame-access`, `layer-checkout`, `reference`, `sample`, `sdk`*

---

### How can you dynamically change effect parameter popup entries in After Effects plugins?

You cannot change the number of popup entries dynamically, but you can change the names of entries. The recommended approach is to create multiple hidden popups with different numbers of options (1, 2, 3, 4, 5, etc.) up to the maximum number of options needed, then show/hide the appropriate popup and populate its strings during ParamChange() or UI() callbacks. This allows the effect to appear to have dynamic popup content while working within SDK constraints.

*Tags: `aegp`, `params`, `reference`, `ui`*

---

### What is a good reference for dynamically modifying popup parameter strings in After Effects plugins?

Adobe's forum thread at https://forums.adobe.com/message/2446142#2446142 contains detailed explanation on how to modify popup entry names and manage hidden popups for dynamic parameter changes. Additionally, the thread 'Re: Questions before writing my plugin' discusses similar techniques for modifying parameters during ParamChange() and UI() calls.

*Tags: `debugging`, `params`, `reference`, `ui`*

---

### How can you get a layer handle from a composition handle to access composition settings like blending mode when working bottom-up through nested compositions?

There is no direct API to convert a composition handle (AEGP_CompH) to a layer handle (AEGP_LayerH). Instead, you must scan the project top-down: use AEGP_GetItemFromComp() to get the project item for the composition you're searching for, then scan all project items using AEGP_GetNextProjItem(). For each composition found, scan its layers using AEGP_GetLayerSourceItem() and match the layer's source with your target composition's source. This allows you to identify which layer in a parent composition references your nested composition, giving you the required AEGP_LayerH.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

### How do you recursively search for a folder in an After Effects project and get its contents?

There is no direct API to get a folder's child items. You need to iterate through all project items and use AEGP_GetItemParentFolder() to check which items belong to the folder you're searching for. For folders found within, repeat the process recursively. Alternatively, you can use AEGP_ExecuteScript() to retrieve item IDs through JavaScript (items use the same IDs in both C API and JavaScript), then continue processing on the C side.

```cpp
ERR(suites.ProjSuite5()->AEGP_GetProjectByIndex(0, &projH));
ERR(suites.ProjSuite5()->AEGP_GetProjectRootFolder(projH, &root_itemH));
ERR(suites.ItemSuite8()->AEGP_CreateNewFolder(folder_name, root_itemH, &root_folderH));
// Then iterate through items and check AEGP_GetItemParentFolder()
```

*Tags: `aegp`, `reference`, `scripting`*

---

### Can you keep AEGP handles like AEGP_ItemH, AEGP_CompH, and AEGP_LayerH across multiple plugin calls?

No, you cannot store these handles across plugin calls. AEGP_ItemH, AEGP_CompH, AEGP_LayerH, AEGP_StreamRefH, and AEGP_EffectRefH are only valid during a single call from After Effects to your effect. Once your plugin returns and AE regains control, these handles may become invalid. To track items across calls, store the item ID instead and search for it again on the next call. The same applies for layer handles using layer IDs.

*Tags: `aegp`, `memory`, `reference`*

---

### How can I draw gradients in an After Effects effect UI using DrawBot?

DrawBot does not have a direct API function for drawing gradients. The recommended approach is to draw the gradient into a buffer using any available means, then convert that buffer to a DrawBot image using the NewImageFromBuffer() function. This allows you to create dynamic gradient fills like color wheels for custom effect panels.

*Tags: `aegp`, `drawbot`, `reference`, `ui`*

---

### How can I retrieve keyframe values and interpolation data from After Effects parameters in a plugin?

Use the AEGP_GetNewStreamValue() function to get parameter values at any composition time. This function allows you to retrieve interpolated values based on keyframes and their temporal interpolation settings (such as Bezier). It is available in several SDK sample projects and will return the calculated value at any given frame, taking into account all keyframe interpolation between start and end frames.

*Tags: `aegp`, `params`, `reference`, `sdk`*

---

### What is a good example project to start learning After Effects plugin development?

The Convolutrix example project included in the After Effects SDK is recommended as a good beginner project to start learning plugin development. It demonstrates core concepts and can be compiled in Visual Studio to produce a working .aex plugin file.

*Tags: `beginner`, `example`, `reference`, `sdk`*

---

### How can I detect when a user selects a layer in After Effects?

There is no direct layer selection event in the AEGP API. However, you can use the idle_hook to periodically check if the composition's layer collection has changed, which allows you to detect when layer selection has occurred.

*Tags: `aegp`, `reference`, `ui`*

---

### How do you query a color picker parameter and get its floating-point values instead of 16-bit 0-255 range?

Use the PF_GetFloatingPointColorFromColorDef function from the ColorParamSuite1. Create a PF_PixelFloat variable and call suites.ColorParamSuite1()->PF_GetFloatingPointColorFromColorDef(in_data->effect_ref, params[COLOR_PICKER_INDEX], &fp_colorP) to convert the color parameter to floating-point representation. This allows you to work with color values in full floating-point precision rather than the default 16-bit packed color format.

```cpp
PF_PixelFloat fp_colorP;
suites.ColorParamSuite1()->PF_GetFloatingPointColorFromColorDef(
    in_data->effect_ref,
    params[MATTE_PICKER],
    &fp_colorP);
```

*Tags: `params`, `reference`, `sdk`, `ui`*

---

### What is the best way to store all pixels from a source layer into an array for processing?

Use the MemorySuite to allocate an int array of any size, then lock the memory handle onto an int pointer which will behave as an array. You can pass a pointer to your custom data as the refcon parameter to the Iterate8Suite, cast it in your iteration function, and fetch your int data according to the x,y coordinates provided by the suite. Alternatively, manually iterate through pixels using getXY() helper function to access PF_Pixel data by calculating the memory offset using rowbytes.

```cpp
static PF_Pixel *getXY(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

// In Render function:
for(int i = 0; i < tInfo->width; i++){
  for(int j = 0; j < tInfo->height; j++){
    PF_Pixel currentPixel = *getXY(*tInfo->input, i, j);
    int alpha = currentPixel.alpha;
    int red = currentPixel.red;
    int green = currentPixel.green;
    int blue = currentPixel.blue;
  }
}
```

*Tags: `aegp`, `memory`, `params`, `reference`*

---

### How can I programmatically detect when a user presses the play button in After Effects?

Use AEGP_RegisterCommandHook() to register a command hook that listens for play events. The relevant command numbers are: 2415 for Play (spacebar) and 2285 for RAM Preview. This allows you to be notified when the user triggers playback rather than implementing a custom onClick function.

```cpp
AEGP_RegisterCommandHook()
// Command numbers:
// 2285 - RAM Preview
// 2415 - Play (spacebar)
```

*Tags: `aegp`, `reference`, `scripting`, `ui`*

---

### Where can I find the source code for AEGP_GetNewStreamValue() and how to work with streams in After Effects plugins?

The After Effects SDK documentation and sample projects are available from Adobe's developer portal. The SDK Guide PDF and sample projects like ProjDumper, Streamie, and Mangler provide good examples of how streams are implemented. You can download the latest SDKs from http://www.adobe.com/devnet/aftereffects.html, and each SDK comes with an SDK guide in PDF format. For CS6 specifically, the download links are available at http://www.adobe.com/devnet/aftereffects/sdk/cs6_eula.html

*Tags: `aegp`, `reference`, `sdk`, `streams`*

---

### What are good sample projects to study for implementing stream data in After Effects plugins?

The ProjDumper, Streamie, and Mangler sample projects included in the After Effects SDK are recommended as good places to see streams implemented. These projects are included in the SDK download packages available from Adobe's developer portal.

*Tags: `aegp`, `open-source`, `reference`, `sample`*

---

### How do I use PF_SPRINTF to format messages in an After Effects plugin?

PF_SPRINTF is a macro that expects the same arguments as the standard C sprintf() function. It expands to (*in_data->utils->ansi.sprintf)(args...). The function takes a buffer (like out_data->return_msg) as the first argument, followed by a format string and any values to format. Do not pass in_data or other non-format arguments where sprintf() expects a format string. The macro must be used in a context where the in_data variable is available.

```cpp
PF_SPRINTF(out_data->return_msg, "test hello");
PF_SPRINTF(out_data->return_msg, "value: %d", some_int);
```

*Tags: `debugging`, `params`, `reference`, `ui`*

---

### What is the standard reference for sprintf() function usage?

The standard C sprintf() function is documented at http://www.cplusplus.com/reference/cstdio/sprintf/. This reference explains the format string syntax and argument requirements that apply to PF_SPRINTF macro calls in After Effects plugins.

*Tags: `reference`, `tool`*

---

### What language and tools should I use to develop an After Effects plugin for controlling layer properties and exporting data?

For your use case of creating a GUI, controlling layers and layer properties (content, effects, transform), and exporting data as XML, you don't need to develop a C plugin. Instead, use JavaScript/ExtendScript to create a script that can be encrypted as a .jsxbin file. Refer to the After Effects Scripting Guide PDF for detailed information on manipulating layers, effects, and properties. The AE Scripting forum at https://forums.adobe.com/community/aftereffects_general_discussion/ae_scripting is also a valuable resource for scripting questions.

*Tags: `deployment`, `reference`, `scripting`, `ui`*

---

### Where can I find the official After Effects SDK and scripting documentation?

The official After Effects SDK, scripting guide, and C API documentation can be found on Adobe's developer network at http://www.adobe.com/devnet/aftereffects.html. The scripting guide walks through JavaScript scripting, and the SDK file contains the C API docs for plug-in development.

*Tags: `reference`, `scripting`, `sdk`*

---

### How can I open a file browser dialog in an After Effects plugin to let users select a file?

There is no direct C API call in the SDK to open a file browser dialog. However, you can use AEGP_ExecuteScript() to execute a script that opens a file browser and retrieves the selected file path, then pass that information back to your plugin.

*Tags: `aegp`, `reference`, `scripting`, `ui`*

---

### Is there a sample project demonstrating how to use AEGP_KEYFRAMESUITE3 for keyframe creation?

The "CheesyCheese" sample project is included in the Adobe After Effects SDK and demonstrates how to implement keyframe creation using AEGP_KEYFRAMESUITE3. This example shows practical usage of the keyframe suite for both AEGP and Effects plugins.

*Tags: `aegp`, `keyframe`, `reference`, `sample-code`*

---

### How can an AEGP plugin modify a composition's current time indicator?

Use the AEGP_SetItemCurrentTime() function from the SDK. This function allows you to set the current time for a given composition item from an AEGP plugin, as opposed to effect plugins which use PF_MoveTimeStep().

*Tags: `aegp`, `reference`, `scripting`*

---

### Why does ProjDumper throw an 'unexpected match name searched for in group' error when trying to get a layer's audio stream?

The error occurs when attempting to get an audio stream from a layer that doesn't have audio. Before calling AEGP_GetNewLayerStream with AEGP_LayerStream_AUDIO, you should first check if the layer's source item has audio by calling AEGP_GetLayerSourceItem followed by AEGP_GetItemFlags with AEGP_ItemFlag_HAS_AUDIO. Note that some layers (cameras, lights, text layers, and vector shapes) don't have a source item at all.

```cpp
AEGP_ItemH itemH(NULL);
ERR(suites.LayerSuite8()->AEGP_GetLayerSourceItem(layerH, &itemH));
if (itemH != NULL) {
  A_long flags;
  ERR(suites.ItemSuite9()->AEGP_GetItemFlags(itemH, &flags));
  if (flags & AEGP_ItemFlag_HAS_AUDIO) {
    AEGP_StreamH streamH(NULL);
    ERR(suites.StreamSuite2()->AEGP_GetNewLayerStream(S_my_id, layerH, AEGP_LayerStream_AUDIO, &streamH));
  }
}
```

*Tags: `aegp`, `debugging`, `reference`, `sdk`*

---

### How do I define a mask parameter in an effect plugin?

Use the PathMaster sample project as a reference, which demonstrates how to add a path parameter to your effect. Note that mask selector parameters are limited to masks on the same layer as the effect only. There is no built-in API for a mask selector on a different layer. To access masks on other layers, use the AEGP suites, but be aware that the API function to render a mask will only work on the local effect layer with mask selector parameters. For cross-layer mask selection, you must either implement your own path rasterizing function or use workarounds such as invisible mask selectors pointing to temporary masks.

*Tags: `aegp`, `mask`, `params`, `reference`, `ui`*

---

### What is placement new and what are its uses?

Placement new is a C++ feature that allows you to construct an object in memory that has already been allocated. Instead of `new Type()` which allocates memory and constructs, placement new uses the syntax `new (existing_memory) Type()` to only construct the object in the already-allocated space. This is useful when you need to manage memory allocation separately from object construction. See: http://stackoverflow.com/questions/222557/what-uses-are-there-for-placement-new

```cpp
Bitmap *somePointer = new (someMemoryAlreadyAllocated) Bitmap();
```

*Tags: `memory`, `reference`*

---

### Is there a sample demonstrating OpenGL usage in After Effects plugins?

Yes, Adobe provides the GLator sample as a reference for using OpenGL in After Effects plugins. This sample can help developers understand how to integrate OpenGL drawing capabilities into their plugins.

*Tags: `gpu`, `opengl`, `reference`, `sample`*

---

### How can I retrieve text properties like font, color, and size from a text layer in After Effects?

You can use the JavaScript API through AEGP_ExecuteScript() to access text layer properties. To fetch the font name from a text layer (CS6 and up), use: var textProp = myTextLayer.property("Source Text"); var textDocument = textProp.value; textDocument.font; This approach works because text layers have limited SDK support, so scripting is the recommended method. The script is passed as a char pointer and doesn't require an external script file. For more details on launching AEGP_ExecuteScript() and retrieving results back to the C side, see: http://forums.adobe.com/message/3625857#3625857

```cpp
var textProp = myTextLayer.property("Source Text");
var textDocument = textProp.value;
textDocument.font;
```

*Tags: `aegp`, `params`, `reference`, `scripting`*

---

### What is the recommended approach for accessing text layer data from C/C++ plugins?

Text layers are a blind spot in the After Effects SDK (at least through CS6). The recommended approach is to use AEGP_ExecuteScript() to pull text data using the JavaScript API, as the JavaScript API has better support for text properties than the C SDK. The script can be passed directly as a char pointer without requiring an external script file. See http://forums.adobe.com/message/3625857#3625857 for implementation details on launching AEGP_ExecuteScript() and retrieving results back to the C side.

*Tags: `aegp`, `reference`, `scripting`, `sdk`*

---

### What is a good reference sample for implementing dynamic parameter visibility in After Effects plugins?

The 'Supervisor' sample in the After Effects SDK demonstrates how to dynamically hide and unhide parameters at runtime. It shows the proper use of the UPDATE_PARAMS_UI command and how to leverage AEGP API calls to inspect layer properties and adjust the UI accordingly.

*Tags: `aegp`, `open-source`, `params`, `reference`, `ui`*

---

### How do you properly calculate row bytes and gutters when converting pixel iteration code from 8-bit to 16-bit in After Effects?

When converting from 8-bit to 16-bit rendering, you need to adjust your row byte calculations to account for the different pixel sizes. For 8-bit (PF_Pixel8), divide rowbytes by sizeof(PF_Pixel8) to get the row width in pixels. The same principle applies to 16-bit (PF_Pixel16), but you must use sizeof(PF_Pixel16) instead. The gutter is calculated as: gutter = (rowbytes / pixelSize) - width. Common issues arise from not properly accounting for the pixel size difference between bit depths, which can cause row byte calculations to be incorrect and lead to rendering failures.

```cpp
A_long sizeOfPF_Pixel8 = sizeof(PF_Pixel8);
A_long sizeInputRowBytes = (input_worldP->rowbytes / sizeOfPF_Pixel8);
A_long sizeOutputRowBytes = (output_worldP->rowbytes / sizeOfPF_Pixel8);
in_gutterL = sizeInputRowBytes - input_worldP->width;
out_gutterL = sizeOutputRowBytes - output_worldP->width;
```

*Tags: `memory`, `mfr`, `reference`, `render-loop`*

---

### How do I extract path data from a stream in After Effects using the SDK?

To get path data, first obtain the stream reference for the path. Then use StreamSuite2()->AEGP_GetNewStreamValue() to retrieve the stream value at a specific time, casting it to PF_PathOutlinePtr. Once you have the path outline pointer, use PathDataSuite1() functions like PF_PathPrepareSegLength() and PF_PathGetSegLength() to query segment data. The NULL parameter for PF_ProgPtr can be passed as NULL in this context.

```cpp
suites.StreamSuite2()->AEGP_GetNewStreamValue(NULL, pathStreamH, AEGP_LTimeMode_CompTime, &time, TRUE, &value);
suites.PathDataSuite1()->PF_PathPrepareSegLength(NULL, (PF_PathOutlinePtr)value.val.mask, index, freq, &pathSegment);
suites.PathDataSuite1()->PF_PathGetSegLength(NULL, (PF_PathOutlinePtr)value.val.mask, index, &pathSegment, &segLength);
```

*Tags: `aegp`, `reference`, `sdk`, `sequence-data`*

---

### How can I draw anti-aliased shapes and text directly into an After Effects effect's output buffer?

DrawBot is not suitable for rendering into the output buffer—it only operates on AE's internal interface graphics buffers. Instead, use OS-native drawing tools: on macOS, use Quartz (Core Graphics) APIs like CGBitmapContextCreate(), CGContextShowTextAtPoint(), and CGContextDrawPath(); on Windows, use GDI+ or equivalent APIs. Create an OS graphics context in memory allocated via the Memory Suite, draw your shapes and text to that context, then copy the pixel data from the OS buffer to the effect's output buffer by locking the memory handle and iterating through pixels. See the CCU (Custom Color UI) sample code in the SDK for an example of accessing the output buffer directly in RAM.

```cpp
osBufferBaseAddress = suites.HandleSuite1()->host_lock_handle(osBufferMemHandle);
// Copy pixel data from OS context to output buffer
// Then unlock and free the OS graphics context
```

*Tags: `drawbot`, `macos`, `output-rect`, `reference`, `windows`*

---

### What is the difference between DrawBot and OS drawing libraries for After Effects plugin development?

DrawBot is designed exclusively for drawing custom UI overlays on AE's interface in CS5 and later. It provides built-in drawing tools and operates only on AE's internal interface graphics buffers. OS drawing libraries (Quartz on macOS, GDI+ on Windows) are general-purpose graphics APIs used to render into effect output images. If you need to draw shapes, lines, or text into an effect's output buffer rather than a UI overlay, you must use OS-native drawing tools combined with the Memory Suite to manage the graphics context, not DrawBot.

*Tags: `drawbot`, `macos`, `reference`, `ui`, `windows`*

---

### Is it possible to create a custom output device in After Effects beyond the built-in options?

Yes, it is possible to create a custom output device in After Effects. The recommended approach is to base your plugin on the EMP (External Monitor Preview) sample plugin that is included in the SDK. This sample demonstrates how to implement custom output device functionality.

*Tags: `aegp`, `deployment`, `output-rect`, `reference`*

---

### What sample plugin should be used as a reference for building custom output device plugins?

The EMP (External Monitor Preview) sample plugin is the recommended reference for creating custom output devices in After Effects. This sample has been discussed in the Adobe forums and is included in the SDK documentation. On macOS, developers may also want to examine the Quicktime VOUT sample for additional reference.

*Tags: `aegp`, `macos`, `open-source`, `output-rect`, `reference`*

---

### What AEGP SDK function can be used to apply a preset to a layer from a plugin?

There is no native C API method for applying an effect preset in the AEGP SDK. The recommended approach is to use AEGP_ExecuteScript to execute the Layer.applyPreset() JavaScript method instead.

*Tags: `aegp`, `reference`, `scripting`, `sdk`*

---

### How can I access pixel data from an AEGP plugin?

An AEGP plugin cannot directly access layer pixels with effects, masks, or transformations applied. Instead, you can access the source pixels of a layer by getting the source item and then using AEGP_RenderAndCheckoutFrame() to render and checkout the frame, followed by AEGP_GetReceiptWorld() to retrieve the pixel data. The source provides pixels without any masks, effects, or transformation applied.

*Tags: `aegp`, `layer-checkout`, `pixel-data`, `reference`*

---

### How can I convert Pixel Bender code that samples and transforms pixel coordinates to a native After Effects plugin?

You can accomplish coordinate transformation and pixel sampling using the C++ API. The recommended approach is to pull values from input layers rather than push to arbitrary pixels. Iterate through output samples and pull input values from transformed locations. For direct buffer pixel access in RAM, study the CCU sample project. For iterating through output and pulling from other locations, examine the Shifter sample. Note that subpixel interpolation to 4 adjacent pixels is not supported by the standard suite.

*Tags: `c++`, `effect_api`, `pixel_bender`, `reference`, `sampling`*

---

### What are good sample projects to learn how to pull and push pixel values in After Effects plugins?

The Adobe After Effects SDK includes two key sample projects: the Shifter sample demonstrates how to iterate through output samples and pull input values from arbitrary locations (useful for coordinate transformation effects), and the CCU sample project shows how to access buffer pixels directly in RAM for pushing values to specific pixel locations.

*Tags: `open-source`, `reference`, `sdk`, `skeleton`*

---

### How do you get the position stream for a nested composition layer in an AEGP plugin?

When you nest a composition (compA) into another composition (compB), the nested composition becomes a layer in compB. To get the position stream for that layer, you need to: 1) Use AEGP_GetCompLayerByIndex() on compB to get the AEGP_LayerH for the layer containing the nested composition, or receive it directly when nesting. 2) Then use AEGP_GetNewLayerStream() with AEGP_LayerStream_POSITION on that layer handle to get the position stream. Note that compositions themselves don't have positions—only layers within compositions do. If you're trying to find which comp a nested composition is being used in, you must scan the project and check each layer's source, as there is no direct API to query nesting relationships.

*Tags: `aegp`, `layer-checkout`, `nested_comp`, `reference`*

---

### How do you disable, enable, and hide parameter controls based on a popup selection in an After Effects plugin?

To disable a parameter, use PF_ParamFlag_SUPERVISE on the popup and handle PF_Cmd_USER_CHANGED_PARAM. In the UserChangedParam callback, disable controls by setting ui_flags |= PF_PUI_DISABLED and enable by clearing the flag with &= ~PF_PUI_DISABLED. To hide parameters, use the DynamicStreamSuite2 with AEGP_SetDynamicStreamFlag and AEGP_DynStreamFlag_HIDDEN. First get the effect handle with AEGP_GetNewEffectForEffect, then get the stream with AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag. When hiding topics, hide all parameters within the topic individually; the topic end param does not need to be hidden. Update the UI with PF_UpdateParamUI after making changes.

```cpp
case PF_Cmd_USER_CHANGED_PARAM:
  ERR(UserChangedParam(in_data, out_data, params, output, reinterpret_cast<PF_UserChangedParamExtra *>(extra)));
  break;

// In UserChangedParam:
if (extra->param_index == POPUP_MODE_ID) {
  if (params[POPUP_MODE_ID]->u.cd.value == 2) {
    params[CONTROL_ID]->ui_flags |= PF_PUI_DISABLED;
  } else {
    params[CONTROL_ID]->ui_flags &= ~PF_PUI_DISABLED;
  }
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(
    in_data->effect_ref,
    CONTROL_ID,
    params[CONTROL_ID]
  ));
}

// For hiding with DynamicStreamSuite:
AEGP_SuiteHandler suites(in_data->pica_basicP);
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, SUPER_FLAVOR, &flavor_streamH));
ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(flavor_streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, hide_themB));
```

*Tags: `aegp`, `params`, `reference`, `ui`*

---

### Is there a sample plugin that demonstrates conditional parameter visibility and disabling in After Effects?

The 'Supervisor' sample project included in the After Effects SDK demonstrates how to hide, show, and disable parameters based on user interaction. It shows the proper use of AEGP_GetNewEffectForEffect, AEGP_GetNewEffectStreamByIndex, and AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to control parameter visibility.

*Tags: `aegp`, `open-source`, `params`, `reference`, `ui`*

---

### What are the best approaches for accessing and sampling specific pixels from a buffer in After Effects?

There are two main approaches: (1) Use the Sample Suite to grab color values from a pixel, subpixel, or area within a buffer - see the Shifter sample; (2) Access pixel data directly in RAM and retrieve the data yourself - see the CCU sample. Choose based on whether you need convenience (Sample Suite) or direct memory access performance.

*Tags: `layer-checkout`, `memory`, `reference`, `sdk`*

---

### How can non-layer effect data be persisted and saved with an After Effects project?

Refer to the Adobe forum discussion at http://forums.adobe.com/message/2837630#2837630 which discusses techniques for saving non-layer effect data with the project file. This resource addresses the challenge of maintaining custom data across sessions in AEGP plugins.

*Tags: `aegp`, `arb-data`, `reference`*

---

### How can I determine the bit depth and pixel format of input/output images in an After Effects plugin?

The easiest way to get the bit depth is using `extra->input->bitdepth` in the smart_render call, which returns 8, 16, or 32. Alternatively, use the PF_WorldSuite2 to call `PF_GetPixelFormat()` on the world. After Effects always uses ARGB pixel format. The pixel format enum returns values like PF_PixelFormat_ARGB128 or PF_PixelFormat_ARGB.

```cpp
PF_WorldSuite2 *wsP = NULL;
ERR(suites.Pica()->AcquireSuite(kPFWorldSuite, kPFWorldSuiteVersion2, (const void**)&wsP));
ERR(wsP->PF_GetPixelFormat(inputP, pixelFormat));
ERR(suites.Pica()->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2));
```

*Tags: `mfr`, `params`, `reference`, `sdk`, `smart-render`*

---

### How can I get a Pixel Bender kernel's match name to use in an After Effects native plugin?

Create a new After Effects project with a composition containing a single solid layer. Apply your Pixel Bender plugin to the layer. Then run this ExtendScript to retrieve the match name: alert(app.project.item(1).layer(1).effect(1).matchName); This will return the match name (typically based on the id data in the Pixel Bender script) that you can use in your native plugin to make After Effects treat both as the same plugin for backward compatibility.

```cpp
alert(app.project.item(1).layer(1).effect(1).matchName);
```

*Tags: `params`, `pipl`, `reference`, `scripting`*

---

### How can two AEGPs communicate with each other?

Custom suites are the way to enable inter-AEGP communication. The key is to ensure all AEGPs have loaded before attempting to use the suite, since AE loads AEGPs before effects. The first idle call is a good time to load and use the suite. Refer to the 'sweetie' sample in the SDK to see how to create a custom suite, and the 'checkout' sample (globalSetup() function) to see how to use a custom suite.

*Tags: `aegp`, `custom-suite`, `reference`*

---

### What are good SDK samples to learn about custom suites in After Effects?

The 'sweetie' sample demonstrates how to create a custom suite for inter-AEGP communication. The 'checkout' sample shows how to use a custom suite, particularly in the globalSetup() function. These samples are included in the After Effects SDK and serve as the primary documentation for custom suite implementation.

*Tags: `aegp`, `open-source`, `reference`, `sdk`*

---

### How can I implement a custom pixel iteration function similar to PF_Iterate8Suite1::iterate?

To implement a custom iteration function, start by examining the 'shifter' sample which implements one of the iteration functions. For direct pixel data access without the iteration suite, consult the 'CCU' sample, particularly its render function. You can also use the iterate_generic() function to obtain threading services without additional functionality. For optimal performance, thread rows rather than individual pixels to reduce CPU overhead per pixel.

*Tags: `mfr`, `pixel-iteration`, `reference`, `render-loop`, `threading`*

---

### What sample plugins demonstrate pixel iteration and direct pixel data access in After Effects?

The 'shifter' and 'CCU' samples are recommended references. The 'shifter' sample implements iteration functions, while the 'CCU' sample demonstrates how to access pixel data directly without using the iteration suite, particularly in its render function implementation.

*Tags: `mfr`, `open-source`, `reference`, `render-loop`*

---

### Is there an API to draw standard After Effects controls like buttons and checkboxes outside the Effect Controls Window?

No, there is no API available to draw standard AE controls outside the ECW (Effect Controls Window). Effect controls can only be created during param_setup in effects and will display only in the ECW and timeline. Custom controls would need to be reimplemented with custom drawing code.

*Tags: `aegp`, `params`, `reference`, `ui`*

---

### How can I detect whether a layer is audio-only in After Effects?

There are multiple approaches to detect audio-only layers: (1) Use AEGP_GetLayerFlags() to check AEGP_LayerFlag_VIDEO_ACTIVE and AEGP_LayerFlag_AUDIO_ACTIVE flags, though this only tells if video is off, not if it's a true audio layer. (2) Use AEGP_GetLayerSourceItem() to get the source item, then call AEGP_GetItemFlags() and check AEGP_ItemFlag_HAS_VIDEO and AEGP_ItemFlag_HAS_AUDIO—if the source has audio but no video, it's definitely audio-only. (3) As a last resort, use AEGP_ExecuteScript() with JavaScript for rare checks. Additionally, audio layers typically have dimensions of 0x0 in the project panel, which can serve as another detection criterion.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

### What is a guide for setting up Visual C++ Express to compile 64-bit applications?

A patch and setup guide for VC2008 Express 64-bit compilation is available at http://www.cppblog.com/xcpp/archive/2009/09/09/vc2008express_64bit_win7sdk.html, which explains how to configure Visual C++ Express with the Windows SDK to enable 64-bit plugin development.

*Tags: `build`, `reference`, `tool`, `windows`*

---

### What is an alternative to Interface Builder for creating simple dialogs on macOS in After Effects plugins?

Use the CFUserNotification API, which is C-based and simpler than Interface Builder and Objective-C. It's suitable for basic dialogs with string input and 1-2 text fields. Refer to Apple's CFUserNotification documentation at http://developer.apple.com/mac/library/documentation/CoreFoundation/Reference/CFUserNotificationRef/Reference/reference.html

*Tags: `macos`, `reference`, `ui`*

---

### What is the AEGP artisan sample plugin and how does it relate to composition rendering?

The 'arti' sample is an AEGP (After Effects General Plugin) of type 'artisan' that demonstrates how to write a custom composition renderer. Artisan plugins replace After Effects' default renderer (like 'advanced3D') and have direct access to intermediate composition rendering results. This makes them capable of accessing composited buffers at different rendering stages, unlike standard effect plugins which only see the final result. This approach is recommended for advanced scenarios requiring intermediate buffer access, though it is noted as being very difficult to implement correctly.

*Tags: `aegp`, `open-source`, `reference`, `render-loop`*

---

### How do you retrieve shape layer content parameters (like Polystar, ZigZag, Repeater) in AEGP?

Shape layer contents are not accessed the same way as effects. For path data, use PF_PathDataSuite1 to retrieve path vertices. For rendered pixels, you cannot directly read the source since shape layers don't have one. As an alternative, you can get the shape and render it yourself, or duplicate the composition, delete everything except the shape layer, and use AEGP_GetReceiptWorld() to render that new composition. The specific method was discussed in detail in an Adobe forum thread.

*Tags: `aegp`, `params`, `reference`, `shape-layer`*

---

### How can I create an After Effects plugin that samples the current frame and displays information graphically in a separate window?

You should build an AEGP (After Effects General Plugin) rather than an effect plugin. Use the 'panelator' sample as a base to create a dockable panel for displaying your graphical output. To access frame data, you have two approaches: (1) Use the 'EMP' (external monitor preview) sample which provides the MyBlit() function to receive the currently viewed composition buffer, registered during entryPoint(). (2) Use AEGP functions to render project items on demand: AEGP_GetMostRecentlyUsedComp, AEGP_RenderAndCheckoutFrame, and AEGP_GetReceiptWorld. The second approach can leverage AE's render cache to avoid re-rendering if the frame has already been processed.

*Tags: `aegp`, `output-rect`, `reference`, `ui`*

---

### What is the panelator sample plugin used for?

The 'panelator' sample is an AEGP that demonstrates how to create a dockable panel in After Effects, similar to built-in palettes like the Info palette. It serves as a template for AEGP plugins that need to display custom user interfaces and allows you to draw arbitrary content within the panel.

*Tags: `aegp`, `reference`, `sample`, `ui`*

---

### What is the EMP sample and how does it deliver frame data to plugins?

EMP (External Monitor Preview) is a sample plugin template that demonstrates how to receive image data from After Effects. It uses the MyBlit() function as the callback where AE delivers the currently viewed composition buffer to the plugin. This function is registered during the entryPoint() function and is useful for plugins that need real-time access to the composition being viewed.

*Tags: `aegp`, `output-rect`, `reference`, `sample`*

---

### How do you use PF_AppGetColor() to retrieve After Effects application colors in a custom effect?

Use the AppSuite3() to call PF_AppGetColor() with the desired color type and a pointer to store the result. The syntax is: suites.AppSuite3()->PF_AppGetColor(PF_App_Color_HOT_TEXT, &appColor1); where PF_App_Color_HOT_TEXT is the color type you want to retrieve and appColor1 is the variable that will store the retrieved color.

```cpp
suites.AppSuite3()->PF_AppGetColor(PF_App_Color_HOT_TEXT, &appColor1);
```

*Tags: `aegp`, `params`, `reference`, `ui`*

---

### How do you properly implement inter-effect communication using AEGP_EffectCallGeneric in After Effects plugins?

When calling an effect from an AEGP (like Sweetie calling Checkout), you must handle timing carefully. The key issue is that you cannot call an effect back immediately while it's still processing its original call to your AEGP. Instead, use idle_hook: when the effect calls your AEGP, set a flag and wait for the next idle_hook call, then initiate AEGP_EffectCallGeneric when the effect is guaranteed to be idle. Alternatively, respond immediately via the same custom suite that did the calling, passing data for the calling effect to process without needing effectCallGeneric. Do not use AEGP_GetNewEffectForEffect with a passed effect_ref—instead, use AEGP_GetLayerEffectByIndex to look up the effect on the target layer. For permanent effect tracking across the project lifetime, store the comp itemID and layer layerID rather than relying on EffectRefH which can become invalid.

```cpp
// From AEGP: wait for idle hook before calling effect back
if (AEFX_AcquireSuite(in_data, out_data, kDuckSuite1, kDuckSuiteVersion1, "Couldn't load suite.", (void**)&dsP)) {
  PF_STRCPY(out_data->return_msg, "No Duck Suite!");
} else {
  if (dsP) {
    dsP->Quack(2);
  }
  AEFX_ReleaseSuite(in_data, out_data, kDuckSuite1, kDuckSuiteVersion1, "Couldn't release suite.");
}

// From Effect: pass in_data to AEGP via custom suite for callback
static SPAPI A_Err CalledFromEffect(void* ptr) {
  PF_InData* pIndata = static_cast<PF_InData*>(ptr);
  // Use AEGP_GetLayerEffectByIndex instead of AEGP_GetNewEffectForEffect
}
```

*Tags: `aegp`, `effect-ref`, `idle-hook`, `inter-plugin`, `reference`, `timing`*

---

### What are the reference projects to study for implementing custom AEGP suites and inter-plugin communication?

The Sweetie and Checkout sample projects in the After Effects SDK demonstrate custom suite implementation for effect-to-AEGP communication. ProjectDumper and Shifter are also referenced as examples of inter-plugin communication patterns. These samples show how to use custom suites (like the Duck Suite example) for effects to safely communicate with AEGPs and other plugins.

*Tags: `aegp`, `open-source`, `reference`, `sample-code`, `sdk`*

---

### What is the best way to create interactive UI elements like draggable masks in After Effects plugins?

Use a custom UI overlay rather than rendering the mask as part of the image. Rendering the mask in the image has several drawbacks: it gets affected by subsequent effects and display channels, is confined to the layer's size, and forces re-renders when toggling visibility. A custom UI is necessary for interactivity since After Effects won't report click events otherwise. The CCU (Custom Composition UI) sample in the After Effects SDK provides an excellent starting point for creating interactive interfaces in the composition window.

*Tags: `custom-ui`, `interactive`, `reference`, `sdk`, `ui`*

---

### Where can I find example code for building custom interactive UI in After Effects composition windows?

The After Effects SDK includes a CCU (Custom Composition UI) sample that demonstrates how to create a simple interactive interface directly in the composition window. This sample is the recommended starting point for developers building custom UIs with interactive elements like draggable shapes and masks.

*Tags: `custom-ui`, `open-source`, `reference`, `sdk`, `ui`*

---

### How do you properly use the transform_world() function to scale and rotate a PF_LayerDef?

To use transform_world() effectively: (1) Pass NULL for mask_world0 to use the whole frame. (2) You can pass one or two matrices—two matrices enable motion blur between transformations. Use syntax (&matrix1, &matrix2) for two matrices or &matrix1 for one, and specify the matrix count. (3) Note that AE's 3x3 matrices have swapped x and y compared to standard math notation—what should be matrix[1][2] goes in matrix[2][1]. See the CCU sample for identity and transformation matrix functions. (4) Always pass a full-frame rectangle initially; transform_world() does not accept NULL for the destination rectangle. The offset should be embedded in the matrix itself, not assumed from the rectangle's top-left corner.

*Tags: `matrix`, `mfr`, `params`, `reference`, `transform`*

---

### Where can I find sample code for creating and manipulating transformation matrices in After Effects plugins?

The CCU sample included in the After Effects SDK contains helper functions for creating identity matrices and concatenating them with scale and rotation transformations. This sample demonstrates proper matrix manipulation for use with transform_world() and similar functions.

*Tags: `matrix`, `mfr`, `reference`, `sample`, `sdk`*

---
