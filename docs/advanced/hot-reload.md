# Hot Reload

> 1 Q&A · source: AE plugin dev community Discord

### Is it possible to hot-reload AE plugins during development without restarting AE?

Hot reload (called 'Edit and Continue' in Visual Studio) was common on 32-bit builds but became less reliable on 64-bit. Even with hot reloading, there are limitations: you can't change ParamSetup because AE calls it only once per session when the plugin is first loaded. Starting AE via the debugger (Visual Studio or Xcode) tends to launch faster than normal startup. Each time you rebuild C++ code, you generally need to restart AE, unlike ExtendScript or CEP where reloads are possible without full restart.

*Tags: `debugging`, `development-workflow`, `hot-reload`, `visual-studio`, `xcode`*

---
