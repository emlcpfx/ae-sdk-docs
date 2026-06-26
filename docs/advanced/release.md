# Release

> 3 Q&As · source: AE plugin dev community Discord

### How do you set up separate debug and release plugin names in Visual Studio for AE plugins?

Use preprocessor conditionals in the PiPL .r file for both the Name and AE_Effect_Match_Name fields. Add _DEBUG to the PiPL custom build command line. You'll likely need different output filenames too. Watch out for rebuilds cleaning the previous output folder -- add a post-build copy step that copies the generated file from the local folder to the MediaCore folder.

```cpp
Name {
    #ifdef _DEBUG
    "plugin_DEBUG"
    #else
    "plugin"
    #endif
},
AE_Effect_Match_Name {
    #ifdef _DEBUG
    "ADBE plugin_DEBUG"
    #else
    "ADBE plugin"
    #endif
},
```

*Tags: `build-configuration`, `debug`, `match-name`, `pipl`, `release`, `visual-studio`*

---

### Why does SmartRender never get called in Release builds even when PreRender returns PF_Err_NONE?

The issue was caused by a mismatch between the SDK code version used for building (2025.2) and the SDK headers (2023). The debug configuration worked fine because it didn't have this version mismatch. Ensure that both the SDK code and headers are from the same version when building.

*Tags: `build`, `debugging`, `release`, `smart-render`, `smartfx`*

---

### Why does SmartRender never get called in Release mode even when PreRender returns PF_Err_NONE?

SmartRender may not be called if there's a version mismatch between the SDK code used for building and the SDK headers being compiled against. In the reported case, the plugin was being built with 2025.2 SDK code but using 2023 SDK headers, which caused SmartRender to not be invoked in Release builds while Debug builds worked fine. Ensure that the SDK version used for compilation matches the SDK headers included in the project.

*Tags: `build`, `debugging`, `release`, `smart-render`*

---
