# Installation

> 4 Q&As · source: AE plugin dev community Discord

### How do you debug the 'plugin wasn't installed correctly' error on Windows?

Use dumpbin (included with Visual C++) to check plugin dependencies: 'dumpbin /dependents plugin.aex'. Common causes: (1) Missing Visual C++ Redistributable packages (download from https://aka.ms/vs/17/release/vc_redist.x64.exe). (2) Missing DLL dependencies. (3) Error in GlobalSetup. (4) Debug builds accidentally linking to debug CRT DLLs (msvcp140d.dll). Try setting a breakpoint on GlobalSetup in debug mode to see if the entry point is reached.

```cpp
dumpbin /dependents plugin.aex
```

*Tags: `debugging`, `dependencies`, `dumpbin`, `installation`, `vcredist`, `windows`*

---

### Can After Effects CS6 be installed and run on Windows 11?

Yes, AE CS6 runs fine on Windows 11. If you have activation issues with the license key, you may need to run it in compatibility mode. Windows 10 is a fallback option if Windows 11 doesn't cooperate with activation.

*Tags: `compatibility`, `cs6`, `installation`, `legacy`, `windows-11`*

---

### How should you handle shared C++ plugins that are used by multiple CEP extensions to avoid uninstall conflicts?

Several approaches: (1) Use the manifest system to flag dependencies for each product, along with a plugin version, and include the version in the plugin name to allow side-by-side installation. (2) Compile the same plugin source multiple times with different build flags that make each compiled plugin used only by one panel/product. (3) Bundle all products together with one merged versioning so users install/uninstall all at once. The core problem is that uninstalling any individual extension currently also uninstalls the shared plugin, breaking other extensions.

*Tags: `aescripts`, `cep-extensions`, `dependency-management`, `installation`, `shared-plugin`*

---

### How can I restrict an After Effects plugin to specific app versions while keeping it in a single installation folder?

Installing a plugin in the Adobe/Common/Plug-ins/7.0/MediaCore folder makes it available in all versions of After Effects and Premiere Pro. Unfortunately, there is no way to restrict it to specific app versions or applications from the MediaCore folder. To restrict a plugin to only specific versions (e.g., After Effects CC 2014-2019), you must install it in each application version's individual plug-ins folder instead of using the shared MediaCore location.

*Tags: `after effects`, `deployment`, `installation`, `plugin`, `premiere`*

---
