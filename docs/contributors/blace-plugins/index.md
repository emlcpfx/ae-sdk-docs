# Blace Plugins

**1 contributions** to AE SDK community knowledge.

Top topics: `debug`, `release`, `pipl`, `visual-studio`, `build-configuration`, `match-name`

---

## How do you set up separate debug and release plugin names in Visual Studio for AE plugins?

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

*Source: aescripts discord · 2026-02-25 · Tags: `debug`, `release`, `pipl`, `visual-studio`, `build-configuration`, `match-name` · [View in Q&A](../qa/debug/)*

---
