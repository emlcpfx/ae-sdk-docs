# Arie Stavchansky

**1 contributions** to AE SDK community knowledge.

Top topics: `hot-reload`, `development-workflow`, `debugging`, `visual-studio`, `xcode`

---

## Is it possible to hot-reload AE plugins during development without restarting AE?

Hot reload (called 'Edit and Continue' in Visual Studio) was common on 32-bit builds but became less reliable on 64-bit. Even with hot reloading, there are limitations: you can't change ParamSetup because AE calls it only once per session when the plugin is first loaded. Starting AE via the debugger (Visual Studio or Xcode) tends to launch faster than normal startup. Each time you rebuild C++ code, you generally need to restart AE, unlike ExtendScript or CEP where reloads are possible without full restart.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `hot-reload`, `development-workflow`, `debugging`, `visual-studio`, `xcode` · [View in Q&A](../qa/hot-reload/)*

---
