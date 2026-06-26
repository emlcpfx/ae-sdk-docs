# Media Encoder

> 4 Q&As · source: AE plugin dev community Discord

### Why does Media Encoder not render plugins correctly or fail to load AEGP plugins?

Media Encoder does not load AEGP plugins. If your effect plugin relies on a companion AEGP plugin for rendering, it will fail in Media Encoder. Issues may also be related to static global variables or elements defined in GlobalData/GlobalSetup. The AEGP functionality would need to be incorporated directly into the effect plugin, or you'd need a standalone/command-line tool alternative.

*Tags: `aegp`, `global-data`, `media-encoder`, `render-issues`*

---

### How can I debug a plugin when it's rendered through AE Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible/separate AE instance for rendering. You can see these processes in the task manager. To debug, attach your debugger to the already running AE process spawned by AME. In Visual Studio, use Debug > Attach to Process. In Xcode, use Debug > Attach to Process by PID or Name.

*Tags: `ame`, `attach-debugger`, `debugging`, `media-encoder`, `render-queue`*

---

### Why does a plugin sometimes render without the effect applied when queued to Media Encoder from After Effects?

Static global variables in the plugin and elements defined in globalData or during global initialization thread can cause issues with Media Encoder compatibility. These should be reviewed and potentially refactored to avoid state persistence issues when the plugin runs in ME's different execution context.

*Tags: `debugging`, `media-encoder`, `memory`, `threading`*

---

### How do you debug a plugin when it's loaded through After Effects Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible After Effects instance for rendering, separate from the main AE process. Breakpoints set in the main application won't be triggered during AME rendering. To debug in this scenario, you need to attach your debugger to the already-running subprocess. On Windows, use the Visual Studio debugger's attach-to-process feature: https://learn.microsoft.com/en-us/visualstudio/debugger/attach-to-running-processes-with-the-visual-studio-debugger?view=vs-2022. You can also observe these processes in Windows Task Manager.

*Tags: `debugging`, `deployment`, `media-encoder`, `windows`*

---
