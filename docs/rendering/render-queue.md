# Render Queue

> 2 Q&As · source: AE plugin dev community Discord

### What is the best practice when a plugin runs out of GPU memory during rendering?

Returning PF_Err_OUT_OF_MEMORY may cause AE to show a black frame without warning the user. Options: (1) Use a C++ text library to render 'Out of GPU memory' text directly onto the frame. (2) Set a static global boolean (protected by mutex) in the render thread, then display the warning in another thread like PF_UpdateParamUI. Avoid using out_data warning messages during renders as they would fail render queue operations and prevent MFR from retrying with fewer threads.

*Tags: `error-handling`, `gpu`, `mfr`, `out-of-memory`, `render-queue`*

---

### How can I debug a plugin when it's rendered through AE Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible/separate AE instance for rendering. You can see these processes in the task manager. To debug, attach your debugger to the already running AE process spawned by AME. In Visual Studio, use Debug > Attach to Process. In Xcode, use Debug > Attach to Process by PID or Name.

*Tags: `ame`, `attach-debugger`, `debugging`, `media-encoder`, `render-queue`*

---
