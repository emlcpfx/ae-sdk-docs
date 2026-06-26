# Plugin

> 8 Q&As · source: AE plugin dev community Discord

### Should I manually allocate memory in effect plugins or follow the sample plugins' approach?

You should follow what the sample plugins are doing rather than manually allocating memory. Manual allocation is problematic because After Effects won't know whether to use delete[] or free() for deallocation, and AE won't be able to track memory usage if you allocate it yourself.

*Tags: `build`, `memory`, `plugin`*

---

### How can you embed MoltenVK as a dependency inside an After Effects plugin bundle to avoid conflicts with other applications?

When embedding MoltenVK in an AE plugin, the standard approach of placing it in the framework folder as a dynamic library with JSON in resources/vulkan doesn't work for plugin bundles. The scaleUp plugin demonstrates using xcframework to embed it as a static library inside the plugin binary, though this approach requires careful configuration as Vulkan instance creation may fail if not set up correctly. The key challenge is ensuring proper linking and resource resolution within the plugin bundle rather than relying on system-wide installation in /etc/vulkan.

*Tags: `deployment`, `macos`, `plugin`, `reference`, `vulkan`*

---

### What are the advantages of using Lua over Python for developing After Effects plugins?

Lua is significantly better suited for plugin development compared to Python. The main advantage is that Lua is much easier to compile in and integrate as a library. Python integration was problematic because it required asking permission to install libraries all over the user's system, which is intrusive. Lua avoids this dependency management issue entirely. Both languages require similarly tedious C bindings/code generation, but Lua's compilation and integration model is cleaner for plugin distribution.

*Tags: `build`, `deployment`, `plugin`, `scripting`*

---

### How can you bind C code to scripting languages for After Effects plugins?

There are multiple approaches to create C bindings for scripting languages in AE plugins. For Python, you can use existing bindings (like cairo/pycairo) and pass parameters as text using sprintf formatting (e.g., sprintf("%s=%s\n", paramname, value)), then use the language's C API for bitmap and layer parameter assignment. For Lua, you can use XML-based C-binder code generation tools. Both approaches are similarly tedious without automation or code-generation tools to facilitate the binding process.

```cpp
sprintf("%s=%s\n", paramname, value)
```

*Tags: `aegp`, `build`, `plugin`, `scripting`*

---

### How can I detect when my effect plugin is first applied to a layer in After Effects?

Detect a new effect application by setting a flag in sequence data during the SEQUENCE_SETUP call, which is the only call unique to a new application. At that time, you cannot get a layer handle because AE hasn't associated the instance yet. Wait for the UPDATE_PARAMS_UI call (after the flag is set) to get the layer handle. Then iterate through effects on the layer and use AEGP_GetInstalledKeyFromLayerEffect() to compare InstallKeys with your effect's InstallKey to identify duplicates.

*Tags: `aegp`, `params`, `plugin`, `sequence-data`*

---

### How can I restrict an After Effects plugin to specific app versions while keeping it in a single installation folder?

Installing a plugin in the Adobe/Common/Plug-ins/7.0/MediaCore folder makes it available in all versions of After Effects and Premiere Pro. Unfortunately, there is no way to restrict it to specific app versions or applications from the MediaCore folder. To restrict a plugin to only specific versions (e.g., After Effects CC 2014-2019), you must install it in each application version's individual plug-ins folder instead of using the shared MediaCore location.

*Tags: `after effects`, `deployment`, `installation`, `plugin`, `premiere`*

---

### Can I disable an After Effects plugin when it loads in Premiere Pro by checking the application ID?

Previously, it was possible to disable a plugin in Premiere Pro by checking in_data->appl_id for 'PrMr' and returning an error code (-1 or similar) at the beginning of the entrypoint function. However, this method no longer works with Premiere Pro CC2018 and later versions. The recommended approach is to install the plugin in version-specific folders rather than relying on application ID checking.

*Tags: `aegp`, `after effects`, `deployment`, `plugin`, `premiere`*

---

### How can I make PF_Topic rename changes non-undoable without cluttering the undo history?

Start and end an undo group with an empty string as the name (two quotes, not NULL). Wrap your AEGP_SetStreamName() call within this undo group. The operation won't appear in the undo queue, but it remains technically undoable—just not listed in the user-visible undo history.

```cpp
// Start undo group with empty name
A_Err err = AEGP_StartUndoGroup("");
// Perform your name change
AEGP_SetStreamName(cur_streamH, new_name);
// End undo group
AEGP_EndUndoGroup();
```

*Tags: `aegp`, `c++`, `params`, `plugin`, `undo`*

---
