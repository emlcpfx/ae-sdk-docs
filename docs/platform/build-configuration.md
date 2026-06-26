# Build Configuration

> 2 Q&As · source: AE plugin dev community Discord

### How do I build an After Effects plugin for Apple Silicon (M1/ARM64)?

You need to do two things: (1) Set the Xcode build architecture to include arm64 (ensure the architecture setting targets ARM), and (2) Add 'CodeMacARM64 {"EffectMain"}' to your PiPL resource file. Without the PiPL entry, AE won't recognize the plugin as ARM-native even if the binary is compiled for arm64.

*Tags: `apple-silicon`, `arm64`, `build-configuration`, `m1`, `mac`, `pipl`, `xcode`*

---

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
