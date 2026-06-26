# Q&A: architecture

**6 entries** tagged with `architecture`.

---

## Can you have multiple effect plugins in one .aex file?

The SDK documentation says it's possible using multiple PiPLs, with AEGPs placed first. However, it's not recommended because: (1) No other hosts including Premiere Pro support multiple PiPLs. (2) If you need to update one plugin, you'd have to ship a new build of all plugins. In practice, adding another PiPL in Xcode with a different name/matchname may only show one plugin. Adobe recommends one PiPL per code fragment.

*Contributors: [**tlafo**](../contributors/tlafo/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2024-05-28 · Tags: `pipl`, `multiple-plugins`, `aex`, `architecture`*

---

## How do you write cross-host plugin code that works for AE, Premiere, OFX, and other hosts?

Write host/format-independent processing code and create thin adapter/wrapper layers per host. This approach has been used successfully for AE/PPro/OFX/Nuke/Frei0r formats. The OFX SDK (partially based on AE SDK) is cleaner in some areas - for example, OFX supports thousands of plugins in a single binary (useful for suites like GMIC with ~2000 plugins), while AE requires separate PiPL/loader for each. When porting, the processing core stays the same and only the parameter declaration and pixel access patterns need per-host adaptation.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: adobe-plugin-devs · 2025-06-06 · Tags: `cross-platform`, `ofx`, `architecture`, `code-reuse`, `plugin-framework`*

---

## Is there a way to create a universal .aex binary for both x64 and Windows ARM, similar to macOS universal binaries?

No, there is no universal binary format for Windows like macOS has. You need to compile separate binaries for each architecture. However, you can cross-compile for both x64 and ARM64 platforms on one system using the latest Visual Studio 2022 (after installing the respective tool sets). Note that AE on ARM is not EC (Emulation Compatible), so unlike Resolve and Nuke which run x64 code emulated on ARM, AE requires native ARM64 plugins. All third-party libraries you link to must also be available in ARM64 format.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/), [**Lloyd Alvarez**](../contributors/lloyd-alvarez/) · Source: aescripts discord · 2025-09-25 · Tags: `windows-arm`, `arm64`, `cross-compile`, `visual-studio`, `architecture`*

---

## What does 'connected instance to instance' mean in the context of After Effects plugins?

It refers to having multiple instances of the plugin on the same or different layers communicating with each other.

*Tags: `aegp`, `params`, `architecture`*

---

## What historical CPU architecture values were stored in the Gestalt API for After Effects plugins?

The Gestalt API values held gestalt values for CPU and FPU on 68x and PowerPC architectures. While the Gestalt API was deprecated on macOS since 10.8, Apple added values for Intel and Silicon ARM CPUs (values 10 and 20). However, Adobe no longer passes these values through to plugins.

*Tags: `macos`, `apple-silicon`, `deprecated`, `reference`, `architecture`*

---

## How do modular plugins like Plexus work, where multiple plugins are stacked to produce a final result?

Modular plugins typically don't render anything themselves. Instead, the individual 'module' effects allow users to input parameters, while a main/master effect reads the parameter values from all the module effects and performs the actual rendering based on those combined parameter values. This is simpler and more elegant than trying to maintain off-screen buffers or create custom channels.

*Tags: `params`, `architecture`, `plugin-design`, `reference`*

---
