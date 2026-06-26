# Plugin Development

> 9 Q&As · source: AE plugin dev community Discord

### How can you add conditional logging to an After Effects plugin without slowing down renders?

Implement a file-based toggle for logging by checking for a specific file (e.g., 'pixel_sorter_log.txt') in global setup at startup. This allows users to enable detailed logging only when needed for debugging, without impacting render performance during normal operation.

*Tags: `debugging`, `logging`, `macos`, `performance`, `plugin-development`*

---

### How can you debug plugin initialization issues by tracking function entry and exit points?

Add logging statements at the entry and exit of key plugin functions, particularly in global setup and parameter setup stages. This helps identify at which stage crashes or entrypoint errors are occurring, making it easier to distinguish between different error types.

*Tags: `aegp`, `debugging`, `plugin-development`*

---

### How can you manage memory for AESDK objects in C++ plugins?

Create C++ wrappers around all AESDK objects to get automatic memory management. This approach helps prevent memory leaks and ensures proper cleanup of After Effects SDK resources.

*Tags: `aegp`, `cpp`, `memory`, `plugin-development`*

---

### Why does my map of effect matchnames to install keys return incorrect install keys when applying effects?

The issue was that the map was declared as `std::map<A_char, A_long>`, storing only a single character instead of the full matchname string. The map should be declared with a string type (e.g., `std::map<std::string, A_long>` or `std::map<A_char*, A_long>`) to properly store and look up the complete effect matchname. When accessing the map with `InstallKeys::keys[*myMatchName]`, only the first character of the matchname was being used as the key, causing incorrect lookups.

```cpp
// Incorrect:
std::map<A_char, A_long> keys;
InstallKeys::keys[*nextMatchName] = nextKey;

// Correct:
std::map<std::string, A_long> keys;
InstallKeys::keys[std::string(nextMatchName)] = nextKey;
```

*Tags: `aegp`, `c++`, `debugging`, `plugin-development`*

---

### How can I prevent users from adding multiple instances of my effect to the same layer?

You can identify your effect uniquely by storing its AEGP_InstalledEffectKey during global setup using AEGP_GetNumInstalledEffects, AEGP_GetNextInstalledEffect, and AEGP_GetEffectMatchName. Then during UPDATE_PARAMS_UI, scan the effect's layer and check each effect's install key using AEGP_GetInstalledKeyFromLayerEffect to detect multiple instances. Once detected, you can alert the user, delete the redundant instance, disable it, or skip rendering for redundant copies by passing input to output. Note that AEGP_DisableCommand cannot remove effects from the menu, and users may still duplicate or copy/paste the effect.

*Tags: `aegp`, `params`, `plugin-development`, `ui`*

---

### How do you prevent video memory leaks when using OpenGL in After Effects plugins, especially during UI parameter changes?

The issue is typically caused by exceptions thrown within OpenGL calls during an interrupt, which causes the code to skip glDeleteTextures calls and jump to catch blocks. To fix this, ensure that glDeleteTextures is called before checking for interrupts using PF_ABORT(), and put breakpoints in catch blocks to verify if exceptions are being thrown during user interaction. The proper order should be: perform GPU operations, delete textures, unbind framebuffers and textures, then check for abort conditions and release suites.

```cpp
glBindFramebuffer(GL_FRAMEBUFFER, 0);
glBindTexture(GL_TEXTURE_2D, 0);
glDeleteTextures(1, &inputFrameTexture);
glDeleteTextures(1, &inputFrameTexture2);
// Then check for abort after cleanup
ERR(PF_ABORT(in_data));
```

*Tags: `debugging`, `gpu`, `memory`, `opengl`, `plugin-development`*

---

### How do you capture when a popup parameter selection changes in an After Effects plugin?

To capture popup menu selection changes, the parameter must be created with the PF_ParamFlag_SUPERVISE flag. When such a parameter is changed, the plugin receives a USER_CHANGED_PARAM call. All plugin calls go through the main entry point function (EffectMain), which should implement the selector to handle this notification.

*Tags: `aegp`, `params`, `plugin-development`, `ui`*

---

### How can I safely use threads in an After Effects plugin while calling AEGP suite functions?

Threading is possible in After Effects plugins, but AEGP suite calls must only be made from the main thread. After Effects behaves unpredictably if suite calls are made from other threads, and the application can become unstable even after the worker thread has completed. Use threads only for background computation; marshal all AE API calls back to the main thread. Boost threads have been confirmed to work reliably when this constraint is followed.

```cpp
static void myFunc(){
  for(int i = 0; i<100; i++){
    // Do computation here, not AE API calls
  }
}
// Call AEGP suites only from main thread
ERR(suites.LayerSuite8()->AEGP_GetActiveLayer(&current_layerH));
```

*Tags: `aegp`, `mainthread`, `plugin-development`, `threading`*

---

### How do you set up After Effects plugin development with the SDK and compile it to .aex format?

To develop an After Effects plugin: 1) Download the SDK (e.g., CC 2015 or later). 2) Ensure you have the required Visual Studio C++ compiler installed (the SDK specifies which version is needed for your AE version). 3) Open one of the example projects from the SDK (such as Convolutrix, which is a good beginner example). 4) In Visual Studio project settings, configure the output directory to point to After Effects' plugin directory or a folder with a shortcut to it. This allows you to build directly into a folder AE reads and debug your plugin. 5) Build the project, which will generate the .aex file. 6) The compiled .aex plugin will then be readable by After Effects. Note that plugin development does not use the Object Model like ExtendScript does—it is C/C++ based SDK development.

*Tags: `aex`, `build`, `plugin-development`, `sdk`, `visual-studio`*

---
