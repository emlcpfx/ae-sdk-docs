# The "Ordinal 5" Bug: AE 2026 Plugin Loading Failure

> **Note:** Version-specific behavior changes should be verified against the current SDK.

## The Error

When loading MyPlugin in After Effects 2026, the plugin failed to load with this Windows system dialog:

```
AfterFX.exe - Ordinal Not Found
---------------------------
The ordinal 5 could not be located in the dynamic link library
```

The plugin would never appear in the Effects menu. No crash log, no AE error -- just this
system-level dialog at startup and the plugin silently not loading.

## Root Cause

### How AE Loads Plugins

When After Effects scans its plug-in folders at startup, it calls `LoadLibrary` on each `.aex`
file. At this point, Windows' PE loader resolves **all implicit DLL dependencies** listed in the
plugin's import table *before* any plugin code runs.

### The Import Table Problem

Our plugin (`MyPlugin.aex`) was linked against `onnxruntime.lib`, which created an **implicit
import dependency** on `onnxruntime.dll`. The import table referenced the DLL's exports by
**ordinal number** (ordinal 5 in this case) rather than by name.

When AE called `LoadLibrary("MyPlugin.aex")`, Windows attempted to resolve `onnxruntime.dll`
immediately. The search order was:

1. The directory containing `AfterFX.exe`
2. The system directory (`C:\Windows\System32`)
3. The Windows directory
4. The current directory
5. Directories in the `PATH` environment variable

Critically, **the plugin's own directory was NOT in the search path**. The
`onnxruntime.dll` sitting next to `MyPlugin.aex` in the plug-in folder was invisible to the
loader. Windows either couldn't find the DLL at all, or found a different/incompatible version
somewhere else on `PATH` that didn't export ordinal 5.

### Why It Worked Before AE 2026

Earlier AE versions may have used `LoadLibraryEx` with `LOAD_WITH_ALTERED_SEARCH_PATH` or set
the DLL directory before loading plugins, which caused Windows to also search the plugin's
directory. AE 2026 changed its plugin loading behavior, removing this implicit search path
assistance, which broke our implicit dependency resolution.

### The Timing Problem

Our plugin *did* have code to set the DLL search path (`PreloadDlls()` in `MyEngine.cpp`),
but that code lived in `InitializeInternal()`, which runs during the plugin's `PF_Cmd_GLOBAL_SETUP`
handler -- **long after** the PE loader had already tried (and failed) to resolve the import table.

The sequence was:

```
AE startup
  -> LoadLibrary("MyPlugin.aex")
    -> PE loader resolves import table
      -> Needs onnxruntime.dll ordinal 5    <-- FAILS HERE, plugin never loads
      -> (Plugin code never executes)
    -> DllMain would run (if imports resolved)
  -> PF_Cmd_GLOBAL_SETUP would run
    -> PreloadDlls() would set DLL directory   <-- TOO LATE, never reached
```

## The Fix

### 1. Delay-Load `onnxruntime.dll`

The key change was in `CMakeLists.txt`:

```cmake
target_link_libraries(MyPlugin PRIVATE
    "${ONNXRUNTIME_ROOT}/lib/onnxruntime.lib"
    delayimp.lib                              # Microsoft delay-load helper
    ole32.lib
    shell32.lib
    shlwapi.lib
)

target_link_options(MyPlugin PRIVATE
    "/DELAYLOAD:onnxruntime.dll"
)
```

**What `/DELAYLOAD` does:** Instead of putting `onnxruntime.dll` in the PE import table
(which Windows resolves at `LoadLibrary` time), the linker generates small **thunks** for each
imported function. The first time any ONNX Runtime function is actually *called*, the thunk
invokes `__delayLoadHelper2` (from `delayimp.lib`), which calls `LoadLibrary` at that moment
to resolve the DLL.

### 2. The Existing `PreloadDlls()` Does the Rest

The plugin already had the right runtime code in `MyEngine.cpp`:

```cpp
static bool PreloadDlls() {
    std::wstring dir = GetPluginDirectory();   // Find where MyPlugin.aex lives
    SetDllDirectoryW(dir.c_str());             // Add to DLL search path
    AddDllDirectory(dir.c_str());              // Also add to secure search list

    std::wstring ortPath = dir + L"\\onnxruntime.dll";
    HMODULE hOrt = LoadLibraryExW(ortPath.c_str(), NULL, LOAD_WITH_ALTERED_SEARCH_PATH);
    // ...
}
```

This function runs during `MyEngine::EnsureSession()`, which is called when the user
clicks **Analyze**. By that point, the plugin is fully loaded and running -- the delay-load
thunks haven't fired yet because no ORT function has been called.

### Fixed Loading Sequence

```
AE startup
  -> LoadLibrary("MyPlugin.aex")
    -> PE loader resolves import table
      -> onnxruntime.dll is NOT in the import table (delay-loaded)
      -> All other deps (kernel32, ole32, etc.) resolve fine
    -> Plugin loads successfully!
  -> PF_Cmd_GLOBAL_SETUP runs
    -> Plugin registers with AE
  ...
  User clicks "Analyze"
  -> MyEngine::EnsureSession()
    -> PreloadDlls()
      -> SetDllDirectoryW(plugin_dir)           // Set search path
      -> LoadLibraryExW("onnxruntime.dll")      // Explicitly load from plugin dir
    -> OrtGetApiBase()                          // First ORT call -- delay-load thunk fires
      -> __delayLoadHelper2 finds onnxruntime.dll already loaded in memory
      -> Thunk resolves to the correct function pointer
    -> ORT session created successfully
```

## Key Takeaways

1. **Never implicitly link third-party DLLs in AE plugins.** Use `/DELAYLOAD` for any DLL
   that ships alongside the plugin rather than with the OS. The host app controls the
   `LoadLibrary` call for your `.aex`, and you have zero control over the DLL search path at
   that point.

2. **Ordinal-based imports are fragile.** The "ordinal 5" error means the loader was trying to
   find a function by its export table index rather than by name. If the DLL isn't found, or a
   wrong version is found, the ordinal won't match.

3. **`PreloadDlls()` + delay-load is the correct pattern.** Explicitly load your DLLs from a
   known path *before* the delay-load thunks fire. This gives you full control over which DLL
   gets loaded regardless of the host application's search path configuration.

4. **Test plugin loading on fresh machines.** This bug was invisible on dev machines where
   `onnxruntime.dll` might have been on `PATH` from other projects.

## Files Changed

| File | Change |
|------|--------|
| `CMakeLists.txt` | Added `delayimp.lib` to link libraries, added `/DELAYLOAD:onnxruntime.dll` linker option |
| `src/MyEngine.cpp` | No change needed -- `PreloadDlls()` already handled runtime loading correctly |

## References

- [Microsoft: Linker support for delay-loaded DLLs](https://learn.microsoft.com/en-us/cpp/build/reference/linker-support-for-delay-loaded-dlls)
- [Microsoft: Dynamic-link library search order](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order)
