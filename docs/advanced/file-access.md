# File Access

> 1 Q&A · source: AE plugin dev community Discord

### How can I access the plugin's file path from within a Mac After Effects AEGP plugin to load bundled resources?

For EFFECT plugins, use PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH, path) after including AE_EffectCB.h. This requires the in_data structure which is only available to EFFECT plugins, not AEGP plugins. You may need to convert colons to forward slashes and add "VOLUMES/" at the beginning on Mac. For AEGP plugins (which don't receive in_data), alternative approaches include using getcwd() to read/write files by name only without a full path, or on Windows using GetModuleFileName((HINSTANCE)&__ImageBase, folder, sizeof(folder)).

```cpp
char path[1024] = {'\0'};
PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH, path);
```

*Tags: `aegp`, `cross-platform`, `file-access`, `macos`, `plugin-resources`*

---
