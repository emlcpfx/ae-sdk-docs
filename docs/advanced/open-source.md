# Open Source

> 61 Q&As · source: AE plugin dev community Discord

### Where should After Effects SDK developers share and discover code snippets and solutions?

Stack Overflow with the 'After Effects SDK' tag is recommended as a good system for searching and hosting code snippets. GitHub can also be used for sharing code examples, though the AESDK community is niche. Creating dedicated channels or communities helps organize and grow the knowledge base.

*Tags: `deployment`, `open-source`, `reference`, `scripting`, `tool`*

---

### Why is Lua a better choice than Python for an After Effects script-driven drawing plugin?

Lua is significantly easier to compile into a plugin compared to Python. With Python, the developer had to request permission to install libraries across the system, which is intrusive. Lua, on the other hand, compiles cleanly into the plugin without requiring system-wide library installations. Both languages require similar effort for C bindings without automation or code generation tools, but Lua's compilation model is much cleaner for plugin distribution.

*Tags: `deployment`, `open-source`, `scripting`*

---

### Where can I find an example After Effects plugin implementation?

dvb metareal shared their After Effects plugin at https://omino.com/pixelblog, currently at version 0.9. This serves as a working reference implementation for plugin development.

*Tags: `open-source`, `reference`, `tool`*

---

### Is there a reference implementation for handling BGRA and YUVA colorspaces in Premiere plugins?

Yes, the SDK noise example in the After Effects SDK demonstrates proper handling of both BGRA and YUVA colorspaces for Premiere. This example shows how to use the special BGRA suite for 8-bit processing and serves as a reference for implementing colorspace support in plugins targeting both AE and Premiere.

*Tags: `bgra`, `open-source`, `premiere`, `reference`, `sdk-example`, `yuva`*

---

### What is Bolt UXP and how does it relate to Premiere Pro plugin development?

Bolt UXP is a resource mentioned in the discussion as relevant to Premiere Pro UXP development. Justin shared this as a reference for developers looking to build UXP-based plugins.

*Tags: `open-source`, `premiere`, `tool`, `uxp`*

---

### What is the Transfusion plugin and how does it demonstrate sequence data caching?

Transfusion is an After Effects plugin by gabgren that demonstrates caching the EffectWorld in sequence data across frames. The plugin stores the result of each render call in sequence_data, then uses that cached result on the next frame for additive effects. This approach works for 8-bit rendering without MFR (Multi-Frame Rendering), though MFR compatibility may require adjustments to this workflow.

*Tags: `caching`, `open-source`, `reference`, `sequence-data`*

---

### Is there an open-source example of volumetric fractal noise implementation for After Effects?

Yes, there is a GitHub project by mes51 that implements volumetric fractal noise: https://github.com/mes51/VolumetricFractalNoise. This project serves as a reference implementation and workaround for volumetric fractal noise effects in After Effects plugins.

*Tags: `gpu`, `open-source`, `reference`, `rendering`*

---

### Is there an open-source example of a Vulkan-based After Effects plugin?

Yes, Wunkolo has open-sourced Vulkanator, a Vulkan-based After Effects plugin project with sample code. This project represents an early push toward more open-source resources in the After Effects plugin development space. Repository: https://github.com/Wunkolo/Vulkanator

*Tags: `gpu`, `open-source`, `reference`, `vulkan`*

---

### Where can I find an example solution for color grid implementation in After Effects plugins?

The colorgrid example in the After Effects SDK contains the solution. This is a reference implementation that demonstrates how to work with color grids in plugin development.

*Tags: `open-source`, `reference`, `sdk`*

---

### What is an example of an After Effects plugin using Vulkan?

ScaleUp is an AE script/plugin that uses Vulkan and MoltenVK for rendering. It can be found at https://aescripts.com/scaleup/

*Tags: `gpu`, `moltenvk`, `open-source`, `reference`, `vulkan`*

---

### Is there an example of integrating Vulkan with After Effects on Apple Silicon?

Wunk is updating the Vulkanator sample project to demonstrate VulkanAPI (via MoltenVK) integration with After Effects on Apple Silicon, showing how to leverage Vulkan for GPU-accelerated rendering on macOS ARM64 systems.

*Tags: `apple-silicon`, `gpu`, `metal`, `open-source`, `vulkan`*

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

### What logging library is recommended for After Effects plugins with file rotation capabilities?

spdlog is recommended for plugin logging. It provides file rotation functionality where you can define the maximum size of individual log files and how many log files to retain, making it suitable for long-running plugin operations.

*Tags: `debugging`, `logging`, `open-source`, `tool`*

---

### What is a good approach to manage memory safety for After Effects SDK objects?

Create C++ wrapper classes around AESDK objects to enable automatic memory management. This approach leverages C++ features like constructors and destructors to ensure proper allocation and deallocation of SDK objects. Additionally, there are already existing C++ wrappers for portions of the After Effects SDK available on GitHub that can be referenced or reused for this purpose.

*Tags: `aegp`, `build`, `memory`, `open-source`, `reference`*

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

### Is there a reference implementation of dynamic dropdown list handling in After Effects plugins?

Diffusae (an AE plugin project) has a working implementation for updating dropdown parameters like a models list. It uses the AEGP StreamSuite approach with AEGP_GetNewEffectStreamByIndex and AEGP_SetStreamValue called from UpdateParamsUI. This approach is more reliable than directly manipulating param_union.pd.u.namesptr.

*Tags: `aegp`, `open-source`, `params`, `reference`, `ui`*

---

### Is there an open-source Rust binding for After Effects plugin development?

The virtualritz/after-effects repository on GitHub provides Rust bindings for After Effects plugin development. This project enables developers to write AE plugins using Rust instead of C++.

*Tags: `build`, `cross-platform`, `open-source`, `reference`, `rust`*

---

### How do you debug Rust bindings for After Effects on macOS with symbols?

Use VSCode with the CodeLLDB extension. Configure a launch.json with the lldb debugger type pointing to the After Effects application bundle, and set up a preLaunchTask to build the plugin before launching. The configuration should specify the full path to the After Effects app (e.g., /Applications/Adobe After Effects 2024/Adobe After Effects 2024.app).

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build"
    }
  ]
}
```

*Tags: `debugging`, `macos`, `open-source`, `rust`*

---

### Is there an open-source Rust binding library for After Effects plugin development?

The virtualritz/after-effects repository on GitHub provides Rust bindings for After Effects plugin development. It enables developers to write AE plugins in Rust with support for debugging on macOS using standard debuggers.

*Tags: `macos`, `open-source`, `reference`, `rust`*

---

### What is a good resource for understanding scope guards and their implementation in C++?

Alex Bizeau from maxon mentioned that scope guards are smart pointers for anything that can use lambda functions. They're valuable memory safety tools because they handle deletion automatically on throwing and return statements without requiring explicit deletion handling at every return point. Maxon has their own implementation, with examples available in their codebase.

*Tags: `debugging`, `memory`, `open-source`, `reference`, `tool`*

---

### What UI approach do some AE plugins use to improve interactivity with parameters?

Some plugins like Gifgun and Datamosh use an external application window (often built with C++ and ImGui) that allows users to adjust parameters interactively in a fullscreen UI, then pre-render and send results back to After Effects. This avoids the need for re-rendering on every parameter change within the comp, which would be required if using a custom UI directly in After Effects like Optical Flares does.

*Tags: `open-source`, `sequence-data`, `ui`, `workflow`*

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

### Is there an open-source skeleton project for Vulkan SDK integration with After Effects plugins?

Eric CPFX shared a Skeleton project designed for using the Vulkan SDK with After Effects plugin development. This project serves as a reference implementation for developers looking to integrate Vulkan into their AE plugins.

*Tags: `build`, `open-source`, `reference`, `skeleton`, `vulkan`*

---

### Where should After Effects plugin developers share and host code snippets?

Stack Overflow is recommended as a good platform for sharing code snippets and knowledge about After Effects SDK development. Using the tag 'After Effects SDK' makes the content searchable and discoverable. While GitHub could be used, Stack Overflow's forum structure is better suited for this niche community compared to chat systems which have limitations for code hosting.

*Tags: `deployment`, `open-source`, `reference`, `scripting`, `tool`*

---

### Is there an open-source After Effects plugin example available?

dvb metareal shared their After Effects plugin at https://omino.com/pixelblog, currently at version 0.9. This serves as a reference implementation for AE plugin development.

*Tags: `deployment`, `open-source`, `reference`*

---

### Where can you find example code for handling Premiere Pro BGRA and YUVA colorspaces?

The sdknoise example in the After Effects SDK demonstrates how to properly handle BGRA and YUVA colorspaces for Premiere Pro plugins. This example shows the correct approach for colorspace handling without relying on problematic iterate suites.

*Tags: `bgra`, `colorspace`, `open-source`, `premiere`, `reference`, `yuva`*

---

### Is there a working sample demonstrating buffer expansion in SmartFX?

Yes, shachar carmi created a working sample based on the 'smarty pants' sample that demonstrates buffer resizing in SmartFX. The sample is available at https://drive.google.com/file/d/1w4Fc9rtGXjU-YT4CC-6q01ft2MOvOLtt/view?usp=sharing. Additionally, an updated version that implements Gaussian blur with buffer expansion as the blur size grows is available at https://drive.google.com/file/d/1JWtvacj-jt5I8XfqxCzz3k_8lGi_nLaI/view?usp=sharing. These samples show how to properly structure the PreRender and SmartRender functions to handle expanded buffers correctly.

*Tags: `open-source`, `reference`, `smart-render`, `smartfx`*

---

### Where can you find an open-source directional blur plugin with buffer expansion implementation?

The AE-Motion repository on GitHub contains a Directional Blur plugin modified to use buffer expansion: https://github.com/NotYetEasy/AE-Motion/tree/main/DirectionalBlur. The repository includes all the plugin source code and can be compiled to test buffer expansion functionality in a real blur plugin implementation.

*Tags: `github`, `open-source`, `reference`, `smartfx`*

---

### Is there an example of passing strings from ExtendScript to a custom After Effects plugin via ExternalObject?

Yes, there is a working example provided by the community that demonstrates C external object communication with ExtendScript. The example shows how to structure a C external object for passing strings between ExtendScript and an After Effects plugin. Link: https://community.adobe.com/t5/after-effects-discussions/issue-passing-string-from-extendscript-to-custom-after-effects-plugin-via-externalobject/m-p/15019735#M259100

*Tags: `aegp`, `open-source`, `reference`, `scripting`*

---

### What is a good SDK sample project that demonstrates handling mouse clicks in the composition window?

The CCU SDK sample project is an official Adobe After Effects SDK example that shows how to get mouse click locations in comp window coordinates and convert them into layer source coordinates. It provides a complete working implementation for custom UI event handling.

*Tags: `open-source`, `reference`, `sample-project`, `sdk`, `ui`*

---

### What is the Supervisor SDK sample project and what does it demonstrate?

The Supervisor SDK sample project is an official Adobe After Effects SDK example that demonstrates proper parameter supervision and UI updating. While somewhat convoluted, the first few lines of its UpdateParameterUI() function show the correct implementation for using PF_UpdateParamUI with parameter copies and proper flag handling.

*Tags: `open-source`, `params`, `reference`, `sdk`, `ui`*

---

### Is there an official sample project demonstrating how to access and manipulate streams in the After Effects SDK?

Yes, Adobe provides the 'project dumper' sample project as part of the After Effects SDK. This sample demonstrates how to access streams and is useful for understanding how to affect parameters in a project using the StreamSuite.

*Tags: `aegp`, `open-source`, `reference`, `sdk`, `streaming`*

---

### What is a reliable library for JSON handling in ExtendScript?

The JSON-js library by Douglas Crockford (https://github.com/douglascrockford/JSON-js) provides a free, standard JSON implementation for ExtendScript. It allows you to use JSON.stringify() and other JSON methods consistently across different systems (macOS and Windows).

*Tags: `extendscript`, `json`, `library`, `open-source`*

---

### What is a good library for saving EffectWorld pixel data to image files in AE plugins?

tinypng is a popular library for saving pixel buffer data to image files when working with AE plugin EffectWorld objects. You can access the pixel data via effect_worldP->data and then use tinypng to serialize it to disk.

*Tags: `memory`, `open-source`, `reference`, `tool`*

---

### Is there an open-source searchable database of After Effects command IDs?

Justin Taylor at Hyper Brew maintains an updated searchable Command ID list for After Effects 2022 at https://hyperbrew.co/blog/after-effects-command-ids/, which is useful for scripting and tool development. There was also a Bitbucket snippet repository at https://bitbucket.org/justin2taylor/workspace/snippets/aLjjBE with command ID resources.

*Tags: `extendscript`, `open-source`, `reference`, `scripting`, `tool`*

---

### What is a good resource for understanding transformation matrices for use in After Effects plugins?

Wikipedia's article on Transformation Matrices (https://en.wikipedia.org/wiki/Transformation_matrix) is a good starting point for understanding how 3x3 matrices work in graphics. Be aware that After Effects uses row-based matrices, which differs from the OpenGL standard, so axis values need to be swapped accordingly. Standard transformation matrix tutorials and code samples from general graphics programming resources apply directly to AE plugin development.

*Tags: `open-source`, `reference`, `transformation`*

---

### Is there a sample project that demonstrates how to extract parameter data from After Effects projects?

Yes, Adobe provides the 'project dumper' sample project as a reference implementation for getting parameter data from plugins using both the C and JavaScript APIs.

*Tags: `open-source`, `params`, `reference`, `scripting`*

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

### What sample projects and tutorials are available for learning After Effects plugin development?

There is a significant shortage of sample projects and tutorials for AE plugin development. Available resources include: (1) A very old MacTech tutorial on After Effects Plugins at http://www.mactech.com/articles/mactech/Vol.15/15.09/AfterEffectsPlugins/index.html, and (2) A paid course at https://www.fxphd.com/details/526/. Beyond these, the Adobe community forums contain extensive information ranging from basics to advanced topics. The SDK itself includes sample plugin codes such as Transformer and Skeleton that can be studied.

*Tags: `open-source`, `reference`, `sdk`, `tutorial`*

---

### Is there a sample plugin that demonstrates PreRender and SmartRender implementation?

The Shifter sample plugin included in the After Effects SDK demonstrates PreRender and SmartRender usage.

*Tags: `open-source`, `reference`, `sample`, `smart-render`*

---

### What are good sample projects to study for implementing stream data in After Effects plugins?

The ProjDumper, Streamie, and Mangler sample projects included in the After Effects SDK are recommended as good places to see streams implemented. These projects are included in the SDK download packages available from Adobe's developer portal.

*Tags: `aegp`, `open-source`, `reference`, `sample`*

---

### What is a good reference sample for implementing dynamic parameter visibility in After Effects plugins?

The 'Supervisor' sample in the After Effects SDK demonstrates how to dynamically hide and unhide parameters at runtime. It shows the proper use of the UPDATE_PARAMS_UI command and how to leverage AEGP API calls to inspect layer properties and adjust the UI accordingly.

*Tags: `aegp`, `open-source`, `params`, `reference`, `ui`*

---

### What sample plugin should be used as a reference for building custom output device plugins?

The EMP (External Monitor Preview) sample plugin is the recommended reference for creating custom output devices in After Effects. This sample has been discussed in the Adobe forums and is included in the SDK documentation. On macOS, developers may also want to examine the Quicktime VOUT sample for additional reference.

*Tags: `aegp`, `macos`, `open-source`, `output-rect`, `reference`*

---

### What are good sample projects to learn how to pull and push pixel values in After Effects plugins?

The Adobe After Effects SDK includes two key sample projects: the Shifter sample demonstrates how to iterate through output samples and pull input values from arbitrary locations (useful for coordinate transformation effects), and the CCU sample project shows how to access buffer pixels directly in RAM for pushing values to specific pixel locations.

*Tags: `open-source`, `reference`, `sdk`, `skeleton`*

---

### Is there a sample plugin that demonstrates conditional parameter visibility and disabling in After Effects?

The 'Supervisor' sample project included in the After Effects SDK demonstrates how to hide, show, and disable parameters based on user interaction. It shows the proper use of AEGP_GetNewEffectForEffect, AEGP_GetNewEffectStreamByIndex, and AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to control parameter visibility.

*Tags: `aegp`, `open-source`, `params`, `reference`, `ui`*

---

### What are good SDK samples to learn about custom suites in After Effects?

The 'sweetie' sample demonstrates how to create a custom suite for inter-AEGP communication. The 'checkout' sample shows how to use a custom suite, particularly in the globalSetup() function. These samples are included in the After Effects SDK and serve as the primary documentation for custom suite implementation.

*Tags: `aegp`, `open-source`, `reference`, `sdk`*

---

### What sample plugins demonstrate pixel iteration and direct pixel data access in After Effects?

The 'shifter' and 'CCU' samples are recommended references. The 'shifter' sample implements iteration functions, while the 'CCU' sample demonstrates how to access pixel data directly without using the iteration suite, particularly in its render function implementation.

*Tags: `mfr`, `open-source`, `reference`, `render-loop`*

---

### What is the AEGP artisan sample plugin and how does it relate to composition rendering?

The 'arti' sample is an AEGP (After Effects General Plugin) of type 'artisan' that demonstrates how to write a custom composition renderer. Artisan plugins replace After Effects' default renderer (like 'advanced3D') and have direct access to intermediate composition rendering results. This makes them capable of accessing composited buffers at different rendering stages, unlike standard effect plugins which only see the final result. This approach is recommended for advanced scenarios requiring intermediate buffer access, though it is noted as being very difficult to implement correctly.

*Tags: `aegp`, `open-source`, `reference`, `render-loop`*

---

### What are the reference projects to study for implementing custom AEGP suites and inter-plugin communication?

The Sweetie and Checkout sample projects in the After Effects SDK demonstrate custom suite implementation for effect-to-AEGP communication. ProjectDumper and Shifter are also referenced as examples of inter-plugin communication patterns. These samples show how to use custom suites (like the Duck Suite example) for effects to safely communicate with AEGPs and other plugins.

*Tags: `aegp`, `open-source`, `reference`, `sample-code`, `sdk`*

---

### Where can I find example code for building custom interactive UI in After Effects composition windows?

The After Effects SDK includes a CCU (Custom Composition UI) sample that demonstrates how to create a simple interactive interface directly in the composition window. This sample is the recommended starting point for developers building custom UIs with interactive elements like draggable shapes and masks.

*Tags: `custom-ui`, `open-source`, `reference`, `sdk`, `ui`*

---
