# Aex

> 4 Q&As · source: AE plugin dev community Discord

### Can you have multiple effect plugins in one .aex file?

The SDK documentation says it's possible using multiple PiPLs, with AEGPs placed first. However, it's not recommended because: (1) No other hosts including Premiere Pro support multiple PiPLs. (2) If you need to update one plugin, you'd have to ship a new build of all plugins. In practice, adding another PiPL in Xcode with a different name/matchname may only show one plugin. Adobe recommends one PiPL per code fragment.

*Tags: `aex`, `architecture`, `multiple-plugins`, `pipl`*

---

### Should .aex plugins be code signed on Windows?

.aex plugins are regular DLLs (just renamed), so they can be codesigned. Microsoft currently allows unsigned DLLs to be loaded from a signed process, but they have hinted this might change in the future (was supposed to happen with Win11). On macOS, a plugin is a bundle (renamed folder), and you don't need to sign the actual plugin binary inside it as long as the bundle itself is signed.

*Tags: `aex`, `code-signing`, `dll`, `macos`, `security`, `windows`*

---

### Can the same .aex plugin work in both After Effects and Premiere Pro?

Yes, but it requires correct coding. Premiere does many things differently than AE and not all features are supported in both hosts. The AE SDK documentation has a dedicated section for Premiere Pro compatibility (https://ae-plugins.docsforadobe.dev/). Notably, Premiere does not support SmartRender, which is a significant difference. There are also long-standing bugs in Premiere's implementation of AE plugin APIs (present since at least 2006) that are regularly reported to but largely ignored by Adobe.

*Tags: `aex`, `compatibility`, `cross-host`, `premiere-pro`, `smartrender`*

---

### How do you set up After Effects plugin development with the SDK and compile it to .aex format?

To develop an After Effects plugin: 1) Download the SDK (e.g., CC 2015 or later). 2) Ensure you have the required Visual Studio C++ compiler installed (the SDK specifies which version is needed for your AE version). 3) Open one of the example projects from the SDK (such as Convolutrix, which is a good beginner example). 4) In Visual Studio project settings, configure the output directory to point to After Effects' plugin directory or a folder with a shortcut to it. This allows you to build directly into a folder AE reads and debug your plugin. 5) Build the project, which will generate the .aex file. 6) The compiled .aex plugin will then be readable by After Effects. Note that plugin development does not use the Object Model like ExtendScript does—it is C/C++ based SDK development.

*Tags: `aex`, `build`, `plugin-development`, `sdk`, `visual-studio`*

---
