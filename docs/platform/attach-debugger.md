# Attach Debugger

> 1 Q&A · source: AE plugin dev community Discord

### How can I debug a plugin when it's rendered through AE Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible/separate AE instance for rendering. You can see these processes in the task manager. To debug, attach your debugger to the already running AE process spawned by AME. In Visual Studio, use Debug > Attach to Process. In Xcode, use Debug > Attach to Process by PID or Name.

*Tags: `ame`, `attach-debugger`, `debugging`, `media-encoder`, `render-queue`*

---
