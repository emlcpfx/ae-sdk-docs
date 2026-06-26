# Tool

> 29 Q&As · source: AE plugin dev community Discord

### Where should After Effects SDK developers share and discover code snippets and solutions?

Stack Overflow with the 'After Effects SDK' tag is recommended as a good system for searching and hosting code snippets. GitHub can also be used for sharing code examples, though the AESDK community is niche. Creating dedicated channels or communities helps organize and grow the knowledge base.

*Tags: `deployment`, `open-source`, `reference`, `scripting`, `tool`*

---

### Where can I find an example After Effects plugin implementation?

dvb metareal shared their After Effects plugin at https://omino.com/pixelblog, currently at version 0.9. This serves as a working reference implementation for plugin development.

*Tags: `open-source`, `reference`, `tool`*

---

### What is Bolt UXP and how does it relate to Premiere Pro plugin development?

Bolt UXP is a resource mentioned in the discussion as relevant to Premiere Pro UXP development. Justin shared this as a reference for developers looking to build UXP-based plugins.

*Tags: `open-source`, `premiere`, `tool`, `uxp`*

---

### What is a good resource for understanding the Jump Flooding Algorithm for distance field generation?

The blog post at https://itscai.us/blog/post/jfa/ provides an intuitive description of the Jump Flooding Algorithm and demonstrates how it applies directly to creating outlines and choker effects from alpha channels.

*Tags: `algorithm`, `distance-field`, `reference`, `tool`*

---

### What drawing libraries or tools are recommended for After Effects plugin development?

Cairo and DrawBot are recommended options for drawing in After Effects plugins. The canvas suite should not be confused with a drawing toolkit—it actually refers to the rendering pipeline rather than a suite for drawing on the viewport.

*Tags: `reference`, `tool`, `ui`*

---

### What is the Adobe ae-plugin-thread-safety repository?

Adobe's ae-plugin-thread-safety repository at https://github.com/adobe/ae-plugin-thread-safety contains examples and best practices for handling thread-safe operations in After Effects plugins, including atomic variable usage and mutex patterns.

*Tags: `open-source`, `reference`, `threading`, `tool`*

---

### What logging library is recommended for After Effects plugins with file rotation capabilities?

spdlog is recommended for plugin logging. It provides file rotation functionality where you can define the maximum size of individual log files and how many log files to retain, making it suitable for long-running plugin operations.

*Tags: `debugging`, `logging`, `open-source`, `tool`*

---

### Is Media Encoder compatible with AEGP plugins for rendering After Effects projects?

AEGP plugins are not supported by Adobe Media Encoder (AME). If your plugin requires AEGP functionality to render, you will need to convert the AEGP plugin into a standalone or command-line tool to use with Media Encoder.

*Tags: `aegp`, `deployment`, `premiere`, `tool`*

---

### What is an easy way to calculate and verify the correct bitmask values for PiPL outflags and outflags2 fields?

Tobias Fleischer created a quick online lookup/cheat sheet tool in JavaScript that allows developers to easily find and verify the correct bitmask values for PiPL outflags and outflags2 fields without needing to run the host application and receive warnings. The tool is available at: https://www.reduxfx.com/piplflags/

*Tags: `build`, `debugging`, `pipl`, `reference`, `tool`*

---

### What is a tool for calculating PiPL outflags and outflags2 bitmasks?

Tobias Fleischer (reduxFX) created an online lookup/cheat sheet for calculating and evaluating bitmasks for the outflags and outflags2 fields in PiPL files. Available at https://reduxfx.com/aeoutflags.htm

*Tags: `deployment`, `pipl`, `reference`, `tool`*

---

### Are there tools available that can remove XMP packets from After Effects project files to reduce file size?

Yes, tools like AE Viewer are available that can remove the XMP packet from After Effects files, significantly reducing file sizes (e.g., from 50mb down to 50kb). This demonstrates that AE bloats project files with substantial XMP metadata that can be stripped if not needed.

*Tags: `deployment`, `reference`, `tool`*

---

### Is there a reference repository documenting After Effects SDK structures and flags?

The virtualritz/after-effects repository on GitHub contains useful references for understanding PF_LayerDef structures and world_flags behavior, including documented examples of how to extract bit depth information from layer definitions.

*Tags: `mfr`, `open-source`, `reference`, `tool`*

---

### Is there an open-source reference for After Effects plugin development in Rust?

Yes, the virtualritz/after-effects repository (https://github.com/virtualritz/after-effects) provides Rust bindings and examples for After Effects plugin development. It includes practical code examples and API patterns learned from real plugin development, including handling 3D rendering integration and bit-depth detection. The crate was published as a result of production plugin development experience.

*Tags: `build`, `open-source`, `reference`, `tool`*

---

### What is Maxon Studio and how does it relate to project templates?

Maxon Studio is a template tool that transforms compositions into capsules which can be reused as templates to regenerate projects. It is similar to the AEGP ProjDumper sample but with significantly more features. The tool has been internal since inception, with initial release delayed due to lacking features and UX. It is being prepared for wider studio release with plans to capture arbitrary plugin data into capsules for reuse across 3rd party and Adobe stock plugins.

*Tags: `aegp`, `arb-data`, `reference`, `tool`*

---

### What is a good resource for understanding scope guards and their implementation in C++?

Alex Bizeau from maxon mentioned that scope guards are smart pointers for anything that can use lambda functions. They're valuable memory safety tools because they handle deletion automatically on throwing and return statements without requiring explicit deletion handling at every return point. Maxon has their own implementation, with examples available in their codebase.

*Tags: `debugging`, `memory`, `open-source`, `reference`, `tool`*

---

### What is a good tool for managing resource cleanup and memory safety in C++ plugin code?

Scope guards are a useful pattern for memory safety in C++ code. They act like smart pointers for any resource, allowing you to use lambda functions to handle cleanup on scope exit. This is particularly valuable for handling exceptions and early returns without explicitly managing deletion at every exit point. Alex Bizeau from maxon recommended the scope_guard library as a reference implementation: https://github.com/ricab/scope_guard/blob/main/scope_guard.hpp. Scope guards work well with both smart pointers and even malloc/free patterns, making them a favorite tool for resource management.

*Tags: `cpp`, `memory`, `reference`, `resource-management`, `smart-pointer`, `tool`*

---

### Are there known issues with ZXPSignCmd and plugin signing on Windows?

Yes, there have been reports of ZXPSignCmd segmentation faults and timestamp-related signing failures on Windows. These issues appear to be affecting plugin developers globally, and Adobe was working on releasing a patched version of the ZXPSignCmd tool to resolve them.

*Tags: `deployment`, `tool`, `windows`*

---

### What is a command ID tool for After Effects plugin development?

Justin shared a command ID tool that helps developers work with After Effects commands. The tool was highlighted in a recent LinkedIn post by Justin and is useful for plugin development workflows.

*Tags: `aegp`, `debugging`, `reference`, `tool`*

---

### Is there a recommended WebGPU library for building After Effects GPU plugins?

The WGPU library is commonly used for WebGPU support in AE plugins due to its good cross-platform compatibility (automatically using Metal on Mac and Vulkan on PC). However, Google's Dawn is noted as having better performance characteristics, though it is more difficult to build and integrate.

*Tags: `cross-platform`, `gpu`, `tool`, `webgpu`*

---

### Where should After Effects plugin developers share and host code snippets?

Stack Overflow is recommended as a good platform for sharing code snippets and knowledge about After Effects SDK development. Using the tag 'After Effects SDK' makes the content searchable and discoverable. While GitHub could be used, Stack Overflow's forum structure is better suited for this niche community compared to chat systems which have limitations for code hosting.

*Tags: `deployment`, `open-source`, `reference`, `scripting`, `tool`*

---

### What is Bolt UXP?

Bolt UXP is a framework or resource for UXP development in Premiere Pro. It was shared as a relevant tool for developers working with UXP plugins.

*Tags: `premiere`, `reference`, `tool`, `ui`, `uxp`*

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

### What resources are available for implementing hot reloading in Xcode development?

For Xcode development, refer to the Stack Overflow discussion on instant run or hot reloading: https://stackoverflow.com/questions/42529081/instant-run-or-hot-reloading-for-xcode. Note that hot reloading was more common in 32-bit builds but became less practical with the transition to 64-bit architecture. Limitations exist when hot reloading is used with After Effects plugins, particularly around parameter setup.

*Tags: `build`, `debugging`, `macos`, `reference`, `tool`*

---

### What is a good library for saving EffectWorld pixel data to image files in AE plugins?

tinypng is a popular library for saving pixel buffer data to image files when working with AE plugin EffectWorld objects. You can access the pixel data via effect_worldP->data and then use tinypng to serialize it to disk.

*Tags: `memory`, `open-source`, `reference`, `tool`*

---

### Is there an open-source searchable database of After Effects command IDs?

Justin Taylor at Hyper Brew maintains an updated searchable Command ID list for After Effects 2022 at https://hyperbrew.co/blog/after-effects-command-ids/, which is useful for scripting and tool development. There was also a Bitbucket snippet repository at https://bitbucket.org/justin2taylor/workspace/snippets/aLjjBE with command ID resources.

*Tags: `extendscript`, `open-source`, `reference`, `scripting`, `tool`*

---

### What is the standard reference for sprintf() function usage?

The standard C sprintf() function is documented at http://www.cplusplus.com/reference/cstdio/sprintf/. This reference explains the format string syntax and argument requirements that apply to PF_SPRINTF macro calls in After Effects plugins.

*Tags: `reference`, `tool`*

---

### What is Dependency Walker and how is it useful for debugging After Effects plugin issues?

Dependency Walker is a diagnostic tool that analyzes executable files and their dependencies. For After Effects plugin development, it helps identify missing or mismatched DLL files and runtime libraries that can cause plugins to fail loading. By running your compiled plugin through Dependency Walker, you can see all unresolved dependencies, which helps troubleshoot errors like 'invalid filter 25::3'. The tool is available at http://www.dependencywalker.com/

*Tags: `build`, `debugging`, `tool`, `windows`*

---

### What tool can I use to check for dependency issues in my After Effects plugin?

Dependency Walker is a useful tool for analyzing plugin dependencies on Windows. It can identify external DLL requirements and help diagnose loading errors. Run it on both 32-bit and 64-bit versions of your plugin to ensure compatibility across architectures.

*Tags: `debugging`, `deployment`, `tool`, `windows`*

---

### What is a guide for setting up Visual C++ Express to compile 64-bit applications?

A patch and setup guide for VC2008 Express 64-bit compilation is available at http://www.cppblog.com/xcpp/archive/2009/09/09/vc2008express_64bit_win7sdk.html, which explains how to configure Visual C++ Express with the Windows SDK to enable 64-bit plugin development.

*Tags: `build`, `reference`, `tool`, `windows`*

---
