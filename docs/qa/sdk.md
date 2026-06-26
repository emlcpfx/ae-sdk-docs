# Q&A: sdk

**139 entries** tagged with `sdk`.

---

## What are the AE SDK documentation URLs?

The old URL ae-plugin-sdk.aenhancers.com is defunct. The correct current URL for AE plugin SDK documentation is https://ae-plugins.docsforadobe.dev/

*Contributors: [**James Whiffin**](../contributors/james-whiffin/) · Source: adobe-plugin-devs · 2025-01-15 · Tags: `documentation`, `sdk`, `resources`*

---

## Are there any good C++ wrapper libraries for the After Effects Filter GUI API?

No well-known general-purpose wrapper exists. Most wrappers that have been built end up being very complex themselves if you try to handle all use cases -- you end up with a non-standard system that is almost as complex as the original, just with more C++ features. Copy-pasting C chunks from the SDK examples works well in practice. That said, even a C wrapper with automatic memory management and state stored in structs would be an improvement over the current state machine. There is also a Rust binding project (https://github.com/virtualritz/after-effects/) that significantly reduces boilerplate compared to C/C++.

*Contributors: [**Gabriel Grenier**](../contributors/gabriel-grenier/), [**Antoine_Autokroma**](../contributors/antoine-autokroma/), [**Lloyd Alvarez**](../contributors/lloyd-alvarez/) · Source: aescripts discord · 2024-09-27 · Tags: `wrapper`, `gui-api`, `cpp`, `rust`, `boilerplate`, `sdk`*

---

## Are developers allowed to redistribute the Adobe SDKs publicly?

Most likely no. You should check with Adobe directly, but the general expectation is that redistribution is not permitted.

*Contributors: [**Lloyd Alvarez**](../contributors/lloyd-alvarez/) · Source: aescripts discord · 2024-10-02 · Tags: `sdk`, `licensing`, `redistribution`, `adobe`*

---

## Where can I find code to get all active camera properties?

The Resizer example in the SDK contains code for accessing camera properties and interfacing with camera layers.

*Tags: `aegp`, `camera`, `sdk`, `layer-checkout`*

---

## Is DirectX rendering already implemented in After Effects production, and has anyone made a plugin using it?

DirectX rendering was not widely available for plugins as of the conversation date. Adobe replaced the UI from OpenGL to DX12 in 2021, and the latest SDK has an official flag for it, but it's likely not yet ready to expose to plugins. It may be added for Windows on ARM device support, where DirectX is the first-party option. Adobe has not been supporting Vulkan, possibly because the version of AFX that passes DX12 handles to plugins hasn't been released yet.

*Tags: `gpu`, `directx`, `windows`, `apple-silicon`, `sdk`*

---

## Where can I find an example solution for color grid implementation in After Effects plugins?

The colorgrid example in the After Effects SDK contains the solution. This is a reference implementation that demonstrates how to work with color grids in plugin development.

*Tags: `reference`, `sdk`, `open-source`*

---

## How can I get all of the active camera's properties in an After Effects plugin?

The Resizer example in the After Effects SDK demonstrates how to work with camera layers and access their properties. This example is a good reference for interfacing with camera layers and retrieving their active properties.

*Tags: `aegp`, `reference`, `sdk`*

---

## What is the purpose of the changedB parameter in PathOutline, and can paths be modified through the SDK?

The changedB parameter appears to signal whether path data has been modified, though the exact mechanism is unclear. The public SDK does not expose functions for modifying paths—they are read-only. It's theorized that After Effects internally uses an undocumented API for path modification that mirrors the PathOutline disposal API, but this functionality is not available to plugin developers.

*Tags: `aegp`, `params`, `sdk`, `reference`*

---

## Why do lock/unlock functions still exist in the After Effects SDK if they weren't ported to 64-bit?

According to Tobias Fleischer, the lock/unlock functions were not ported when the AE codebase moved to 64-bit in CS6 (2011), yet they still exist in the current SDK and sample code calls them. While rowbyte notes these functions are somewhat redundant in a 64-bit address space, they still provide a safe way to dereference handles (which are effectively double pointers in 64-bit) without confusion. The SDK provides a DH macro as an alternative for direct dereferencing.

*Tags: `aegp`, `memory`, `sdk`, `64-bit`, `reference`*

---

## What changes are coming to the Premiere SDK regarding capture and recording tools?

According to the SDK roadmap, Premiere will be removing tools for capture recording functionality in the future, signaling the deprecation of tape-based workflows.

*Tags: `premiere`, `sdk`, `deployment`*

---

## How can I make a plugin support 8-bit, 16-bit, and 32-bit color depths without duplicating code for each bit depth?

Use C++ templates to create a single implementation that can be instantiated for different data types and range constants. This allows you to share the same core logic while supporting multiple bit depths. You can also manually convert higher bit-depth input to 8-bit within the plugin itself if needed, rather than relying on After Effects' automatic conversion which may introduce unwanted color noise.

*Tags: `templates`, `bit-depth`, `color-processing`, `codebase-reuse`, `sdk`, `c++`*

---

## How can I make an After Effects plugin save a new file when a UI button is clicked?

The After Effects SDK only handles importing files into projects or rendering to custom file types. To save arbitrary data to a file from a button click in a plugin, use standard C file I/O functions directly: fopen(), fwrite(), and fclose(). This is similar to how you would use .saveDlg(), .open(), .write(), and .close() in ExtendScript/JavaScript.

```cpp
fopen/fwrite/fclose
```

*Tags: `ui`, `aegp`, `sdk`, `reference`*

---

## How can I checkout masks and paths from a different layer using the After Effects SDK?

You can fetch any mask using AEGP_MaskSuite6, then read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape parameter, you need to traverse the layer's dynamic streams. However, be aware that mask and shape vertex data is delivered unmodified—masks won't include the "expand" parameter modification, and shapes won't show rounding if applied by the user. Additionally, some shapes are parametric (like star shapes) and have no shape parameter to read.

*Tags: `sdk`, `layer-checkout`, `aegp`, `masks`, `sequence-data`*

---

## What is the correct first argument to pass to transform_world and other suite callbacks in After Effects plugins?

According to community expert Shachar Carmi, the Adobe documentation is incorrect about the first argument. Instead of passing in_data as documented, you should pass NULL or in_data->effect_ref. Passing NULL works because many suite callbacks accept null instead of effect_ref, and the PF_INTERRUPT macro internally uses effect_ref to make interrupt checks. The incorrect documentation may be legacy from before CC2015 when a separate rendering thread was introduced.

*Tags: `sdk`, `transform_world`, `params`, `debugging`, `reference`*

---

## Does After Effects provide built-in matrix multiplication functions?

After Effects does not offer native matrix handling functions. However, the Artie sample plugin contains code for multiplying 4x4 matrices. For 3x3 matrix multiplication, you can implement your own using standard matrix multiplication algorithms, such as the example from Stack Overflow: https://stackoverflow.com/questions/25250188/multiply-two-matrices-in-c

```cpp
for (int i = 0; i < a; i++)
  for (int j = 0; j < d; j++) {
    Mat3[i][j] = 0;
    for (int k = 0; k < c; k++)
      Mat3[i][j] += Mat1[i][k] * Mat2[k][j];
  }
```

*Tags: `reference`, `sdk`, `tool`*

---

## How can I get the mouse position relative to the composition when the user clicks in the comp preview window?

Mouse coordinates are passed to the plugin through UI event callbacks. You can get the mouse click location in composition window coordinates and convert those into layer source coordinates. The CCU SDK sample project demonstrates how to implement this functionality with working code examples.

*Tags: `ui`, `reference`, `sdk`, `mouse-input`, `coords`*

---

## What is a good SDK sample project that demonstrates handling mouse clicks in the composition window?

The CCU SDK sample project is an official Adobe After Effects SDK example that shows how to get mouse click locations in comp window coordinates and convert them into layer source coordinates. It provides a complete working implementation for custom UI event handling.

*Tags: `reference`, `sdk`, `open-source`, `ui`, `sample-project`*

---

## Is there an API to set custom layer icons for AEGP plugins?

No official API exists for customizing layer icons in AEGP. While there are no standard SDK methods to change layer icons (like the star icon for shape layers or camera icon for camera layers), some developers have explored workarounds by directly manipulating the After Effects interface as an OS window, though this approach is not recommended or officially supported.

*Tags: `aegp`, `ui`, `sdk`*

---

## How should I handle string conversions between std::string and After Effects A_char and A_UTF16Char types?

Create a utility class with static methods for bidirectional string conversion. Use strcpy_s for A_char conversion and std::wstring_convert with std::codecvt_utf8_utf16 for UTF-16 conversions. Always ensure null-termination and check buffer bounds before copying. Use try-catch blocks for UTF-16 conversion to handle range_error exceptions.

```cpp
#include <locale>
#include <codecvt>
#include <string>
class AEStringConverter {
public:
  static PF_Err StringToAChar(const std::string& inString, A_char* outAChar, size_t bufferSize) {
    PF_Err err = PF_Err_NONE;
    errno_t copyResult = strcpy_s(outAChar, bufferSize, inString.c_str());
    if (copyResult != 0) err = PF_Err_OUT_OF_MEMORY;
    return err;
  }
  static PF_Err StringToAUTF16Char(const std::string& inString, A_UTF16Char* outAChar, size_t maxOutChars) {
    PF_Err err = PF_Err_NONE;
    std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
    std::wstring utf16String = converter.from_bytes(inString);
    if (utf16String.size() + 1 > maxOutChars) return PF_Err_OUT_OF_MEMORY;
    std::copy(utf16String.begin(), utf16String.end(), outAChar);
    outAChar[utf16String.size()] = L'\0';
    return err;
  }
  static PF_Err AUTF16CharToString(const A_UTF16Char* inAUTF16Char, std::string* outString) {
    PF_Err err = PF_Err_NONE;
    try {
      std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
      *outString = converter.to_bytes((wchar_t*)inAUTF16Char);
    } catch (const std::range_error&) {
      err = PF_Err_OUT_OF_MEMORY;
    } catch (...) {
      err = PF_Err_OUT_OF_MEMORY;
    }
    return err;
  }
};
```

*Tags: `sdk`, `string-handling`, `character-encoding`, `error-handling`, `cross-platform`, `build`*

---

## What is the correct error code to return from an After Effects plugin when a string conversion or operation fails?

Use PF_Err_OUT_OF_MEMORY as the standard error code for failed operations. Avoid PF_Err_INTERNAL_STRUCT_DAMAGED because it tells After Effects that the effect instance is corrupted and After Effects will stop communicating with that instance. PF_Err_OUT_OF_MEMORY is the appropriate way to signal that an operation failed and its results should be ignored.

*Tags: `sdk`, `error-handling`, `aegp`*

---

## How can I predict the required buffer size for string conversions in Windows?

Use the Windows API function MultiByteToWideChar with a null pointer for the output buffer parameter. This will return the required buffer size in wide characters without actually performing the conversion. This allows you to accurately allocate the necessary buffer before conversion.

*Tags: `sdk`, `windows`, `string-handling`, `memory`*

---

## How do you properly apply CANNOT_TIME_VARY and COLLAPSE_TWIRLY flags to arbitrary parameters in After Effects plugins?

These flags should be placed in the param flags argument (one position before the UI flags argument), not in the PF_PUI_CONTROL flags. The param flags and param UI flags are separate arguments in PF_ADD_ARBITRARY2, and CANNOT_TIME_VARY and COLLAPSE_TWIRLY belong in the param flags position, not the UI flags position where PF_PUI_CONTROL resides.

```cpp
PF_ADD_ARBITRARY2("Custom Parameter",
  UI_BOX_WIDTH,
  UI_BOX_HEIGHT,
  PF_ParamFlag_CANNOT_TIME_VARY | PF_ParamFlag_COLLAPSE_TWIRLY,
  PF_PUI_CONTROL | PF_PUI_DONT_ERASE_CONTROL,
  def.u.arb_d.dephault,
  PARAM_DISK_ID,
  ARB_REFCON);
```

*Tags: `params`, `arb-data`, `ui`, `sdk`*

---

## Is it possible to hot reload After Effects plugins without restarting the application?

No, hot reloading is not practical for native C++ After Effects plugins. Each time you build C++ code, you must restart After Effects because the application calls parameter setup only once per session when the plugin is first loaded, so hot reloading cannot trigger re-initialization of parameters. However, ExtendScript and CEP plugins can be reloaded without restarting the entire AE application. Debugging through an IDE like Microsoft Visual Studio or Apple Xcode can make the restart cycle faster than manually launching AE.

*Tags: `params`, `build`, `debugging`, `sdk`, `scripting`*

---

## Is there a sample plugin that demonstrates resizing the output frame?

The 'Resizer' SDK sample project is the recommended reference for learning how to resize output frames in After Effects plugins.

*Tags: `reference`, `sdk`, `output-rect`*

---

## What needs to be unique when creating multiple arbitrary parameters in an After Effects plugin?

When creating multiple arbitrary parameters, each parameter needs its own unique parameter ID (like CUSTOMUI_DISK_ID). The ARB_REFCON can be anything you want—it doesn't need to be unique; After Effects doesn't care about its content. What matters is how you use it: you can point it to a function handler or class instance specific to that parameter. Each arbitrary parameter must have its own allocated handle for the default value, even if the default values are the same. The PF_CustomUIInfo registration only needs to be done once per PARAM_SETUP call, not per parameter.

```cpp
switch(param_index) {
  case ARB_1_INDEX:
    ArbHandler1();
    break;
  case ARB_2_INDEX:
    ArbHandler2();
    break;
}
```

*Tags: `arb-data`, `params`, `ui`, `sdk`*

---

## How should you handle multiple arbitrary parameters in the HandleArbitrary function?

Rather than using a switch statement on param_index for each arbitrary parameter, a more elegant C++ approach is to use ARB_REFCON as a pointer to the appropriate handler (either a function pointer or class instance). Each arbitrary parameter gets its own ARB_REFCON pointing to its specific handler. This eliminates the need for large switch statements and allows you to use a class-based architecture where a generic arb base class handles common operations and specific subclasses override methods as needed. This approach scales much better for plugins with many arbitrary parameters.

*Tags: `arb-data`, `params`, `sdk`*

---

## How can I access post-processed shape path vertices (with effects like offset, zig zag, or wiggle applied) using the After Effects SDK?

Unfortunately, After Effects does not provide direct access to post-processed paths through the SDK. You can only read the parameter streams directly and then manually emulate the processed path behavior by implementing the effects yourself.

*Tags: `aegp`, `sdk`, `shape-layers`, `params`*

---

## Should I use multithreading or MFR in my After Effects plugin, or can both be used together?

Both multithreading (MT) and Multi-Frame Rendering (MFR) can coexist in plugins. While they may seem incompatible at first, MFR and MT don't necessarily compete for CPU resources because not all steps in the rendering pipeline of a single frame are multi-threadable. MFR excels at handling non-multi-threadable steps, while the multi-threadable parts are often CPU cache-friendly and don't compete with MFR threads. The Adobe After Effects engineering team has extensively tested these scenarios and optimized the mechanism to handle a wide range of rendering scenarios, including plugins that aren't MFR-enabled and those relying on GPU rendering.

*Tags: `mfr`, `threading`, `render-loop`, `sdk`*

---

## What is the Supervisor SDK sample project and what does it demonstrate?

The Supervisor SDK sample project is an official Adobe After Effects SDK example that demonstrates proper parameter supervision and UI updating. While somewhat convoluted, the first few lines of its UpdateParameterUI() function show the correct implementation for using PF_UpdateParamUI with parameter copies and proper flag handling.

*Tags: `sdk`, `params`, `ui`, `reference`, `open-source`*

---

## How can you enable an alpha channel in a COLOR parameter for user editing?

The native COLOR parameter in After Effects does not support alpha channel editing in the color picker. According to community experts, you cannot enable this directly. However, you have two workarounds: (1) Use PF_AppColorPickerDialog to access the system color dialog, though it typically won't have an alpha component either, or (2) Implement your own custom color picking dialog and launch it via param supervision or custom UI. Alternatively, consider adding a separate opacity parameter instead of trying to include alpha in the color picker.

*Tags: `params`, `ui`, `sdk`, `color-picker`*

---

## How do you set keyframes on effect parameters during plugin initialization?

Setting keyframes during GlobalSetup won't work because the effect_ref is garbage at that point since there's no effect instance yet. Instead, set keyframes on the first call to UPDATE_PARAMS_UI. Alternatively, if you're adding the effect via script, set the keyframes through the script instead of the plugin code. The correct approach involves using the AEGP KeyframeSuite functions (AEGP_StartAddKeyframes, AEGP_AddKeyframes, AEGP_SetAddKeyframe, AEGP_EndAddKeyframes) with a valid effect reference obtained after the effect instance is created.

```cpp
ERR(suites.UtilitySuite5()->AEGP_RegisterWithAEGP(NULL, pluginName, &pluginID));
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(pluginID, in_data->effect_ref, &effectPH));
ERR(suites.StreamSuite5()->AEGP_GetNewEffectStreamByIndex(pluginID, effectPH, 25, &streamH));
ERR(suites.StreamSuite5()->AEGP_GetNewStreamValue(pluginID, streamH, AEGP_LTimeMode_CompTime, 0, TRUE, &valueP));
valueP.val.one_d = 20;
ERR(suites.KeyframeSuite4()->AEGP_StartAddKeyframes(streamH, &akH));
ERR(suites.KeyframeSuite4()->AEGP_AddKeyframes(akH, AEGP_LTimeMode_CompTime, 0, &indexPL));
ERR(suites.KeyframeSuite4()->AEGP_SetAddKeyframe(akH, indexPL, &valueP));
ERR(suites.KeyframeSuite4()->AEGP_EndAddKeyframes(true, akH));
```

*Tags: `params`, `aegp`, `scripting`, `sdk`*

---

## How can I detect when an effect is removed from a layer or when a layer is deleted in an After Effects plugin?

There is no direct API for detecting effect removal, but you can use two approaches: (1) Scan the project on an idle hook to catalog instances of your effect, and if an effect that was present is missing on the next scan, it was removed (note: this is not immediate and happens on the next idle hook call after deletion). (2) Monitor cut and delete operations using command hooks, though this is complex and unreliable because not all deletion methods trigger menu commands (e.g., redo operations just set project state without triggering delete commands). The PF_Cmd_SEQUENCE_SETDOWN command is called when the project closes or memory is purged, not when the effect is removed by the user. First, you need to define what 'removal' means for your use case (e.g., does it include undo states?) before choosing the best approach.

```cpp
case PF_Cmd_SEQUENCE_SETDOWN:
err = SequenceSetdown(in_data,out_data);
err = SequenceSetdown2(in_data, out_data);
break;
```

*Tags: `aegp`, `sequence-data`, `debugging`, `sdk`*

---

## How can I access AEGP suites within an IdleHook callback?

The IdleHook callback does not have direct access to in_data, so you cannot directly access suites through the in_data->pica_basicP pointer. You need to store a reference to the AEGP_SuiteHandler or pica_basicP pointer in a global or static variable during plugin initialization (in EffectMain or at plugin startup), and then retrieve it in your IdleHook callback to execute suite operations.

*Tags: `aegp`, `threading`, `sdk`*

---

## Why does an After Effects effect plugin crash when saving a project after upgrading to the latest SDK with MFR support?

The crash on save is likely related to sequence data flattening. When upgrading to newer SDKs, if you set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag during global setup but don't implement the PF_Cmd_SEQUENCE_FLATTEN selector, After Effects will crash when trying to save the project. The SEQUENCE_FLATTEN command is sent when saving or duplicating sequences and requires you to flatten sequence data containing pointers or handles so it can be written to disk. Additionally, verify you're allocating new handles when modifying instance sequence data rather than changing data in the same handle. Basic troubleshooting includes cleaning build intermediates, rebuilding from scratch, testing sample SDK projects, and progressively commenting out plugin code to isolate the problem.

*Tags: `mfr`, `sequence-data`, `sdk`, `debugging`, `build`*

---

## What is the purpose of the PF_Cmd_SEQUENCE_FLATTEN selector in After Effects plugins?

PF_Cmd_SEQUENCE_FLATTEN is sent by After Effects when saving or duplicating sequences. It requires the plugin to flatten sequence data containing pointers or handles so the data can be correctly written to disk and saved with the project file. The flattened data must be properly byte-ordered for file storage. When this command is received, the plugin should free unflat data and set out_data->sequence_data to point to the new flattened data. As of SDK 6.0, if an effect's sequence data has been flattened, the effect may be deleted without receiving PF_Cmd_SEQUENCE_SETDOWN, and After Effects will dispose of the flat sequence data.

*Tags: `sequence-data`, `sdk`, `arb-data`, `deployment`*

---

## Can I save sequence data during SequenceSetdown?

No. SequenceSetdown is AE's mechanism for telling your plugin that data needs to be destructed and freed. The data saved with the project is the last handle used or flattened (depending on whether you have set the flattening flag). SequenceSetdown is not the appropriate place to save data, as the logic is that the data may not just be a simple handle to be freed, but rather a structure with pointers that need to be separately destructed and freed.

*Tags: `sequence-data`, `memory`, `aegp`, `sdk`*

---

## How do I properly flatten sequence data so it persists when reopening a project?

To flatten sequence data, set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag during global setup. This will cause your plugin to receive PF_Cmd_SEQUENCE_FLATTEN calls where you can perform the flattening. Do not attempt to flatten data in SequenceSetdown, as that callback is only for destructing and freeing data, not for saving it.

*Tags: `sequence-data`, `params`, `sdk`, `aegp`*

---

## How can I detect when an After Effects project is being saved or opened in a plugin?

There is no reliable way to detect in advance that a project is about to save. However, you can use sequence data to handle save events. Set the PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING flag on your effect instance, and After Effects will call your effect with a handle to store data during save operations, copy/paste events, and other occasions when sequence data is persisted. You can also check during idle time if the project dirty status has changed to deduce that a save occurred, but this is reactive rather than predictive.

*Tags: `sequence-data`, `aegp`, `sdk`*

---

## Can you hide the second dropdown in PF_ADD_LAYER layer selector parameters?

No, the second dropdown (which allows selection between "Source", "Masks", or "Effects & Masks") cannot be hidden. That is the standard appearance of the layer selector in current versions of the After Effects SDK.

*Tags: `params`, `ui`, `sdk`, `mfr`*

---

## How do you get the duration of a composition layer in the After Effects SDK for C++?

To get layer duration, use AEGP_GetLayerDuration(). To get composition duration, use AEGP_GetItemFromComp() followed by AEGP_GetItemDuration().

*Tags: `aegp`, `sdk`, `c++`, `reference`*

---

## How can a C++ effect plugin modify transform parameters of other layers in a project?

Use the StreamSuite to affect any parameter (which are internally streams) in the project. The 'project dumper' sample project demonstrates how to access streams. However, since CC2015, you cannot change the project during a render call. To modify project parameters, use UI events (anything other than render and pre-render calls) when you can modify the project as needed.

*Tags: `aegp`, `params`, `render-loop`, `streaming`, `sdk`*

---

## Is there an official sample project demonstrating how to access and manipulate streams in the After Effects SDK?

Yes, Adobe provides the 'project dumper' sample project as part of the After Effects SDK. This sample demonstrates how to access streams and is useful for understanding how to affect parameters in a project using the StreamSuite.

*Tags: `reference`, `open-source`, `sdk`, `aegp`, `streaming`*

---

## How do I resolve 'Cannot open source file' errors when moving After Effects SDK example projects to a new location?

When copying AE SDK example projects like the supervisor folder to a new location, include directories only apply to .h and .hpp header files, not to .c and .cpp source files. Source files use relative paths from the Visual Studio project file location. To fix the error, either re-import the missing files or maintain the original SDK folder structure relative to where your .vcxproj file is located. Simply adjusting the include directories in project properties will not resolve paths for source files.

*Tags: `build`, `windows`, `sdk`, `reference`*

---

## How can I prevent the timeline properties from opening when setting AEGP_DynStreamFlag_HIDDEN?

According to shachar carmi, there is no way to expose a parameter in the ECW (Effect Control Window) without exposing it in the timeline. As a workaround, try using PF_ParamFlag_SKIP_REVEAL_WHEN_UNHIDDEN flag, which may prevent the parameter from being exposed in both places. Alternatively, PF_UpdateParamUI differs from AEGP_SetDynamicStreamFlag in its behaviors and may produce more suitable results. The 'Supervisor' SDK sample project demonstrates how PF_UpdateParamUI is used.

*Tags: `aegp`, `params`, `ui`, `sdk`*

---

## What is a good SDK sample project to learn how to manage dynamic parameter visibility?

The 'Supervisor' SDK sample project is recommended as a reference for understanding how to use PF_UpdateParamUI for managing parameter visibility and behavior in After Effects plugins. It demonstrates techniques for controlling whether parameters appear in the Effect Control Window and timeline.

*Tags: `sdk`, `reference`, `params`, `ui`*

---

## What is the most performant way to iterate through pixels when converting 8-bit to 16-bit data in After Effects plugins?

Use IterateGeneric to iterate through each line instead of each pixel, which provides much better performance compared to iterating pixel-by-pixel. Additionally, consider using PF_COPY for the actual pixel conversion operation, and refer to methods from the Transformer example that convert PF_Pixel to PF_Pixel16 if needed.

*Tags: `memory`, `caching`, `performance`, `aegp`, `sdk`*

---

## How do you disable a layer's video switch from a C++ plugin effect?

Use AEGP_GetEffectLayer to get the layer handle from the effect reference, then use AEGP_SetLayerFlag with AEGP_LayerFlag_VIDEO_ACTIVE set to FALSE to disable the layer's video output. This is more efficient for render performance than setting opacity to 0.

```cpp
AEGP_LayerH layerH;
suites.AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
suites.AEGP_SetLayerFlag(layerH, AEGP_LayerFlag_VIDEO_ACTIVE, FALSE);
```

*Tags: `aegp`, `layer-checkout`, `performance`, `sdk`*

---

## What language and tools are needed to create After Effects plugins?

After Effects plugins are written in C/C++. While you can work with other languages, all calls to the AE API must be in C/C++. The SDK is shipped with Visual Studio project files for Windows and Xcode project files for Mac. Effect controls windows and composition window overlay graphics are created using the DrawBot suite supplied by the AE SDK.

*Tags: `sdk`, `c/c++`, `windows`, `macos`, `ui`, `build`*

---

## How can I localize parameter labels in After Effects plugins to display in different languages like Chinese?

Adobe provides official documentation on localization for After Effects plugins. The localization guide is available in the official After Effects SDK documentation at https://ae-plugins.docsforadobe.dev/intro/localization.html, which covers how to implement parameter label localization for different languages.

*Tags: `params`, `ui`, `localization`, `reference`, `sdk`*

---

## How can I access and modify layer styles like inner shadow using the AE SDK?

Use AEGP_DynamicStreamSuite4 to access layer styles. You need to explore the layer's dynamic stream groups to find the specific style properties you want to modify, such as inner shadow values.

*Tags: `aegp`, `layer-checkout`, `sdk`, `reference`*

---

## How can I update plugin parameters when the effect is first applied to a layer?

Parameter value changes made directly during UPDATE_PARAMS_UI are ignored by After Effects. Instead, use SetStreamValue to modify parameter values during initialization, which will persist the changes. While not considered best practice, this is the reliable method to initialize parameters on first application of the effect.

```cpp
params[EM_TOPLEFT]->u.td.x_value = FLOAT2FIX(globP->x0);
params[EM_TOPLEFT]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `sdk`, `update`*

---

## Why does AEGP_GetLayerToWorldXform return zeros when called from multiple threads using Iterate8Suite1?

Most AEGP functions are not thread-safe. AEGP_GetLayerToWorldXform should only be called from the main thread that After Effects called your plug-in with. Calling it from other threads will return zeros or cause internal verification failures. Only a few AEGP functions are documented as thread-safe.

*Tags: `aegp`, `threading`, `memory`, `sdk`*

---

## What is the recommended sample project for learning how to create a dockable panel in After Effects?

The SDK sample project 'Panelator' demonstrates how to create a dockable panel in After Effects. This sample shows the fundamentals of building a panel interface that can be populated with custom controls and integrated into the AE workspace.

*Tags: `aegp`, `ui`, `reference`, `sdk`*

---

## How should parameter IDs be numbered to avoid issues when reordering parameters between plugin versions?

Parameter IDs in the non-_ID enum should start from 1, not 0. Value 0 is reserved for the default invisible layer parameter that serves as the effect's input. If you start numbering IDs from 0 (NULL), After Effects may assume you haven't bothered to ID params and will auto-assign IDs, which causes problems when reordering parameters in new versions. Always explicitly set parameter IDs starting from 1 to maintain backwards compatibility.

```cpp
enum
{
  MOVEMENT_DIRECTION_ID = 1,
  AMOUNT_SLIDER_ID = 2,
  NEW_FIELD_SLIDER_ID = 3
}
```

*Tags: `params`, `sdk`, `versioning`, `backwards-compatibility`*

---

## What is the recommended approach for changing parameter order in newer versions of an After Effects plugin while preserving saved project data?

Adobe's AE Plugin SDK Guide provides detailed documentation on changing parameter orders: https://ae-plugin-sdk-guide.readthedocs.io/effect-details/changing-parameter-orders.html. The guide explains how to properly handle parameter ID assignment and reordering to maintain backwards compatibility with projects saved using older plugin versions.

*Tags: `params`, `versioning`, `reference`, `sdk`*

---

## Where can I find the official After Effects SDK guide?

The After Effects SDK guide is hosted at https://ae-plugin-sdk-guide.readthedocs.io/. This is the current location for the official documentation after previous links were moved.

*Tags: `sdk`, `reference`, `documentation`, `aegp`*

---

## How can I use AEGP_FastBlur from WorldSuite2 within an effect plugin instead of an AEGP?

If you need to blur the input layer, you must copy its content into an AEGP_WorldH. However, for intermediate processing, you can create an AEGP_WorldH and use AEGP_FillOutPFEffectWorld to wrap the AEGP world into an effect world, allowing you to access the same world in both forms.

*Tags: `aegp`, `effectworld`, `blur`, `worldsuite`, `sdk`*

---

## What should be passed for the AEGP_PluginID argument when calling AEGP_New from within an effect plugin?

Passing NULL for the AEGP_PluginID argument works in effect plugins, though this may not be the recommended approach. The AEGP_PluginID is normally required by AEGP functions, but NULL can be used as a workaround when calling AEGP_New from within an effect context.

*Tags: `aegp`, `pluginid`, `effect`, `sdk`*

---

## How should PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS be used correctly when adding new parameters to legacy plugin versions?

The flag PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS is meant to provide a default value for old projects that don't have the new parameter. However, the PF_ADD_CHECKBOX macro copies the passed default over the passed value, which can cause unexpected behavior. The solution is to either copy the macro code into your own files and fix it there, or manually define the parameter without using the macro, defining it the old-fashioned way. Avoid modifying the AE headers directly as SDK updates would undo the changes.

```cpp
def.u.bd.value = FALSE;        // value for legacy projects which did not have this param
PF_ADD_CHECKBOX(
    STR(StrID_Checkbox_Param_Name),
    STR(StrID_Checkbox_Description),
    TRUE, // value for new applications, and when reset
    PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS,
    DOWNSAMPLE_DISK_ID);
```

*Tags: `params`, `pipl`, `sdk`, `reference`*

---

## How can I create off-screen image buffers in After Effects SDK and draw to them?

It is possible to create off-screen image buffers in After Effects SDK. You can use intermediate buffers by working with the output buffer handed to the effect. The buffer has a 'data' pointer for the base address of image pixels and a 'rowbytes' variable for the number of bytes per row (stride). You can draw to custom buffers and then copy the result into the output buffer. After Effects provides native buffers called 'worlds' in the SDK documentation that can be used as intermediates.

*Tags: `sdk`, `memory`, `render-loop`, `output-rect`*

---

## How do I access input from different layers and at different times in an After Effects plugin?

Inputs from different layers and at different times can be accessed using layer parameters. Every effect has a default layer param (index 0) used to checkout the input layer. To get images from other times, look up 'WIDE_TIME' in the After Effects SDK documentation. Both the effect's own layer and other layers are accessed via layer parameters.

*Tags: `sdk`, `params`, `layer-checkout`*

---

## How can I get the After Effects version string (e.g., 'After Effects CC 2017') in my plugin?

There is no direct SDK function to retrieve the version string. Instead, use PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH_W, wPath) to get the path to your plugin files, then decipher the AE version from the file path. Alternatively, look for the 'Presets' folder which is always in the same directory as AfterFX.exe. You can also check the Windows registry at HKEY_LOCAL_MACHINE\SOFTWARE\Adobe\After Effects\ or the macOS library at /Library/Preferences/com.Adobe.After Effects.paths.plist.

```cpp
A_UTF16Char wPath[AEFX_MAX_PATH];
PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH_W, wPath);
```

*Tags: `aegp`, `pipl`, `windows`, `macos`, `sdk`*

---

## How can I get camera position in world coordinates when the camera is parented to another layer?

Use AEGP_GetEffectCameraMatrix() to get the camera matrix post all parenting transformations. You can then extract the camera position from that matrix.

```cpp
ERR(suites.StreamSuite4()->AEGP_GetLayerStreamValue(camera_layerH, AEGP_LayerStream_ANCHORPOINT, AEGP_LTimeMode_CompTime, &comp_timeT, FALSE, &stream_anchor, NULL));
```

*Tags: `aegp`, `camera`, `3d`, `matrix`, `sdk`*

---

## How can I detect if a layer is a shape layer using the AEGP?

Use the function AEGP_GetLayerObjectType() and check if the result equals AEGP_ObjectType_VECTOR to determine if a layer is a shape layer.

```cpp
AEGP_GetLayerObjectType()
// Returns AEGP_ObjectType_VECTOR for shape layers
```

*Tags: `aegp`, `shape`, `layer-checkout`, `sdk`*

---

## What do the origin_x and origin_y fields in PF_EffectWorld represent when checking out a shape layer?

The origin_x and origin_y fields in the PF_EffectWorld structure represent the offset position of the checked-out shape within its buffer. When checking out a shape layer, the resulting buffer is the size of the shape itself (not the layer), and the origin fields indicate where that shape is positioned relative to the layer's coordinate space.

*Tags: `layer-checkout`, `shape`, `output-rect`, `sdk`*

---

## How can you collapse a parameter in the After Effects timeline using the SDK?

An alternative way to collapse a parameter is to use AEGP_SetStreamFlags() to hide and then unhide the parameter, which may collapse the parameter in the timeline as well. This approach can be used in addition to PF_ParamFlag_START_COLLAPSED and PF_ParamFlag_COLLAPSE_TWIRLY, which work for collapsing twirlies in the Effect Control Window (ECW).

*Tags: `sdk`, `params`, `ui`, `aegp`*

---

## How can I group multiple plugin operations into a single undo step in After Effects?

Use StartUndoGroup and EndUndoGroup to bunch multiple operations together into one undo step. This prevents the undo stack from appearing limited when multiple actions are performed.

*Tags: `aegp`, `undo`, `sdk`*

---

## Why are undo groups not working when called from PF_Cmd_USER_CHANGED_PARAM?

There are two common issues: (1) When creating the action string, ensure it is properly null-terminated by using string literal syntax like `const A_char action[] = "Text";` instead of character array syntax. (2) AEGP_StartUndoGroup() cannot accept NULL as a parameter in newer After Effects versions—you must pass an empty string (two consecutive quotes "") instead. Also note that undo groups may only work properly with AEGP-style callbacks, and you should verify that parameter value changes are working in your context.

```cpp
const A_char action[] = "Text";
suites.UtilitySuite5()->AEGP_StartUndoGroup(action);
params[TEXTBOX_GRAD_OFFSETSTARTX]->u.fs_d.value = 50;
params[TEXTBOX_GRAD_OFFSETSTARTX]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
suites.UtilitySuite5()->AEGP_EndUndoGroup();
```

*Tags: `params`, `undo`, `aegp`, `sdk`*

---

## Is plugin_id always necessary when calling AEGP functions?

For the most part, you can pass NULL instead of the plugin_id to AEGP functions and they will work. However, there is one important exception: when registering command hooks, you should actually use the plugin_id, otherwise you might not get the hook/events you requested.

*Tags: `aegp`, `sdk`, `reference`*

---

## Why does an angle parameter show weird fixed-point values instead of the expected float value?

Angle parameters in After Effects use fixed-point representation internally. To convert a fixed-point angle value to float, use the FIX_2_FLOAT(X) macro. Alternatively, use the PF_AngleParamSuite1::PF_GetFloatingPointValueFromAngleDef() function from the SDK, which handles the conversion automatically.

```cpp
FIX_2_FLOAT(paramdef.u.ad)
```

*Tags: `params`, `sdk`, `aegp`, `reference`*

---

## What does the PF_ prefix stand for in the After Effects SDK?

PF stands for 'Plug-in Filter', as opposed to AEGP which stands for 'After Effects General Plug-in'. The PF_ prefix is used throughout the SDK for parameters and classes related to filter plugins.

*Tags: `pipl`, `aegp`, `sdk`, `reference`*

---

## What is the correct way to change a parameter value during PF_Event_IDLE?

According to Adobe SDK documentation, parameter values can only be modified during PF_Cmd_USER_CHANGED_PARAM and specific PF_Cmd_EVENT types (PF_Event_DO_CLICK, PF_Event_DRAG, PF_Event_KEYDOWN). To change parameters outside these events, use AEGP_SetStreamValue() from the AEGP suite. If the parameter has keyframes, use the keyframe suite instead, though AEGP_SetStreamValue() will still work despite what the documentation states. AEGP_SetStreamValue() is available in all After Effects versions since 2000.

```cpp
params[someindex]->u.fs_d.value = somevalue;
params[someindex]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `sdk`, `idle-event`*

---

## How can you detect which mouse button (left, middle, or right) was pressed in an After Effects plugin?

You cannot rely on the AeData structure to determine which mouse button was pressed. Instead, you must use the operating system API. For Windows, use the GetKeyState() function with virtual key codes: VK_LBUTTON for left button, VK_MBUTTON for middle button, and VK_RBUTTON for right button. Check if the result masked with 0x100 is non-zero to determine if the button is currently pressed.

```cpp
case button::left: return (GetKeyState(VK_LBUTTON) & 0x100) != 0;
case button::middle: return (GetKeyState(VK_MBUTTON) & 0x100) != 0;
case button::right: return (GetKeyState(VK_RBUTTON) & 0x100) != 0;
```

*Tags: `ui`, `windows`, `sdk`, `aegp`*

---

## How can I make PF_Param_POINT and PF_Param_POINT_3D parameters maintain constant values independent of layer resizing?

PF_Param_POINT and PF_Param_POINT_3D parameters by default scale proportionally with layer dimensions. To maintain absolute coordinates invariant under layer size changes, shachar carmi recommends three workarounds: (1) Create a custom parameter with custom composition UI to have full control over behavior; (2) Add an expression to your point parameter programmatically that divides the current position by layer dimensions and multiplies by a scaling factor to maintain the same absolute coordinates when layer dimensions change; (3) Try the PF_ValueDisplayFlag_PIXEL flag, though its applicability to point parameters is undocumented and may have unexpected behavior.

*Tags: `params`, `ui`, `sdk`*

---

## What is a good reference sample project for implementing 3D matrix transformations in After Effects plugins?

The "Artie" sample project included in the After Effects SDK contains macros for getting XY coordinates of a texture using a 4x4 matrix. This sample is useful as a proof of concept for implementing 3D transform functions when antialiasing is not a critical concern.

*Tags: `reference`, `3d`, `matrix`, `sample`, `sdk`, `open-source`*

---

## Why does applying a buffer-expanding effect like blur after a SmartFX effect cause an offset in the output?

When using SmartFX to expand the output buffer to the full composition size, subsequent buffer-expanding effects (such as blur or drop shadow) can alter the output origins. To fix this, you need to explicitly set the output->origin_x and output->origin_y parameters in your SmartFX preRender function. This ensures that downstream effects don't inadvertently offset your effect's output.

```cpp
UnionLRect(&req.rect, &extra->output->result_rect);
UnionLRect(&req.rect, &extra->output->max_result_rect);
// Also set:
extra->output->origin_x = /* appropriate value */;
extra->output->origin_y = /* appropriate value */;
```

*Tags: `smartfx`, `buffer`, `output-rect`, `sdk`*

---

## How can I group parameters into folders in an After Effects plugin?

Parameters can be grouped into folders (commonly referred to as "groups") by defining them between PF_ADD_TOPIC and PF_END_TOPIC macros. Topics can also be nested to create hierarchical parameter organization.

*Tags: `params`, `ui`, `pipl`, `sdk`*

---

## How can I force a rerender after PF_Cmd_SEQUENCE_RESETUP or PF_Cmd_SEQUENCE_SETUP when effect state changes?

Use GuidMixInPtr to incorporate custom state flags (such as a watermarked or license status flag) into the state that After Effects uses to determine whether a cached frame is valid. When you push the relevant flag into the mix, After Effects will automatically trigger a re-render when that state changes. This feature is documented in the CC2015 SDK and later versions.

*Tags: `sequence-data`, `caching`, `compute-cache`, `sdk`, `render-loop`*

---

## What sample projects and tutorials are available for learning After Effects plugin development?

There is a significant shortage of sample projects and tutorials for AE plugin development. Available resources include: (1) A very old MacTech tutorial on After Effects Plugins at http://www.mactech.com/articles/mactech/Vol.15/15.09/AfterEffectsPlugins/index.html, and (2) A paid course at https://www.fxphd.com/details/526/. Beyond these, the Adobe community forums contain extensive information ranging from basics to advanced topics. The SDK itself includes sample plugin codes such as Transformer and Skeleton that can be studied.

*Tags: `reference`, `tutorial`, `open-source`, `sdk`*

---

## Is there a reference in the SDK documentation about bundling AEGP plugins with effect plugins?

According to the SDK guide, if you want to add an AEGP plug-in to the same binary as an effect plugin, the PiPL of the AEGP must be the first one in the resource list. This allows you to avoid creating a separate installer while still gaining access to AEGP features like idle hooks for proper effect synchronization.

*Tags: `aegp`, `pipl`, `sdk`, `plugin-architecture`, `reference`*

---

## How can I trigger an event or function when the playhead crosses a marker in After Effects?

There is no direct marker event in the After Effects SDK. Instead, you can use an idle hook to monitor the composition's current time and detect marker crossings by comparing the previous time with the current time. Alternatively, you can listen to time change events and perform the same check. You may also investigate AEGP_RegisterCommandHook to see if a command is triggered on marker events.

*Tags: `aegp`, `sdk`, `reference`, `marker-detection`*

---

## How can I add markers to a layer from an effect panel in After Effects?

The user reported finding a solution to add markers to layers from an effect panel, though the specific implementation details were not fully shared in this conversation. The discussion suggests using the After Effects SDK to manipulate layer markers programmatically from custom UI panels.

*Tags: `sdk`, `ui`, `params`, `aegp`*

---

## How do I access and manipulate individual RGB pixel values in an After Effects plugin?

To access individual pixels and their RGB values, examine the 'shifter' sample project included in the After Effects SDK. This sample demonstrates how to read and modify individual pixel data, which is fundamental for any plugin that needs to perform per-pixel color operations.

*Tags: `sdk`, `reference`, `pixels`, `color`, `sample`*

---

## How do I access pixel data from different frames in an After Effects plugin?

The 'checkout' sample project in the After Effects SDK demonstrates how to retrieve images from points in time other than the currently processed frame. This is useful for plugins that need to reference previous or future frames, such as temporal effects or frame-blending operations.

*Tags: `sdk`, `reference`, `layer-checkout`, `frame-access`, `sample`*

---

## How can you integrate Cairo graphics library with the After Effects SDK for 2D drawing?

To integrate Cairo with the AE SDK, you need to draw onto a native Cairo buffer, and when you're done, convert it pixel by pixel into an AE buffer. This approach is simpler than OpenGL for modest 2D graphics tasks, as you don't need to manage framebuffer objects and contexts. Instead of using schemes like sizeof(GL_RGBA) for buffer calculations, you perform a pixel-by-pixel conversion from the Cairo buffer to the AE output buffer.

*Tags: `sdk`, `graphics`, `cairo`, `buffer`, `2d-rendering`*

---

## How do you set a Layer Parameter value from within the After Effects SDK?

Use AEGP_SetStreamValue() to set layer parameter values. First, get the effect layer and its parent composition using AEGP_GetEffectLayer() and AEGP_GetLayerParentComp(). Create the desired layer and get its ID with AEGP_GetLayerID(). Then retrieve the effect stream by index using AEGP_GetNewEffectStreamByIndex(), get a new stream value with AEGP_GetNewStreamValue(), set the layer_id field in the AEGP_StreamValue2 structure to your new layer's ID, and finally call AEGP_SetStreamValue() to apply the change. Remember to dispose of the stream value and stream when done.

```cpp
AEGP_StreamValue2 layerParamStreamVal;
ERR(suites.StreamSuite4()->AEGP_GetNewEffectStreamByIndex(aegp_plugin_id, effectRefH, PARAM_CONTENTLAYER_1_ID, &streamRefH));
ERR(suites.StreamSuite4()->AEGP_GetNewStreamValue(aegp_plugin_id, streamRefH, AEGP_LTimeMode_LayerTime, &zeroTime, TRUE, &layerParamStreamVal));
layerParamStreamVal.val.layer_id = newLayerID1;
ERR(suites.StreamSuite4()->AEGP_SetStreamValue(aegp_plugin_id, streamRefH, &layerParamStreamVal));
ERR(suites.StreamSuite4()->AEGP_DisposeStreamValue(&layerParamStreamVal));
ERR(suites.StreamSuite4()->AEGP_DisposeStream(streamRefH));
```

*Tags: `aegp`, `params`, `layer-checkout`, `sdk`*

---

## How can I retrieve keyframe values and interpolation data from After Effects parameters in a plugin?

Use the AEGP_GetNewStreamValue() function to get parameter values at any composition time. This function allows you to retrieve interpolated values based on keyframes and their temporal interpolation settings (such as Bezier). It is available in several SDK sample projects and will return the calculated value at any given frame, taking into account all keyframe interpolation between start and end frames.

*Tags: `aegp`, `params`, `reference`, `sdk`*

---

## How do you change the plugin name and menu in an After Effects SDK sample plugin?

The plug-in name and menu are set in the resource file (namePiPL.r). After you change it, you'll need to clean the build and re-build the project, otherwise the new settings will not apply in the compiled plug-in.

*Tags: `pipl`, `build`, `sdk`, `resource`*

---

## How can an After Effects plugin programmatically create keyframes on its own parameters when a button is pressed?

During the USER_CHANGED_PARAM event for the button parameter, use AEGP_GetNewEffectForEffect to get the effect reference, then use AEGP_GetNewEffectStreamByIndex with the target parameter index to get the parameter stream. Once you have the stream reference, you can use keyframe manipulation functions to set keyframes as needed.

*Tags: `params`, `aegp`, `keyframes`, `sdk`*

---

## How do you set up After Effects plugin development with the SDK and compile it to .aex format?

To develop an After Effects plugin: 1) Download the SDK (e.g., CC 2015 or later). 2) Ensure you have the required Visual Studio C++ compiler installed (the SDK specifies which version is needed for your AE version). 3) Open one of the example projects from the SDK (such as Convolutrix, which is a good beginner example). 4) In Visual Studio project settings, configure the output directory to point to After Effects' plugin directory or a folder with a shortcut to it. This allows you to build directly into a folder AE reads and debug your plugin. 5) Build the project, which will generate the .aex file. 6) The compiled .aex plugin will then be readable by After Effects. Note that plugin development does not use the Object Model like ExtendScript does—it is C/C++ based SDK development.

*Tags: `aex`, `sdk`, `build`, `visual-studio`, `plugin-development`*

---

## What is a good example project to start learning After Effects plugin development?

The Convolutrix example project included in the After Effects SDK is recommended as a good beginner project to start learning plugin development. It demonstrates core concepts and can be compiled in Visual Studio to produce a working .aex plugin file.

*Tags: `sdk`, `reference`, `example`, `beginner`*

---

## How can you update a layer's mask using the After Effects SDK?

Use the AEGP_MaskSuite and AEGP_MaskOutlineSuite to update layer masks. Note that these suites require copying mask vertices individually. For rare operations or simpler mask copying between layers, consider using AEGP_ExecuteScript() with JavaScript, which may offer a more straightforward solution.

*Tags: `aegp`, `mask`, `sdk`, `layer-checkout`*

---

## What is the simplest way to copy a full mask from one layer to another in After Effects SDK?

While AEGP_MaskSuite and AEGP_MaskOutlineSuite allow mask manipulation, they require vertex-by-vertex copying. For a simpler approach to full mask copying, use AEGP_ExecuteScript() to execute JavaScript code, which typically offers more straightforward mask operations than the C SDK.

*Tags: `aegp`, `mask`, `scripting`, `sdk`*

---

## Why does my PF_Cmd_DO_DIALOG case handler run constantly instead of only when the dialog button is clicked?

The issue is likely a missing break statement on the case above PF_Cmd_DO_DIALOG. Without a break, execution falls through to the dialog case, causing it to run repeatedly. Ensure each case statement in your switch block has a proper break statement to prevent fall-through execution.

```cpp
case PF_Cmd_DO_DIALOG:
  err = Register(in_data, out_data);
  break;
```

*Tags: `ui`, `debugging`, `sdk`*

---

## How do you query a color picker parameter and get its floating-point values instead of 16-bit 0-255 range?

Use the PF_GetFloatingPointColorFromColorDef function from the ColorParamSuite1. Create a PF_PixelFloat variable and call suites.ColorParamSuite1()->PF_GetFloatingPointColorFromColorDef(in_data->effect_ref, params[COLOR_PICKER_INDEX], &fp_colorP) to convert the color parameter to floating-point representation. This allows you to work with color values in full floating-point precision rather than the default 16-bit packed color format.

```cpp
PF_PixelFloat fp_colorP;
suites.ColorParamSuite1()->PF_GetFloatingPointColorFromColorDef(
    in_data->effect_ref,
    params[MATTE_PICKER],
    &fp_colorP);
```

*Tags: `params`, `ui`, `sdk`, `reference`*

---

## Where can I find the source code for AEGP_GetNewStreamValue() and how to work with streams in After Effects plugins?

The After Effects SDK documentation and sample projects are available from Adobe's developer portal. The SDK Guide PDF and sample projects like ProjDumper, Streamie, and Mangler provide good examples of how streams are implemented. You can download the latest SDKs from http://www.adobe.com/devnet/aftereffects.html, and each SDK comes with an SDK guide in PDF format. For CS6 specifically, the download links are available at http://www.adobe.com/devnet/aftereffects/sdk/cs6_eula.html

*Tags: `aegp`, `reference`, `sdk`, `streams`*

---

## How do you set up debugging for an After Effects SDK example project in Visual Studio?

In the project properties window, set the command for debugging by providing the path to the afterfx.exe file. You can ignore any symbols-related warnings that appear.

*Tags: `debugging`, `build`, `windows`, `sdk`*

---

## Where can I find the official After Effects SDK and scripting documentation?

The official After Effects SDK, scripting guide, and C API documentation can be found on Adobe's developer network at http://www.adobe.com/devnet/aftereffects.html. The scripting guide walks through JavaScript scripting, and the SDK file contains the C API docs for plug-in development.

*Tags: `reference`, `scripting`, `sdk`*

---

## Can I set a higher priority for my custom AEIO to override Adobe's built-in handler for a format like FLV?

No, the After Effects SDK does not support overriding the priority of built-in AEIOs. After Effects will ignore custom AEIOs that associate to formats it already handles natively. The recommended workaround is to ask users to change the file extension to a different one (e.g., .flv to .custom_flv), and optionally create an import menu entry to automate this extension change for better user experience.

*Tags: `aeio`, `sdk`, `aegp`, `deployment`*

---

## How can I get a list of the selected layers when my plugin is activated?

You can check the currently selected layers using the AEGP function AEGP_GetNewCollectionFromCompSelection(). This allows you to retrieve the user's current layer selection without modifying the scene.

*Tags: `aegp`, `layer-checkout`, `sdk`*

---

## How can I detect if an effect instance is already applied to a layer to prevent duplicate applications?

To detect if another instance of your effect is present on a layer, you need to scan the layer's effects using AEGP_GetLayerNumEffects and AEGP_GetLayerEffectByIndex, then check each effect's installed key using AEGP_GetInstalledKeyFromLayerEffect and match it to your own. If an effect is already present in a saved effect, you'll receive a SequenceResetup() or Unflatten() call instead of SequenceSetup(). Note that an effect cannot directly remove another effect from the same layer; you can either use a separate AEGP with an idle hook for deletion, or set a flag on the second instance to disable its processing.

*Tags: `aegp`, `params`, `sdk`*

---

## How do you get the correct coordinate in layer coordinate system when using the iterate function?

When a layer is partially off the comp, After Effects may give you a reduced buffer. The effectWorld structure shows you the offset values needed to correctly map pixel coordinates. When iterating, you need to account for this offset by using outPixel=GetColor(x+offsetX,y+offsetY). Additionally, there is an "iterate offset function" available in the SDK that can help handle this automatically. The input and output buffers for non-collapsed layers are in layer coordinates, but AE diminishes the buffer at 20% increments based on what part of the layer is out of sight, not just individual pixels.

```cpp
outPixel=GetColor(x+offsetX,y+offsetY)
```

*Tags: `layer-checkout`, `iterate`, `output-rect`, `sdk`*

---

## Why does ProjDumper throw an 'unexpected match name searched for in group' error when trying to get a layer's audio stream?

The error occurs when attempting to get an audio stream from a layer that doesn't have audio. Before calling AEGP_GetNewLayerStream with AEGP_LayerStream_AUDIO, you should first check if the layer's source item has audio by calling AEGP_GetLayerSourceItem followed by AEGP_GetItemFlags with AEGP_ItemFlag_HAS_AUDIO. Note that some layers (cameras, lights, text layers, and vector shapes) don't have a source item at all.

```cpp
AEGP_ItemH itemH(NULL);
ERR(suites.LayerSuite8()->AEGP_GetLayerSourceItem(layerH, &itemH));
if (itemH != NULL) {
  A_long flags;
  ERR(suites.ItemSuite9()->AEGP_GetItemFlags(itemH, &flags));
  if (flags & AEGP_ItemFlag_HAS_AUDIO) {
    AEGP_StreamH streamH(NULL);
    ERR(suites.StreamSuite2()->AEGP_GetNewLayerStream(S_my_id, layerH, AEGP_LayerStream_AUDIO, &streamH));
  }
}
```

*Tags: `aegp`, `debugging`, `sdk`, `reference`*

---

## Why does AEGP_GetLayerSourceItem return NULL even with a valid composition handle?

Some layers don't have a source item, including cameras, lights, text layers, and vector shapes. Before using AEGP_GetLayerSourceItem, verify that the layer is one that can have a source item. If itemH is NULL after the call, the layer type doesn't support source items.

*Tags: `aegp`, `layer-checkout`, `sdk`*

---

## What is the difference between using ERR() macro and direct error assignment in AEGP plugin code?

The ERR() macro checks if an error has already occurred (err != NULL) and skips the function call if one has. This prevents cascading errors but also skips subsequent operations. Direct assignment (err = doStuff()) executes the function regardless of prior errors. ERR() is the preferred technique as it provides better error handling flow control, but you must manually reset err = NULL if you want to continue execution after an error.

```cpp
ERR(suites.SomeFunction());
// vs
err = suites.SomeFunction();
```

*Tags: `aegp`, `debugging`, `sdk`*

---

## How do you set a parameter group to default to open (expanded) instead of collapsed?

Use the PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG flag. To selectively control which groups are open or closed, set the PF_ParamFlag_COLLAPSE_TWIRLY flag in the def.flags structure when calling PF_ADD_TOPIC(). For example: AEFX_CLR_STRUCT(def); def.flags = PF_ParamFlag_COLLAPSE_TWIRLY; PF_ADD_TOPIC("name", GROUP_DISK_ID);

```cpp
AEFX_CLR_STRUCT(def);
def.flags = PF_ParamFlag_COLLAPSE_TWIRLY;
PF_ADD_TOPIC("name", GROUP_DISK_ID);
```

*Tags: `params`, `ui`, `sdk`*

---

## How can you hide all shy layers in a composition when AEGP_SetCompFlags is not available in the SDK?

While AEGP_SetCompFlags() does not exist in the SDK, you can use AEGP_ExecuteScript() to execute JavaScript code that accomplishes this. The JavaScript code to hide shy layers in the active composition is: app.project.activeItem.hideShyLayers = true;

```cpp
app.project.activeItem.hideShyLayers = true;
```

*Tags: `aegp`, `scripting`, `sdk`*

---

## What is the recommended approach for accessing text layer data from C/C++ plugins?

Text layers are a blind spot in the After Effects SDK (at least through CS6). The recommended approach is to use AEGP_ExecuteScript() to pull text data using the JavaScript API, as the JavaScript API has better support for text properties than the C SDK. The script can be passed directly as a char pointer without requiring an external script file. See http://forums.adobe.com/message/3625857#3625857 for implementation details on launching AEGP_ExecuteScript() and retrieving results back to the C side.

*Tags: `aegp`, `scripting`, `sdk`, `reference`*

---

## What does the 'invalid filter 25::3' error mean when loading SDK examples in After Effects?

The 'invalid filter 25::3' error typically indicates a dependency issue with the compiled plugin. This often occurs when the Visual Studio compiler version or runtime libraries don't match what After Effects expects. For CS5 and above, plugins must be compiled as x64 (64-bit), not x86 (32-bit), as After Effects no longer loads x86 plugins. Use Dependency Walker to verify all dependencies are present and correct. Additionally, ensure you have the correct Visual C++ runtime libraries installed, such as MSVCR100D.DLL from the Microsoft Visual C++ Redistributable package.

*Tags: `sdk`, `build`, `windows`, `debugging`, `pipl`*

---

## How do I extract path data from a stream in After Effects using the SDK?

To get path data, first obtain the stream reference for the path. Then use StreamSuite2()->AEGP_GetNewStreamValue() to retrieve the stream value at a specific time, casting it to PF_PathOutlinePtr. Once you have the path outline pointer, use PathDataSuite1() functions like PF_PathPrepareSegLength() and PF_PathGetSegLength() to query segment data. The NULL parameter for PF_ProgPtr can be passed as NULL in this context.

```cpp
suites.StreamSuite2()->AEGP_GetNewStreamValue(NULL, pathStreamH, AEGP_LTimeMode_CompTime, &time, TRUE, &value);
suites.PathDataSuite1()->PF_PathPrepareSegLength(NULL, (PF_PathOutlinePtr)value.val.mask, index, freq, &pathSegment);
suites.PathDataSuite1()->PF_PathGetSegLength(NULL, (PF_PathOutlinePtr)value.val.mask, index, &pathSegment, &segLength);
```

*Tags: `aegp`, `sequence-data`, `sdk`, `reference`*

---

## What AEGP SDK function can be used to apply a preset to a layer from a plugin?

There is no native C API method for applying an effect preset in the AEGP SDK. The recommended approach is to use AEGP_ExecuteScript to execute the Layer.applyPreset() JavaScript method instead.

*Tags: `aegp`, `scripting`, `sdk`, `reference`*

---

## How can I make an After Effects plugin compiled with CS6 SDK work with CS5 and 5.5?

You need to do more than just swap headers—you must also update the API version in the .r (PiPL) file. The recommended approach is to copy your code to a working CS5 project and recompile with the CS5 SDK. While Adobe states that suites are never removed or altered, rare deprecations have occurred (e.g., in CS6). If you need to support multiple versions, use preprocessor definitions to conditionally select the correct suite version for each target SDK version.

```cpp
#define SOME_SUITE someSuiteVer6  // for CS4
#define SOME_SUITE someSuiteVer7  // for CS5

// In code:
suites.SOME_SUITE()->someFunction("yo yo!");
```

*Tags: `sdk`, `cross-version`, `suites`, `pipl`, `build`*

---

## How can I set a slider parameter to match the height or width of the layer the plugin is applied to?

PF_Cmd_PARAMS_SETUP occurs before the plugin is applied to the layer, so it cannot read layer parameters or even identify which layer it's on. Instead, use the PF_Cmd_UPDATE_PARAMS_UI callback that occurs afterwards to set parameter values. While the documentation recommends only cosmetic changes during this phase, you can use the Stream Suite to update parameter values programmatically rather than direct parameter array access.

*Tags: `params`, `aegp`, `effect_api`, `sdk`*

---

## Why does the PiPL resource compiler fail with LNK1104 error when building an After Effects plugin in Visual Studio 2008?

The PiPL resource compiler and linker fail when the After Effects SDK is installed in a non-standard location. The SDK should be placed in the standard path (e.g., C:\Program Files\Adobe\After Effects CS6 SDK\) rather than in custom user directories. If you must use a different location, you can configure the custom build step to point to the tool in another location, but this requires additional configuration effort.

*Tags: `build`, `pipl`, `windows`, `visual-studio`, `sdk`*

---

## How do you open and read files in an AEIO plugin using the SDK API?

The After Effects SDK does not provide built-in file I/O functions. Instead, you use standard C library functions like fopen, fread, and fclose to open and read files. When you receive a file path as A_UTF16Char in functions like VerifyFileImportable(), you can use it directly with these standard functions.

*Tags: `aeio`, `import`, `sdk`, `aegp`*

---

## How do you convert A_UTF16Char strings to 8-bit char strings in After Effects plugins?

You can convert A_UTF16Char (UTF-16) to char (8-bit) by iterating through the UTF-16 string and copying each character to a buffer. Create a char buffer large enough to hold the converted string, then use a while loop to copy characters from the UTF-16 string pointer to the char buffer pointer, incrementing both pointers. Ensure the buffer is null-terminated and large enough to avoid overflow.

```cpp
char buffer8[1024] = {'\0'};
char *char8 = buffer8;
A_UTF16Char *char16 = thatUTF16String;
while(*char16) {
  *char8++ = *char16++;
}
*char8 = NULL;
```

*Tags: `aeio`, `sdk`, `cross-platform`*

---

## What are good sample projects to learn how to pull and push pixel values in After Effects plugins?

The Adobe After Effects SDK includes two key sample projects: the Shifter sample demonstrates how to iterate through output samples and pull input values from arbitrary locations (useful for coordinate transformation effects), and the CCU sample project shows how to access buffer pixels directly in RAM for pushing values to specific pixel locations.

*Tags: `skeleton`, `reference`, `open-source`, `sdk`*

---

## Is there a direct C++ API method to replace a layer's source in After Effects?

There is no direct C++ AEGP API method for replacing a layer's source. The functionality exists in the Java API as layer.source.replace(), but in C++ it must be accessed indirectly using DoCommand with command ID 2299. However, this indirect method requires making the composition the frontmost window, which is difficult to achieve from an AEGP plugin. The feature simulates alt+dragging a new source onto a selected layer in the composition window.

*Tags: `aegp`, `api`, `layer`, `sdk`*

---

## What are the best approaches for accessing and sampling specific pixels from a buffer in After Effects?

There are two main approaches: (1) Use the Sample Suite to grab color values from a pixel, subpixel, or area within a buffer - see the Shifter sample; (2) Access pixel data directly in RAM and retrieve the data yourself - see the CCU sample. Choose based on whether you need convenience (Sample Suite) or direct memory access performance.

*Tags: `memory`, `layer-checkout`, `sdk`, `reference`*

---

## How do I use multiple sliders and pass their values to the Render function in After Effects plugins?

Use the refcon (a void pointer that points to a structure or class) to pass multiple slider values to the iterate() function. Instead of creating separate iterate() calls for each slider, consolidate your processing into a single iteration call by storing all slider values in a single refcon structure. This reduces overhead since each iteration function call has its own CPU cost. The refcon is flexible—you can add as many separate values as needed to it, and you're not required to treat the input pixel as input and output pixel as output in a specific way.

```cpp
suites.iterate8suite()->iterate(
  lines_so_far,
  total_lines,
  input_world,
  &output->extent_hint,
  refcon,  // Pass structure containing all slider values here
  ProcessFunction,
  output_world);
```

*Tags: `params`, `ui`, `render-loop`, `sdk`*

---

## What is the correct way to create custom hidden streams and store data on layers using AEGP?

There are limited options for storing custom data on layers with AEGP. Dynamic streams (like strokes from the paint tool and text animators) exist but are very specific. For persistent data across sessions, the recommended approach is to keep the data within the AEGP itself rather than on the layer, storing each layer's ID and its comp's project item ID to identify and retrieve the associated data. For temporary data that can be regenerated, storing it in the AEGP is also preferred. Alternative approaches include using effect parameters, layer comments, or layer marker comments, though these have limitations with the SDK.

*Tags: `aegp`, `arb-data`, `layer-checkout`, `sdk`*

---

## How can I determine the bit depth and pixel format of input/output images in an After Effects plugin?

The easiest way to get the bit depth is using `extra->input->bitdepth` in the smart_render call, which returns 8, 16, or 32. Alternatively, use the PF_WorldSuite2 to call `PF_GetPixelFormat()` on the world. After Effects always uses ARGB pixel format. The pixel format enum returns values like PF_PixelFormat_ARGB128 or PF_PixelFormat_ARGB.

```cpp
PF_WorldSuite2 *wsP = NULL;
ERR(suites.Pica()->AcquireSuite(kPFWorldSuite, kPFWorldSuiteVersion2, (const void**)&wsP));
ERR(wsP->PF_GetPixelFormat(inputP, pixelFormat));
ERR(suites.Pica()->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2));
```

*Tags: `params`, `mfr`, `smart-render`, `sdk`, `reference`*

---

## How can I make my After Effects plugin support multiple bit depths instead of just 8-bit?

You cannot explicitly restrict AE to only process at specific bit depths through plugin settings. Instead, check the bitdepth at runtime using `extra->input->bitdepth`, and if the bitdepth is not supported, copy the input to output without processing and inform the user via a message overlay or alert that the effect requires a specific bit depth.

*Tags: `params`, `smart-render`, `ui`, `sdk`*

---

## What are good SDK samples to learn about custom suites in After Effects?

The 'sweetie' sample demonstrates how to create a custom suite for inter-AEGP communication. The 'checkout' sample shows how to use a custom suite, particularly in the globalSetup() function. These samples are included in the After Effects SDK and serve as the primary documentation for custom suite implementation.

*Tags: `aegp`, `open-source`, `reference`, `sdk`*

---

## Why does the After Effects SDK build succeed but produce no .dll or .aex file?

By default in the CS4 SDK, plugins build to c:\program files\adobe\after effect 9.0\plug-ins\sdk (note "after effect 9.0" instead of "CS4"). If you don't see output in your expected directory, check the Visual Studio Linker options to verify the output directory is set correctly, or search your computer for the .aex file as it may be building to an unexpected location.

*Tags: `build`, `sdk`, `after_effects_sdk`, `visual_studio`, `deployment`*

---

## How do you fix a Skeleton project that won't build with a web deployment error?

If the Skeleton project shows a conversion warning about "Web deployment to the local IIS server is no longer supported" and won't build, this error disables your build settings. The simplest solution is to duplicate a working project from the SDK and replace only the .cpp and .h files with the Skeleton files, rather than trying to manually fix the corrupted build settings.

*Tags: `build`, `sdk`, `skeleton`, `visual_studio`, `after_effects_sdk`*

---

## Can Visual C++ Express be used to compile 64-bit After Effects plugins?

Yes, Visual C++ Express can be used to compile 64-bit plugins by downloading the Windows SDK and using it as the compiler. A patch is available that enables 64-bit compilation support in VC2008 Express with Windows 7 SDK.

*Tags: `build`, `windows`, `sdk`, `64-bit`*

---

## Why can't VC++ 2008/2010 Express open After Effects SDK example projects?

The After Effects CS5 SDK example projects require 64-bit compiler configurations, which are not checked by default in the Visual Studio installer. You need to install the 64-bit compiling options during Visual Studio setup. Additionally, ensure the Windows SDK is installed and that Visual Studio is configured to look in the correct SDK paths. The Premiere Pro CS5 SDK works because it includes both 32-bit and 64-bit configurations to support 32-bit applications like Encore CS5.

*Tags: `build`, `windows`, `sdk`, `debugging`*

---

## How can I access the composited color of layers beneath the current layer in After Effects?

There is no direct API to access intermediate composition buffers during normal effect rendering, as After Effects does not render bottom-to-top and optimizes by skipping rendering of fully opaque regions. However, several workarounds exist: (1) Write an AEGP artisan-type plugin (see the 'arti' sample) which handles composition rendering and has access to intermediate buffers, though this is very difficult; (2) Create a duplicate composition with only needed layers and render it using AEGP_GetReceiptWorld(); (3) Apply your effect on an adjustment layer, which receives the composited buffer of underlying layers, then use checked-out layer params to access original sources; (4) Use sampleImage() expressions on hidden parameters to sample pixel data, though this is slow and limited; (5) Ask the user to provide the background as a layer parameter, a common practice in effects that need background information.

*Tags: `aegp`, `render-loop`, `layer-checkout`, `sdk`, `params`*

---

## How can you create dynamic stroke parameters similar to the Paint effect in After Effects plugins?

Strokes are handled by the dynamicStreamSuite, which allows you to add, remove, and alter strokes and text layer parameters. However, custom parameters that behave like strokes cannot be created through the API. Two workarounds are: (1) Create your effect with hidden parameter groups (e.g., 50 groups, each with width, length, start, and end sliders) and unhide them as needed—this is simpler but limited by the number of compiled groups. (2) Create a dummy effect with the desired sliders and add new instances of it to the layer whenever more strokes are needed—this is more flexible and allows unlimited strokes but is significantly more complex to implement.

*Tags: `params`, `ui`, `sdk`, `aegp`, `pipl`*

---

## Can you create custom stream group types in After Effects plugins beyond existing types?

No, you can only add stream groups and atoms of existing types through the API. You cannot create custom types of your own. Any custom parameter grouping behavior must be implemented using workarounds such as hidden parameter groups or multiple plugin instances.

*Tags: `params`, `sdk`, `aegp`*

---

## How can I set an effect parameter value on a layer in an After Effects plugin?

Effect parameters in After Effects are referred to as streams. Use the AEGP_SetStreamValue() function to set parameter values. To determine which stream index corresponds to which parameter, you need to experiment with different effects and learn their stream numbers. If you're applying effects you've written yourself, you should know the parameter index numbers directly.

*Tags: `aegp`, `params`, `sdk`*

---

## How can I prevent a slider parameter from resetting to its default value when the Effect Reset button is clicked?

You cannot tell a parameter to ignore the reset operation directly. However, you can store the parameter value in sequence data and restore it when the SEQUENCE_RESETUP callback is triggered. Alternatively, if the parameter is used as a unique identifier, you can skip storing it as a parameter altogether and maintain it only in sequence data, retrieving it via AEGP_EffectCallGeneric() when needed. You could also try setting a different default value during the param_setup call, though param_setup is typically called only once when the effect is first applied.

*Tags: `params`, `sequence-data`, `aegp`, `sdk`*

---

## What are the reference projects to study for implementing custom AEGP suites and inter-plugin communication?

The Sweetie and Checkout sample projects in the After Effects SDK demonstrate custom suite implementation for effect-to-AEGP communication. ProjectDumper and Shifter are also referenced as examples of inter-plugin communication patterns. These samples show how to use custom suites (like the Duck Suite example) for effects to safely communicate with AEGPs and other plugins.

*Tags: `reference`, `open-source`, `aegp`, `sample-code`, `sdk`*

---

## Why won't my After Effects plugin built with SDK 7.0 load on Mac CS4 when it works fine on Windows?

The issue is related to architecture support differences between SDK versions. AE7 was the last version to run on PPC processors; on Intel chips it ran emulated. CS3 was the first to run native on Intel chips and SDK 8.0 (CS3) was the first SDK to support plugins running native on Intel chipsets (and PPC). The CS3 SDK offers backwards compatibility to AE7, allowing plugins to work across AE7, CS3, and CS4 on both PC and Mac. In contrast, the CS4 SDK is not backwards compatible. To achieve cross-version compatibility on Mac, you should check the resource file declarations in the CS3 SDK samples, which contain separate entry point declarations for Intel and PPC architectures.

*Tags: `macos`, `windows`, `sdk`, `build`, `cross-platform`, `pipl`*

---

## Can I draw custom graphics and keyframe indicators on the After Effects TimeGraph?

No, the After Effects SDK does not provide buffers or APIs for drawing on the TimeGraph or timeline. You can only draw into buffers that AE provides for the composition window, layer window, and effects controls window. Custom parameter UIs created in the Effects Controls Window will not appear in the timeline. As an alternative, you can create a custom UI using an arbitrary data type parameter that displays a synchronized alternative timeline view centered on the current time cursor, allowing users to edit your special keyframes with custom interpolation behavior.

*Tags: `ui`, `params`, `arb-data`, `aegp`, `sdk`*

---

## What is the best way to create interactive UI elements like draggable masks in After Effects plugins?

Use a custom UI overlay rather than rendering the mask as part of the image. Rendering the mask in the image has several drawbacks: it gets affected by subsequent effects and display channels, is confined to the layer's size, and forces re-renders when toggling visibility. A custom UI is necessary for interactivity since After Effects won't report click events otherwise. The CCU (Custom Composition UI) sample in the After Effects SDK provides an excellent starting point for creating interactive interfaces in the composition window.

*Tags: `ui`, `custom-ui`, `interactive`, `reference`, `sdk`*

---

## Where can I find example code for building custom interactive UI in After Effects composition windows?

The After Effects SDK includes a CCU (Custom Composition UI) sample that demonstrates how to create a simple interactive interface directly in the composition window. This sample is the recommended starting point for developers building custom UIs with interactive elements like draggable shapes and masks.

*Tags: `ui`, `custom-ui`, `sdk`, `reference`, `open-source`*

---

## Where can I find sample code for creating and manipulating transformation matrices in After Effects plugins?

The CCU sample included in the After Effects SDK contains helper functions for creating identity matrices and concatenating them with scale and rotation transformations. This sample demonstrates proper matrix manipulation for use with transform_world() and similar functions.

*Tags: `mfr`, `reference`, `sdk`, `sample`, `matrix`*

---
