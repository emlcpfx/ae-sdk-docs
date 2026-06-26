# Dependencies

> 1 Q&A · source: AE plugin dev community Discord

### How do you debug the 'plugin wasn't installed correctly' error on Windows?

Use dumpbin (included with Visual C++) to check plugin dependencies: 'dumpbin /dependents plugin.aex'. Common causes: (1) Missing Visual C++ Redistributable packages (download from https://aka.ms/vs/17/release/vc_redist.x64.exe). (2) Missing DLL dependencies. (3) Error in GlobalSetup. (4) Debug builds accidentally linking to debug CRT DLLs (msvcp140d.dll). Try setting a breakpoint on GlobalSetup in debug mode to see if the entry point is reached.

```cpp
dumpbin /dependents plugin.aex
```

*Tags: `debugging`, `dependencies`, `dumpbin`, `installation`, `vcredist`, `windows`*

---
