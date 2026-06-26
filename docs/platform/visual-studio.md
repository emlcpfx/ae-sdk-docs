# Visual_Studio

> 2 Q&As · source: AE plugin dev community Discord

### Why does the After Effects SDK build succeed but produce no .dll or .aex file?

By default in the CS4 SDK, plugins build to c:\program files\adobe\after effect 9.0\plug-ins\sdk (note "after effect 9.0" instead of "CS4"). If you don't see output in your expected directory, check the Visual Studio Linker options to verify the output directory is set correctly, or search your computer for the .aex file as it may be building to an unexpected location.

*Tags: `after_effects_sdk`, `build`, `deployment`, `sdk`, `visual_studio`*

---

### How do you fix a Skeleton project that won't build with a web deployment error?

If the Skeleton project shows a conversion warning about "Web deployment to the local IIS server is no longer supported" and won't build, this error disables your build settings. The simplest solution is to duplicate a working project from the SDK and replace only the .cpp and .h files with the Skeleton files, rather than trying to manually fix the corrupted build settings.

*Tags: `after_effects_sdk`, `build`, `sdk`, `skeleton`, `visual_studio`*

---
