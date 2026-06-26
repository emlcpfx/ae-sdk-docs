# Api

> 10 Q&As · source: AE plugin dev community Discord

### How can you check if After Effects is running in headless render engine mode?

Use the AppSuite4 API to check the render engine flag. Call PF_IsRenderEngine() to determine if the renderer is running in headless mode, which is useful for plugins that need to behave differently during background rendering versus interactive use.

```cpp
PF_Boolean bIsRenderEngine = true;
suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine);
```

*Tags: `aegp`, `api`, `debugging`, `render-loop`*

---

### How can I retrieve layer comments from within an After Effects plugin using the C API?

There is no direct C API available to get layer comments. The recommended workaround is to use AEGP_ExecuteScript() to execute a script that retrieves the layer comment, and then retrieve the result back to the C side.

*Tags: `aegp`, `api`, `scripting`*

---

### How can I set keyframe velocity and influence values using the AEGP API?

The AEGP_SetKeyframeTemporalEase function is broken and does not properly set keyframe velocity and influence values—these settings simply do not stick. You can set keyframe interpolation types (linear, hold, etc.) but not velocity. A workaround is to use AEGP_ExecuteScript() to execute JavaScript that sets the interpolations instead, though this approach is slower when dealing with many properties and layers.

*Tags: `aegp`, `api`, `keyframe`, `velocity`, `workaround`*

---

### Is there an API to retrieve the rotation value of individual characters when a text animator is applied?

No, there is no such API available in After Effects. However, you can get the text in its current transformation as shape vertices and attempt to deduce the rotation from the vertex data, though this approach is highly problematic and not reliable.

*Tags: `aegp`, `api`, `scripting`, `text`*

---

### How can I replace an image on a layer using the After Effects C API?

There is no direct C API function to replace a layer's source footage. However, you can work around this by using AEGP_ExecuteScript() to execute JavaScript code from within your plugin. Get the layer's comp item ID, then use the JavaScript API to change the layer source. This approach keeps the JavaScript call internal to your plugin, so users are unaware it's being used.

```cpp
AEGP_FootageH footageH = NULL;
ERR(suites.FootageSuite5()->AEGP_NewFootage(S_my_id,
  psd_path,
  &key1,
  NULL,
  FALSE,
  NULL,
  &footageH));
AEGP_ItemH layer_itemH = NULL;
ERR(suites.FootageSuite5()->AEGP_AddFootageToProject(footageH, new_folderH, &layer_itemH));
// Then use AEGP_ExecuteScript() to change the layer source via JavaScript API
```

*Tags: `aegp`, `api`, `layer-checkout`, `scripting`*

---

### Is it possible to override or modify the After Effects Export to XFL command through the API?

No, the After Effects API does not offer means of overriding or modifying the export to XFL command. However, as an alternative, you could apply a script to the generated XML to correct it automatically after export, or run a script that exports data from AE directly in the format you need.

*Tags: `aegp`, `api`, `export`, `scripting`, `xfl`*

---

### How can I execute scripts from within an After Effects plugin?

You can use the AEGP_ExecuteScript() function call to run scripts directly from a plugin without needing to store them as separate files on disk. This is a built-in AEGP suite function for script execution.

*Tags: `aegp`, `api`, `scripting`*

---

### Is there a direct C++ API method to replace a layer's source in After Effects?

There is no direct C++ AEGP API method for replacing a layer's source. The functionality exists in the Java API as layer.source.replace(), but in C++ it must be accessed indirectly using DoCommand with command ID 2299. However, this indirect method requires making the composition the frontmost window, which is difficult to achieve from an AEGP plugin. The feature simulates alt+dragging a new source onto a selected layer in the composition window.

*Tags: `aegp`, `api`, `layer`, `sdk`*

---

### Are After Effects plugins cross-platform compatible between Mac and Windows?

Yes, plugins developed for After Effects are generally cross-platform compatible. Over 99% of the After Effects API is completely cross-platform without special notes. The main differences come when using OS-level APIs (such as opening OS-level windows for options). For core functionality like handling buffers, pixels, RAM, and API suites, the same code can be used on both platforms. However, there are several platform-specific considerations: Windows defines 'long' as 32-bit while Mac defines it as 64-bit (best practice is to use Adobe API types), Windows uses backslashes in paths while Mac uses forward slashes, Windows uses UTF-16 for file operations while Mac uses UTF-8, there are byte order issues for image manipulation, and the build outputs differ (Visual Studio generates a single file while Xcode generates a bundle). For best results, have developers familiar with each platform handle their respective ports.

*Tags: `api`, `cross-platform`, `macos`, `windows`*

---

### What type-definition practices should be used for cross-platform After Effects plugin development?

When developing cross-platform After Effects plugins, it is best practice to use Adobe API-defined types rather than native OS types. This is because Adobe has carefully designed their type definitions to handle platform differences correctly. For example, the native 'long' type is defined as 32-bit on Windows but 64-bit on Mac, which can cause compatibility issues. By using Adobe's API type definitions throughout your code, you avoid these pitfalls and ensure consistent behavior across platforms.

*Tags: `api`, `cross-platform`, `macos`, `windows`*

---
