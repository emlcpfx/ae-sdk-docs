# Q&A: scripting

**128 entries** tagged with `scripting`.

---

## Is Lua or Python a better choice for embedding a scripting language inside an After Effects plugin?

Lua is a significantly better fit for embedding inside an AE plugin. It is very easy to compile in and has a small footprint. Python is much clunkier to embed because it requires installing libraries on the user's system (asking permission to install packages), which is intrusive. While there may be cleaner ways to embed Python, Lua is far simpler to integrate. Both require similarly tedious C binding work without some kind of automation or code-generation tooling. dvb metareal shipped 'omino lua' as a script-driven drawing plugin using this approach.

*Contributors: [**dvb metareal**](../contributors/dvb-metareal/), [**rowbyte**](../contributors/rowbyte/) · Source: adobe-plugin-devs · 2023-08-16 · Tags: `lua`, `python`, `scripting`, `embedding`, `c-bindings`, `plugin-architecture`*

---

## How can a JavaScript executed from ExecuteScript open the default email client?

Use the browser to open a mailto: URL. Create a URL with the format 'mailto:email@address.com' and open it in the default browser, which will interpret the mailto: protocol and launch the default mail application.

*Tags: `scripting`, `cross-platform`*

---

## Does the Stable Diffusion plugin include the SD model or fetch it externally?

The plugin does not ship with the model. Instead, it pulls the model from Hugging Face on the first use.

*Tags: `deployment`, `scripting`*

---

## What technology is used to bridge Python with After Effects for plugin development?

A Python/AE/C++ bridge was built to enable creation of Python plugins. The bridge is stable and well-developed but still being refined for cleaner implementation.

*Tags: `scripting`, `build`, `cross-platform`*

---

## Is there a way for a CEP panel to communicate with a C++ plugin?

There is no direct communication mechanism between CEP and C++ plugins. However, indirect communication is possible through other means such as using ExecuteScript to trigger commands.

*Tags: `cep`, `scripting`, `plugin-communication`*

---

## How can a C++ plugin open a CEP panel?

A C++ plugin can open a CEP panel by using ExecuteScript to find the Command ID of the extension by name, then executing that command through the execute command suite.

*Tags: `cep`, `scripting`, `ui`*

---

## How can C++ plugin code communicate with a CEP panel?

C++ can open a CEP by executing scripting that calls the CEP. For CEP to plugin communication, store values to transfer in JavaScript memory using global variables. For large data sizes that are expensive in memory, use a temp file instead. Alternatively, set a hidden checkbox that is activated by script to catch the value. Another solution is to run a hidden panel as a server that receives requests from the plugin and adjusts values in CEP.

*Tags: `cep`, `scripting`, `params`, `memory`*

---

## How can C++ check if a CEP panel is opened?

C++ can check if a CEP is opened using the TCP protocol or by using a hidden checkbox that the CEP activates via scripting.

*Tags: `cep`, `scripting`, `debugging`*

---

## Can C++ PersistentDataSuite3 and ExtendScript app.preferences share the same settings?

The question was asked but not answered in the conversation. The user clarified they meant app.preferences rather than app.settings, but no definitive answer was provided about whether these two APIs can share the same underlying settings storage.

*Tags: `scripting`, `aegp`, `params`*

---

## Why does GuidMixInPtr not invalidate frames when using global variables or sequence data?

GuidMixInPtr appears to work only with AEGP-managed values or stack-allocated local variables. The workaround is to use an AEGP function or script to trigger frame invalidation, such as changing the composition background color via AEGP_ExecuteScript. This forces After Effects to check which frames need invalidation.

```cpp
if(extra->cb->GuidMixInPtr) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(bool), reinterpret_cast<void*>(&random_bool));
}
```

*Tags: `smart-render`, `aegp`, `scripting`*

---

## How can you trigger frame invalidation from a background worker thread without direct AEGP access?

Use AEGP_ExecuteScript from IdleHook to run an After Effects script that modifies a composition property (like background color) or a hidden parameter value. This forces After Effects to re-evaluate which frames need invalidation. Be aware that this fails if modal dialogs are open.

*Tags: `aegp`, `scripting`, `threading`, `idle-hook`*

---

## How can you pass data between a button parameter and a script execution in an After Effects plugin?

Store the script text in an arbitrary data parameter. When a button is clicked, use AEGP_ExecuteScript() to execute the script string. You can pass data by find-and-replacing values in the script string before execution, and the script can return data back to After Effects.

```cpp
AEGP_ExecuteScript()
```

*Tags: `arb-data`, `params`, `ui`, `scripting`, `aegp`*

---

## Is there a way to get the file path of the current After Effects project?

There is no direct API to get the project file path in After Effects. The question notes that this is ambiguous until the project is saved. The broader use case of associating additional user files with a project and having them participate in the Gather process is not directly supported through standard APIs.

*Tags: `aegp`, `scripting`, `deployment`*

---

## How can you get the file path of the current After Effects project?

There is no direct API to get the current project file path before it's saved. The path is ambiguous until the project has been saved at least once.

*Tags: `aegp`, `scripting`*

---

## Is there a way to associate additional user files with an After Effects project so they participate in the Gather process?

There is no built-in API to add arbitrary files to a project for inclusion in the Gather process. This would require custom workarounds outside the standard After Effects file format.

*Tags: `aegp`, `scripting`, `deployment`*

---

## Have you used nuitka for Python compilation on macOS with numpy dependencies?

One developer tried nuitka once but had limited experience with it. They preferred using Cython for compilation instead to handle their specific needs.

*Tags: `scripting`, `macos`, `deployment`*

---

## What are the size limitations for storing custom metadata in AE projects using xmpPacket from ExtendScript?

There is a practical size limit of less than 500kb for xmpPacket data. AE itself stores more than 500kb in its own XMP Packet for reference. Tools like AE Viewer can remove XMP packets to significantly reduce file sizes (e.g., from 50mb down to 50kb).

*Tags: `scripting`, `arb-data`, `memory`*

---

## Is metadata stored via xmpPacket regulated by the undo manager in After Effects?

No, XMP metadata modifications do not appear to be regulated by the undo manager and are not undoable.

*Tags: `scripting`, `arb-data`*

---

## What is a pseudoeffect and how is it used?

A pseudoeffect is a nicer alternative to using standard expression control effects like Slider Control or Point Control. It is written in XML and creates a regular-looking effect parameter UI but doesn't actually do anything—it just acts as controls that can be read from expressions, making the interface look more professional than having multiple Slider Controls.

*Tags: `params`, `ui`, `scripting`*

---

## Can mask outline modifications be done as a regular effect that procedurally modifies path vertices each frame rather than as a one-time destructive operation?

You cannot modify any project state in the render thread (like adding layers, changing layer properties, or masks). However, procedurally changing mask vertices might be possible using expressions. Baking the paths is also an effective alternative approach, though not as magical.

*Tags: `aegp`, `scripting`, `render-loop`, `params`*

---

## If mask path APIs are available in the JavaScript API, why couldn't they be used in C++ plug-ins by running the script within the plug-in?

The conversation suggests this is theoretically possible - you can run scripts within a plug-in to access JavaScript API functionality. However, this approach has limitations compared to native C++ implementation, particularly regarding per-character path separation which is still being lobbied for in the C++ plugin API.

*Tags: `scripting`, `aegp`, `params`, `debugging`*

---

## Is there a way to serialize all plugin data including arbitrary data for template/capsule creation?

For first-party plugins where you know the handle, you can serialize the data yourself. However, for third-party and Adobe stock plugins, it is very tricky. The main challenge is accessing the flatten/unflatten versions of arbitrary data which are not exposed via AEGP functions. Some plugins have serializable stack variable handles (like 'curves') while others contain pointers (like 'liquify') that cannot be reliably serialized and deserialized. Extended script investigation may be a possible avenue to explore.

*Tags: `arb-data`, `aegp`, `scripting`, `params`*

---

## How can you access the Adobe application version from an After Effects plugin?

You can use ExtendScript with 'app.version' via the aegp execute script. Alternatively, if your plugin supports both After Effects and Premiere Pro with GPU features, you can access the version through the Premiere Pro PICA Suite AppInfo, which is called before global setup. The version data is typically formatted like '24.3', and you need to add 2000 to convert it. For versions with additional components like '24.3.x', some parsing is required.

*Tags: `aegp`, `scripting`, `premiere`, `cross-platform`*

---

## What is the equivalent of Global Data for sharing state across multiple plugins?

The answer was not completed in the conversation. Alex Bizeau was asked whether an extra AEGP plugin is needed to handle cross-plugin data or if the SuiteSuite is the mechanism for this, but no definitive answer was provided.

*Tags: `aegp`, `arb-data`, `scripting`*

---

## How can you set a flag for the entire After Effects session across all plugins that resets on the next launch?

You can use the aegp suite with stream params, or implement a simpler solution using global variables in one of your plugins and dynamic symbol loading with dlopen(). This allows other plugins to dynamically load a function from that plugin to access and read the state of the shared global variable, avoiding the need for a separate companion aegp plugin.

*Tags: `aegp`, `params`, `cross-platform`, `scripting`*

---

## Is there a way to share session-wide state between plugins without creating a companion aegp plugin?

Yes, you can use dynamic symbol loading with dlopen() to load a function from one of your existing plugins. Other plugins can then call this function to get the state of a global variable defined in that plugin, eliminating the need for a separate companion plugin.

*Tags: `aegp`, `scripting`, `cross-platform`*

---

## What pattern should be used to convert traditional After Effects C API functions to Rust?

Convert functions that return PF_Err with output parameters to Rust Result types. For example, a C function like `PF_Err GetThing(Thing* output);` becomes `fn get_thing() -> Result<Thing, Error>;` in the Rust crate.

```cpp
// C API
PF_Err GetThing(Thing* output);

// Rust
fn get_thing() -> Result<Thing, Error>;
```

*Tags: `scripting`, `rust`, `error-handling`*

---

## How do you debug ExtendScript in After Effects?

On Windows, you can use the standard debugging tools. On macOS, the situation was previously difficult because you needed to use the Intel version which was very slow, but Adobe updated the debugger about a month ago to be Apple Silicon native, which is a significant quality of life improvement.

*Tags: `scripting`, `debugging`, `macos`, `apple-silicon`*

---

## What is the correct way to read kerning property in After Effects 2024.3 character range API?

The autoKernType property needs to be set to 'none' or 'undefined' in order to read the kerning property. However, there appears to be a bug where it doesn't actually read that value correctly, particularly when keyframes are involved - it often stays as 'metric' even when kerning is manually set to a high value.

*Tags: `scripting`, `params`*

---

## How can internally computed position values from a plugin render be exposed to users for use in expressions or as parameters?

The recommended approach is to output the internally computed values through a point control parameter. This allows users to access the generated positions either in expressions or by exporting them as nulls. The answer suggests this is a viable solution, with a reference to similar functionality being planned for the Vision plugin on aescripts.com.

*Tags: `params`, `ui`, `scripting`, `arb-data`*

---

## How can you set text on a text layer from an effect on a frame-by-frame basis?

You need to communicate with an AEGP plugin that can set the text of a layer for each frame, since invoking scripts via the render loop does not work for this purpose.

*Tags: `aegp`, `scripting`, `render-loop`, `text`*

---

## What is a workaround for the broken Kerning API in ExtendScript for text layers?

The Kerning API works well in expression script but is broken in ExtendScript. A workaround is to write an expression on a text layer that writes a value to a separate text layer, then read the value from that second layer back in ExtendScript.

*Tags: `scripting`, `ui`*

---

## What is a method to set source text dynamically in After Effects using rendered data?

Encode and render dead pixels in your render output, then use a sampleImage expression to retrieve those pixels which can then set the source text. This method mostly works with a few caveats and also works with Null objects.

*Tags: `scripting`, `ui`, `render-loop`*

---

## How can Plugin V2 detect when it's loaded in a project saved with V1 and default to Advanced Mode instead of Simple Mode?

Check the After Effects documentation for project compatibility and parameter versioning. The SDK provides mechanisms to detect legacy project data and conditionally set parameter visibility based on whether old parameters are present in the loaded project.

*Tags: `params`, `ui`, `scripting`*

---

## How can you keep layer parameters synced between multiple locations when users apply keyframes and expressions to one?

This is acknowledged as a difficult problem. One approach mentioned is using AEGP or a script in userchangedparam to keep them synced, but keeping them synchronized when keyframes and expressions are applied to one location and mirroring it constantly is noted as being challenging.

*Tags: `aegp`, `scripting`, `params`, `ui`*

---

## Where should After Effects SDK developers share and discover code snippets and solutions?

Stack Overflow with the 'After Effects SDK' tag is recommended as a good system for searching and hosting code snippets. GitHub can also be used for sharing code examples, though the AESDK community is niche. Creating dedicated channels or communities helps organize and grow the knowledge base.

*Tags: `open-source`, `reference`, `tool`, `deployment`, `scripting`*

---

## Why is Lua a better choice than Python for an After Effects script-driven drawing plugin?

Lua is significantly easier to compile into a plugin compared to Python. With Python, the developer had to request permission to install libraries across the system, which is intrusive. Lua, on the other hand, compiles cleanly into the plugin without requiring system-wide library installations. Both languages require similar effort for C bindings without automation or code generation tools, but Lua's compilation model is much cleaner for plugin distribution.

*Tags: `scripting`, `deployment`, `open-source`*

---

## How can you pass parameters between C and scripting languages in After Effects plugins?

Parameters can be passed as text using sprintf formatting. For example, parameters are formatted as text strings like "paramname=value" and then parsed by the scripting layer. Additionally, some bindings libraries (like PyCairo for Python or custom XML-based C binders for Lua) provide C APIs for assigning bitmap data and layer parameters directly between the compiled C code and the scripting layer.

```cpp
sprintf("%s=%s\n", paramname, value)
```

*Tags: `scripting`, `params`, `aegp`*

---

## How can you trigger parameter updates in Premiere from CEP script interactions when PF_UserChangedParam doesn't work?

When working with AE/Premiere plugins updated by CEP backend data, using a hidden parameter updated via script that calls PF_UserChangedParam works well in After Effects, but Premiere does not recognize script interactions as user actions. Alternative approaches include monitoring external file dependencies for reactivity (though this can be slow), or implementing a manual button to force refresh as a workaround.

*Tags: `premiere`, `cep`, `params`, `scripting`, `cross-platform`*

---

## How can a JavaScript plugin script open the default email client from ExecuteScript?

Use the mailto: URI scheme by opening a URL with the format 'mailto:email@address.com'. The browser will interpret the mailto: protocol and open the default mail application.

*Tags: `scripting`, `deployment`, `cross-platform`*

---

## How do you load machine learning models in an After Effects plugin without shipping them with the tool?

Models can be pulled from Hugging Face on first use rather than being included with the plugin distribution. This approach reduces initial download size and keeps models up to date.

*Tags: `deployment`, `scripting`, `aegp`*

---

## What approach can be used to create Python-based After Effects plugins?

A Python/AE/C++ bridge can be built to enable Python plugin development for After Effects. This allows developers to write plugins in Python while maintaining compatibility with AE's native plugin system, though the implementation can be complex and require several months of development to achieve stability and polish.

*Tags: `scripting`, `build`, `cross-platform`, `aegp`*

---

## Is there a way to communicate between a CEP panel and a C++ plugin in After Effects?

Direct communication between CEP and C++ plugins is not possible. However, indirect communication can be achieved: a C++ plugin can open a CEP panel by using the ExecuteScript to find the Command ID (menu) by the extension name, then executing that command through the command suite. CEP cannot directly change plugin parameter values, and the plugin cannot directly check if a CEP panel is open or bring it into focus.

*Tags: `cep`, `c++`, `scripting`, `ui`, `aegp`*

---

## How can C++ plugins communicate with CEP panels to exchange parameter values?

There are several approaches: (1) CEP can change parameter values via scripting, and C++ can open a CEP by executing a script that calls it; (2) C++ can check if CEP is open using TCP protocol or a hidden checkbox that CEP activates via scripting; (3) Use a hidden panel running as a server that receives requests from the plugin and adjusts values in CEP; (4) For CEP-to-plugin communication, store values in JavaScript memory using global variables, or use temporary files if memory size is a concern; (5) In AE-only scenarios, set a hidden checkbox activated by script to catch the value.

*Tags: `cep`, `scripting`, `params`, `ui`, `communication`*

---

## How can you implement a button in an After Effects plugin that triggers a script and passes data to it?

Use an arbitrary data parameter to store text/script content, then trigger AEGP_ExecuteScript() via a button event to execute the script string. You can manipulate the script string (e.g., via find-and-replace) before execution to pass data into it, and the script can return data back to After Effects.

*Tags: `aegp`, `arb-data`, `ui`, `scripting`*

---

## What is the recommended way to store plugin scripts as part of the project rather than in external folders?

Store scripts in arbitrary data parameters within the plugin rather than maintaining separate script files in external folders. This keeps scripts bundled with the project/plugin and avoids the clunky workflow of managing scripts outside the After Effects project structure.

*Tags: `arb-data`, `scripting`, `params`*

---

## What is the best approach to restrict two float sliders so that one cannot be equal to or greater than the other?

Handle parameter changes in the UserChangedParams callback. When one slider changes, check the value of both sliders and apply corrective values to maintain the constraint. However, avoid updating parameter min/max values dynamically while sliding, as this can cause issues with keyframing—keyframes with previously acceptable values can become unmutable if they later violate the constraint. A better approach is to treat both parameters as interchangeable "endpoints" and use min() and max() in your code to determine which is Start and which is End, rather than enforcing strict ordering on the parameters themselves.

*Tags: `params`, `ui`, `scripting`*

---

## How do you disable the crash warning popup in After Effects 2024?

Press CMD+F12 to open the Debug Database console. Click the hamburger menu next to the console and change to Debug Database. Search for 'ShowPreviousCrashWarning' and turn it off. This setting can also be configured in the prefs.txt file and can be scripted for automation.

*Tags: `debugging`, `macos`, `scripting`, `ui`*

---

## What is the Adobe After Effects expressions documentation for layer space transformations?

The Adobe After Effects expressions documentation for layer space is available at https://ae-expressions.docsforadobe.dev/layer-space.html. This reference defines how layer space works in expressions and how path.points() correctly returns vertices in true layer space, independent of the layer's transform properties.

*Tags: `reference`, `scripting`, `documentation`*

---

## What are alternatives to Nuitka for compiling Python code in After Effects plugins, especially for resolving dependency conflicts on macOS?

Cython is a viable alternative to Nuitka for compiling Python code in AE plugins. It can be used to compile specific portions of Python code and has fewer dependency conflicts compared to Nuitka, particularly when dealing with libraries like NumPy on macOS (which can have issues with libopenblas.0.dylib).

*Tags: `scripting`, `macos`, `build`, `cross-platform`, `deployment`*

---

## What are the size limitations for storing custom metadata in After Effects projects using xmpPacket from ExtendScript?

Custom metadata stored via xmpPacket in ExtendScript has a practical size limit of less than 500kb. After Effects itself stores significantly more than 500kb of data in its XMP packet for reference. The metadata is not regulated by the undo manager and does not appear to be undoable.

*Tags: `arb-data`, `scripting`, `reference`*

---

## What is a pseudoeffect in After Effects plugin development?

A pseudoeffect is a nicer alternative to using standard expression control effects like Slider Control or Point Control. Pseudoeffects are written in XML and create a regular-looking effect parameter UI without actually performing any rendering—they function purely as control elements that can be read from expressions. They provide a more professional appearance than exposing multiple Slider Controls directly to users.

*Tags: `ui`, `params`, `scripting`*

---

## Can you procedurally modify mask vertices each frame in an After Effects plugin effect?

You cannot modify project state (including masks) in the render thread of a plugin. However, procedurally changing mask vertices might be possible using expressions. Alternatively, 'baking' the paths is an effective workaround, though not as dynamic. If the functionality is available in the JavaScript API, you could potentially run scripts from within your plugin to achieve this.

*Tags: `aegp`, `render-loop`, `params`, `scripting`*

---

## What is the AEGP_MaskOutlineSuite and how does it differ from procedural mask modification?

The AEGP_MaskOutlineSuite allows you to work with mask outlines, but modifications made through it would be destructive and one-time only (similar to running a script that modifies path vertices). This differs from a regular effect plugin that could procedurally modify path vertices each frame before rasterization.

*Tags: `aegp`, `params`, `scripting`*

---

## How can you detect the After Effects version from within a plugin?

You can use ExtendScript with 'app.version' via the AEGP execute script. Alternatively, if your plugin supports both After Effects and Premiere Pro with GPU acceleration, you have access to a Premiere Pro PICA Suite AppInfo that provides the version. The version data is in format like 24.3, and you need to add 2000 to get the actual version number (e.g., 24.3 becomes 2024.3). Some versions may include additional components like 24.3.x which require parsing.

*Tags: `aegp`, `scripting`, `version-detection`, `premiere`, `gpu`*

---

## What causes the 'Cannot run a script while a modal dialog is waiting for response' error when calling scripting from a plugin?

This error occurs when attempting to run scripting from a plugin while a modal dialog is active and waiting for user response. It is a limitation of using scripting from plugins - the scripting engine cannot execute while After Effects has a modal dialog blocking the main thread. This can happen even after checking if scripting is available first, and may occur during parameter setup.

*Tags: `scripting`, `aegp`, `debugging`, `ui`*

---

## How can you find and use the 'Reveal in Composition' command ID in After Effects scripting?

The 'Reveal in Composition' command ID can be found using app.findMenuCommandId('Reveal in Composition'), which returns 2775. However, this command may not work in all contexts and might return null if the menu is just a category menu rather than an executable command. An alternative approach is to mimic the behavior manually by opening the desired composition and selecting the desired layer, which often produces better results than relying on the command ID.

```cpp
app.findMenuCommandId('Reveal in Composition'); /// 2775
```

*Tags: `scripting`, `ui`, `aegp`, `reference`*

---

## What is the command ID for 'Reveal in Timeline' in After Effects?

The 'Reveal in Timeline' command ID is 2536, though it may require specific context to work properly or may not function as expected in all scenarios. Users attempting to use this command should be aware that it may need particular conditions to be met or may behave differently than anticipated.

*Tags: `scripting`, `ui`, `aegp`, `reference`*

---

## What should be considered when mapping After Effects command IDs to avoid conflicts?

When creating mappings of AE command IDs, avoid treating keys as unique since a single command name can map to multiple IDs with different meanings. For example, 'undo' maps to both ID 16 (the actual undo command) and ID 2371 (clear undo), but only one will display. Ensure the correct ID is shown for each command's actual function.

*Tags: `aegp`, `scripting`, `reference`, `debugging`*

---

## How do you debug ExtendScript for After Effects on macOS?

ExtendScript debugging on macOS has traditionally been difficult, requiring the Intel version which is slow. However, Adobe recently updated the debugger (about a month ago) to be Apple Silicon native, which is a significant quality of life improvement for macOS developers.

*Tags: `scripting`, `debugging`, `macos`, `apple-silicon`*

---

## How should autoKernType be set to read kerning property values in After Effects text layers?

The autoKernType property needs to be set to either 'none' or 'undefined' in order to read the kerning property. When autoKernType is set to 'metric' or 'optical' (which means automatic by the host), the kerning property may not read or write as expected. Manual kerning is indicated by setting autoKernType to 'undefined', but this behavior appears to have issues when keyframes are involved in AE 2024.3.

*Tags: `scripting`, `params`, `debugging`*

---

## Can a plugin expose generated data back to the user for use in expressions?

The question was asked but not answered in the conversation. No response was provided about whether plugins can expose data to expressions or other user-accessible formats.

*Tags: `aegp`, `params`, `scripting`*

---

## How can internally computed positions from a plugin be exposed to users in After Effects?

Internally computed positions can be exposed to users by outputting them through a point control parameter. This allows users to index the values in expressions or access them programmatically. One practical example is exporting bounding boxes as nulls, which is a technique used in projects like Vision (https://aescripts.com/vision).

*Tags: `params`, `ui`, `scripting`, `expressions`, `arb-data`*

---

## How can internally computed positions generated during plugin rendering be exposed to users for use in expressions or parameters?

When a plugin generates its own set of positions as part of the render process (rather than reading from an existing AE path), those values can potentially be exposed to users in two ways: (1) by indexing them in expressions, allowing users to reference computed values dynamically, or (2) by outputting them through a point control parameter, which would make the values available as a controllable/readable parameter in the After Effects UI.

*Tags: `params`, `ui`, `scripting`, `aegp`*

---

## What are alternative approaches to accessing text layer properties when the standard AEGP text API is broken?

Alex Bizeau shared that the Kerning API in ExtendScript is broken, but works well in Expression Script. A workaround is to use ExtendScript to write an expression to a text layer that writes values to a separate text layer, then read that second layer's value back in ExtendScript. This bypasses the broken AEGP text kerning API. While hacky, this demonstrates using expressions as an intermediary to work around AE API limitations.

*Tags: `aegp`, `scripting`, `ui`, `reference`*

---

## What is a method to pass text data from a render into an expression in After Effects?

One semi-reliable method is to encode and render dead pixels in your render output, then use a sampleImage expression to retrieve those pixels, which can then set the source text. This technique works with both regular layers and Null objects, though it has a few caveats.

*Tags: `scripting`, `expressions`, `render-loop`, `workaround`*

---

## How can I detect when a plugin loads an old project to change default UI visibility between versions?

You can use the After Effects documentation to detect project compatibility. When loading an old V1 project in V2, check if legacy parameters exist or have non-default values during initialization, then programmatically set your UI mode (Simple vs Advanced) accordingly. The actual implementation details are found in the official AE SDK documentation.

*Tags: `params`, `ui`, `scripting`, `reference`*

---

## How can you keep effect parameters synced between multiple locations when users apply keyframes and expressions?

This is identified as a challenging problem in plugin development. When trying to mirror parameters across multiple UI locations (such as keeping layer effect controls synced with a master adjustment layer control), synchronization becomes difficult if the user applies keyframes or expressions to either location. The conversation suggests using AEGP (After Effects General Plugin) calls or userchangedparam script callbacks as potential approaches, but the questioner notes that constantly mirroring these complex parameter states is not straightforward.

*Tags: `aegp`, `params`, `ui`, `scripting`*

---

## Where should After Effects plugin developers share and host code snippets?

Stack Overflow is recommended as a good platform for sharing code snippets and knowledge about After Effects SDK development. Using the tag 'After Effects SDK' makes the content searchable and discoverable. While GitHub could be used, Stack Overflow's forum structure is better suited for this niche community compared to chat systems which have limitations for code hosting.

*Tags: `open-source`, `reference`, `tool`, `deployment`, `scripting`*

---

## What are the advantages of using Lua over Python for developing After Effects plugins?

Lua is significantly better suited for plugin development compared to Python. The main advantage is that Lua is much easier to compile in and integrate as a library. Python integration was problematic because it required asking permission to install libraries all over the user's system, which is intrusive. Lua avoids this dependency management issue entirely. Both languages require similarly tedious C bindings/code generation, but Lua's compilation and integration model is cleaner for plugin distribution.

*Tags: `scripting`, `plugin`, `build`, `deployment`*

---

## How can you bind C code to scripting languages for After Effects plugins?

There are multiple approaches to create C bindings for scripting languages in AE plugins. For Python, you can use existing bindings (like cairo/pycairo) and pass parameters as text using sprintf formatting (e.g., sprintf("%s=%s\n", paramname, value)), then use the language's C API for bitmap and layer parameter assignment. For Lua, you can use XML-based C-binder code generation tools. Both approaches are similarly tedious without automation or code-generation tools to facilitate the binding process.

```cpp
sprintf("%s=%s\n", paramname, value)
```

*Tags: `scripting`, `plugin`, `build`, `aegp`*

---

## How can you trigger parameter updates in Premiere Pro from a CEP script when PF_userChangedParam doesn't work?

In After Effects, using a hidden parameter updated by a script calling PF_userChangedParam works well for triggering updates. However, Premiere Pro does not consider script interactions as user actions, so this approach is not reactive. Alternative solutions include: monitoring external file dependencies for changes (though this is less reactive), or setting up a button to force manual refresh of the plugin parameters.

*Tags: `premiere`, `scripting`, `params`, `cep`*

---

## How can you retrieve the string value of an arbitrary parameter in an After Effects effect using ExtendScript?

One approach is to use a hidden checkbox parameter that is supervised. From the JavaScript side, leave a message in global scope that the C side can read using AEGP_ExecuteScript(). Toggle the checkbox to trigger a USER_CHANGED_PARAM call. From the C side, use AEGP_ExecuteScript() to check for the message, read the string value from the arb data, and pass it back to JavaScript via AEGP_ExecuteScript(). The execution then returns to JavaScript where the string value is available in global scope. An alternative simpler approach is to use hidden UI elements and send the string to their controller name, then retrieve the property name and combine them in the JSX side. Note that there may be a limit on parameter name length, so avoid exceeding it to prevent crashes.

*Tags: `scripting`, `arb-data`, `aegp`, `params`*

---

## How can I invoke an AEGP from ExtendScript without creating a menu item?

There are two main approaches: (1) Use AEGP_Menu_NONE with InsertMenuCommand to avoid creating a visible menu, though this may trigger an alert sound on startup. (2) Set a flag in the JavaScript global scope and have the AEGP check for that flag on idle_hook calls, which occur 20-50 times per second. This provides asynchronous communication without menu creation. Alternatively, use a C external object to bridge ExtendScript and AEGP directly, though this requires more complex setup.

*Tags: `aegp`, `scripting`, `debugging`*

---

## Is there an example of passing strings from ExtendScript to a custom After Effects plugin via ExternalObject?

Yes, there is a working example provided by the community that demonstrates C external object communication with ExtendScript. The example shows how to structure a C external object for passing strings between ExtendScript and an After Effects plugin. Link: https://community.adobe.com/t5/after-effects-discussions/issue-passing-string-from-extendscript-to-custom-after-effects-plugin-via-externalobject/m-p/15019735#M259100

*Tags: `aegp`, `scripting`, `reference`, `open-source`*

---

## Is it possible to hot reload After Effects plugins without restarting the application?

No, hot reloading is not practical for native C++ After Effects plugins. Each time you build C++ code, you must restart After Effects because the application calls parameter setup only once per session when the plugin is first loaded, so hot reloading cannot trigger re-initialization of parameters. However, ExtendScript and CEP plugins can be reloaded without restarting the entire AE application. Debugging through an IDE like Microsoft Visual Studio or Apple Xcode can make the restart cycle faster than manually launching AE.

*Tags: `params`, `build`, `debugging`, `sdk`, `scripting`*

---

## How can I retrieve layer comments from within an After Effects plugin using the C API?

There is no direct C API available to get layer comments. The recommended workaround is to use AEGP_ExecuteScript() to execute a script that retrieves the layer comment, and then retrieve the result back to the C side.

*Tags: `aegp`, `scripting`, `api`*

---

## What is the best practice for spawning a window with text input UI from effect parameters?

There are several approaches: (1) Use AEGP_ExecuteScript to launch a JavaScript prompt for easy cross-OS compatibility—it takes text input from the user and you can check the returned value to see if the user hit 'ok' or 'cancel'. (2) Use OS-level windows for more advanced UI, which works on both macOS and Windows but requires more documentation review. (3) Store the text data using either sequence data or an arbitrary (arb) parameter—the arb route is preferred as it allows undo/redo. You can add a custom UI to the arb parameter that both displays the current string value and is clickable, or use a simple button parameter to trigger the window while keeping the arb parameter hidden.

*Tags: `ui`, `params`, `arb-data`, `cross-platform`, `aegp`, `scripting`*

---

## How are colors represented in After Effects expressions?

In After Effects expressions, colors use a four-number array format: [red, green, blue, alpha]. All four values must be provided in expressions, or you can use color space conversions to generate the values. While the color data type supports four channels internally, not all color parameters expose the alpha channel for user editing.

```cpp
[red, green, blue, alpha]
```

*Tags: `params`, `expressions`, `scripting`*

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

## How can I handle the save event in an After Effects plugin (AEGP)?

You can use the command_hook to detect save events, but it is not 100% reliable as sometimes saves occur without triggering a call. A more robust approach is to implement an idle_hook on your AEGP that checks if the project is dirty. Track the dirty flag state between hook calls—if the project transitions from dirty to clean between successive idle hook invocations, it indicates the user has saved the project.

*Tags: `aegp`, `scripting`, `reference`*

---

## How can a C++ plugin communicate with a CEP panel in After Effects?

C++ plug-ins and CEP panels can communicate using Vulcan (CSXS Events). However, there is no official Vulcan implementation in the C++ SDK. As a workaround, developers have used JSX scripts with AEGP_ExecuteScript to dispatch Vulcan events from plugin to plugin, and invisible checkboxes in the plugin UI to catch messages from CEP back to the plugin.

*Tags: `cep`, `aegp`, `scripting`, `plugin-communication`*

---

## Why does JSON sometimes work and sometimes fail when called from a plugin via AEGP_ExecuteScript?

JSON is not natively defined in ExtendScript, though it may be available in CEP. The intermittent availability suggests that another script on the system may define the JSON standard globally, making it available to all subsequent scripts. Once defined, it persists until After Effects is restarted. To ensure consistent behavior, use the Douglas Crockford JSON-js library (json2.js) explicitly in your script.

*Tags: `scripting`, `aegp`, `json`, `extendscript`*

---

## How can you rename an effect instance programmatically using AEGP?

To rename an effect instance, get the effect's param 1 reference stream, then retrieve its parent stream using AEGP_GetNewParentStreamRef. This parent stream is the effect's stream, which you can then rename using AEGP_SetStreamName.

*Tags: `aegp`, `params`, `scripting`*

---

## Can invisible effect parameters be accessed from expressions in After Effects?

Invisible parameters marked with PF_PUI_INVISIBLE cannot be accessed by name or match name in expressions, but they can be accessed by their index number. This was confirmed through experimentation where accessing by index worked successfully, while name and match name references returned property missing errors.

*Tags: `expressions`, `params`, `scripting`, `aegp`*

---

## How can you dynamically find After Effects menu command IDs instead of hardcoding them?

Use app.findMenuCommandId() to dynamically look up command IDs by name. This is the recommended approach because command numbers can change between After Effects versions. For example: id = app.findMenuCommandId("copy"); or app.executeCommand(app.findMenuCommandId("Close Project"));

```cpp
id = app.findMenuCommandId("copy");
app.executeCommand(app.findMenuCommandId("Close Project"));
```

*Tags: `scripting`, `extendscript`, `ui`, `cross-platform`*

---

## Is there an open-source searchable database of After Effects command IDs?

Justin Taylor at Hyper Brew maintains an updated searchable Command ID list for After Effects 2022 at https://hyperbrew.co/blog/after-effects-command-ids/, which is useful for scripting and tool development. There was also a Bitbucket snippet repository at https://bitbucket.org/justin2taylor/workspace/snippets/aLjjBE with command ID resources.

*Tags: `scripting`, `extendscript`, `reference`, `tool`, `open-source`*

---

## How can you trigger cache purging from within a C++ plugin when a user changes a parameter?

You can use JavaScript execution via the C API to trigger the cache purge command. Execute `app.executeCommand(2370)` through the ExecuteScript function. However, this approach is strongly discouraged because purging RAM forces users to re-cache all other timelines or portions of the current timeline. Instead, ensure that parameter changes properly invalidate cached frames automatically, or use an invisible parameter change to force invalidation if needed.

```cpp
app.executeCommand(2370);
```

*Tags: `params`, `scripting`, `caching`, `memory`*

---

## Where should AEGP_ExecuteScript be called in a C++ plugin to display a dialog each time the effect is applied?

AEGP_ExecuteScript should not be placed in GlobalSetup or ParamsSetup, as both occur only once per session. Instead, use sequence_setup, which is called once when a new instance is applied to a layer (not when duplicated or copy/pasted), or update_params_ui, which requires tracking to avoid launching the dialog multiple times for the same instance.

*Tags: `aegp`, `scripting`, `ui`, `params`*

---

## How can I create a custom text input field in the Effect Control Window instead of using a JavaScript popup?

Custom UI elements in the ECW or comp window must be handled exclusively via DrawBot. You cannot add OS-level text fields directly to the UI. Instead, you need to either: (1) translate user input through AE's event system onto an offscreen text controller and copy its image buffer, or (2) create a text editor from scratch using the event system and DrawBot. Alternatively, use a JavaScript window for input and display the result as non-interactive text in the ECW that launches the JavaScript window when clicked.

*Tags: `ui`, `drawbot`, `aegp`, `scripting`*

---

## How can I change the current time (CTI) in After Effects using an AEGP plugin?

To change the current time (CTI) with a plugin, first get the comp's item from the effect layer, then use AEGP_SetItemCurrentTime. The process involves: (1) using AEGP_GetEffectLayer to get the layer handle, (2) using AEGP_GetLayerParentComp to get the composition handle, (3) using AEGP_GetItemFromComp to get the item handle from the composition, and (4) finally calling AEGP_SetItemCurrentTime with the item handle and desired time value.

```cpp
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &m_layerH);
suites.LayerSuite8()->AEGP_GetLayerParentComp(m_layerH, &m_compH);
suites.CompSuite11()->AEGP_GetItemFromComp(m_compH, &compItemH);
suites.ItemSuite9()->AEGP_SetItemCurrentTime(compItemH, &compTime);
```

*Tags: `aegp`, `cti`, `scripting`, `ui`, `reference`*

---

## How can you refresh the UI to show parameter changes made in PF_Cmd_USER_CHANGED_PARAM?

The PF_OutFlag_REFRESH_UI flag can only be set from specific command selectors (PF_Cmd_EVENT, PF_Cmd_RENDER, PF_Cmd_DO_DIALOG), not from PF_Cmd_USER_CHANGED_PARAM where parameter changes typically need to be reflected. One workaround is to use AEGP_ExecuteScript() to execute JavaScript that triggers a UI update, or restructure the plugin logic to defer UI updates until one of the supported command selectors is called.

*Tags: `params`, `ui`, `render-loop`, `aegp`, `scripting`*

---

## Why does AEGP_ExecuteScript fail with a modal dialog error when called in PF_Cmd_SEQUENCE_RESETUP?

You cannot run AEGP_ExecuteScript while any After Effects dialog is open, including the project loading progress window that appears during sequence setup. The error 'Unable to execute script at line 0. After Effects error: Can not run a script while a modal dialog is waiting for response' indicates a modal dialog is blocking script execution. You need to engineer your process so the script runs at a different time, such as in PF_Cmd_UPDATE_PARAMS_UI instead, where no blocking dialogs are present.

*Tags: `aegp`, `sequence-data`, `scripting`, `debugging`*

---

## Is there a way to access app.settings.saveSetting from C++ plugin code instead of ExtendScript?

The conversation does not provide a direct C++ API equivalent for app.settings.saveSetting. However, the suggested workaround is to defer script execution to a different command handler like PF_Cmd_UPDATE_PARAMS_UI where no modal dialogs are blocking execution, allowing AEGP_ExecuteScript to run successfully.

*Tags: `aegp`, `scripting`, `params`*

---

## How can I find the command ID for Numpad+0 (play preview from start) in After Effects?

Setup a command hook with the 'all commands' flag. Once you've made some preliminary calls, you can press 0 and catch the command ID that is triggered. This method works because the Numpad+0 command doesn't have a corresponding menu item, so app.findMenuCommandId() won't work directly.

*Tags: `scripting`, `aegp`, `ui`, `debugging`*

---

## Is there an API to retrieve the rotation value of individual characters when a text animator is applied?

No, there is no such API available in After Effects. However, you can get the text in its current transformation as shape vertices and attempt to deduce the rotation from the vertex data, though this approach is highly problematic and not reliable.

*Tags: `text`, `api`, `aegp`, `scripting`*

---

## What is an alternative approach to copying and pasting keyframes between parameters in After Effects plugins?

Instead of manually copying keyframe properties through AEGP_GetKeyframeTemporalEase, you can emulate copy and paste functionality using a collection to set selected keyframes and then execute the native After Effects copy/paste commands via AEGP_DoCommand. Alternatively, you can implement the entire process in JavaScript and invoke it from the C API using AEGP_ExecuteScript().

*Tags: `aegp`, `scripting`, `params`*

---

## How can I access data from another plugin in After Effects?

Getting parameter data is relatively straightforward using both the C and JavaScript APIs, and the 'project dumper' sample project demonstrates this. However, there are two significant limitations: (1) accessing arbitrary parameter data (custom type params) is difficult—while you can retrieve the data chunk, deciphering its structure is challenging without knowing the format; (2) accessing sequence data (data stored internally by the plugin rather than in params) is only possible if the plugin provides specific custom infrastructure to expose it, otherwise only the plugin itself can access it.

*Tags: `arb-data`, `params`, `sequence-data`, `scripting`, `reference`*

---

## Is there a sample project that demonstrates how to extract parameter data from After Effects projects?

Yes, Adobe provides the 'project dumper' sample project as a reference implementation for getting parameter data from plugins using both the C and JavaScript APIs.

*Tags: `reference`, `params`, `scripting`, `open-source`*

---

## How do I fix the 'could not convert Unicode characters' error when setting an output file path in After Effects plugins?

Use A_UTF16Char instead of A_char for the output path parameter. The correct approach is to cast a wide string literal to A_UTF16Char and pass it to AEGP_SetOutputFilePath. Example: const A_UTF16Char *outPath = reinterpret_cast<const A_UTF16Char *>(L"C:\\whee.mov"); ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(0, 0, (A_char*)outPath));

```cpp
const A_UTF16Char *outPath = reinterpret_cast<const A_UTF16Char *>(L"C:\\whee.mov");
ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(0, 0, (A_char*)outPath));
```

*Tags: `aegp`, `scripting`, `unicode`, `output-rect`*

---

## Why does AEGP_ExecuteScript cause a crash on macOS but not Windows when closing After Effects?

The crash is likely caused by something in the script itself, not the AEGP_ExecuteScript function. Use binary search debugging: delete half of your script and run again, repeating until the error disappears to isolate the problematic code. In the reported case, the issue was caused by using dlg.close() instead of dlg.hide() on macOS, which behaves differently between platforms.

*Tags: `aegp`, `scripting`, `macos`, `debugging`, `cross-platform`*

---

## How can an AEGP communicate with a CEP/ScriptUI panel to pass data back and forth?

You can use AEGP_ExecuteScript() to send data from the AEGP to your JavaScript panel, and retrieve data from the JavaScript side via the returned value. For layer selection detection, you can use comp.selectedLayers and comp.selectedProperties from JavaScript with a scheduled task to check for selection changes at intervals. Alternatively, use idle_hook in the AEGP to check periodically and execute scripts via AEGP_ExecuteScript(). For deeper integration, there is a JavaScript SDK that allows you to build an AEGP that gets called to execute custom JavaScript APIs.

*Tags: `aegp`, `scripting`, `ui`, `cep`*

---

## What is the performance difference between using a JavaScript scheduled task versus idle_hook for detecting changes in After Effects?

While a direct performance comparison was not formally tested, idle_hook is intuitively less resource-intensive than a scheduled task. For AEGPs, the update menu hook may be called when selections change, though it's uncertain whether it fires when the selection changes or only when the relevant menu is exposed.

*Tags: `aegp`, `scripting`, `performance`, `render-loop`*

---

## How can a C/C++ plugin propagate internal data to JavaScript in After Effects?

Use the AEGP_ExecuteScript() function from the Utility Suite to execute JavaScript code from the C/C++ plugin side. For simple cases, call suites.UtilitySuite5()->AEGP_ExecuteScript(NULL, "alert('hello world!')", FALSE, NULL, NULL); For more elaborate work where you need to retrieve return values from the script, refer to the detailed example in the Adobe forum thread at https://forums.adobe.com/message/3625857

```cpp
suites.UtilitySuite5()->AEGP_ExecuteScript(NULL, "alert('hello world!')", FALSE, NULL, NULL);
```

*Tags: `aegp`, `scripting`, `c-plugin`, `javascript`, `data-communication`*

---

## How do you create a new composition from an imported image or footage item in After Effects?

There is no single command to create a composition directly from footage. Instead, you need to gather the item's data using AEGP_ItemSuite functions like AEGP_GetItemDimensions and AEGP_GetItemDuration to retrieve the necessary information, then use AEGP_CreateComp to create the new composition with that data.

*Tags: `aegp`, `scripting`, `reference`*

---

## How do you recursively search for a folder in an After Effects project and get its contents?

There is no direct API to get a folder's child items. You need to iterate through all project items and use AEGP_GetItemParentFolder() to check which items belong to the folder you're searching for. For folders found within, repeat the process recursively. Alternatively, you can use AEGP_ExecuteScript() to retrieve item IDs through JavaScript (items use the same IDs in both C API and JavaScript), then continue processing on the C side.

```cpp
ERR(suites.ProjSuite5()->AEGP_GetProjectByIndex(0, &projH));
ERR(suites.ProjSuite5()->AEGP_GetProjectRootFolder(projH, &root_itemH));
ERR(suites.ItemSuite8()->AEGP_CreateNewFolder(folder_name, root_itemH, &root_folderH));
// Then iterate through items and check AEGP_GetItemParentFolder()
```

*Tags: `aegp`, `reference`, `scripting`*

---

## What is the simplest way to copy a full mask from one layer to another in After Effects SDK?

While AEGP_MaskSuite and AEGP_MaskOutlineSuite allow mask manipulation, they require vertex-by-vertex copying. For a simpler approach to full mask copying, use AEGP_ExecuteScript() to execute JavaScript code, which typically offers more straightforward mask operations than the C SDK.

*Tags: `aegp`, `mask`, `scripting`, `sdk`*

---

## What is a good approach for creating user option dialogs in AEIO plugins that work on both Windows and macOS?

You can use AEGP_ExecuteScript() to implement user option dialogs in JavaScript, which provides a cross-platform solution that works identically on Windows and macOS. This avoids the need to maintain separate platform-specific dialog code. For implementation details, see the example at https://forums.adobe.com/message/3625857#3625857 and search the Adobe After Effects scripting community forum at https://forums.adobe.com/community/aftereffects_general_discussion/ae_scripting for additional script examples with checkboxes and dropdown lists.

*Tags: `aeio`, `scripting`, `ui`, `cross-platform`, `aegp`*

---

## How can I programmatically detect when a user presses the play button in After Effects?

Use AEGP_RegisterCommandHook() to register a command hook that listens for play events. The relevant command numbers are: 2415 for Play (spacebar) and 2285 for RAM Preview. This allows you to be notified when the user triggers playback rather than implementing a custom onClick function.

```cpp
AEGP_RegisterCommandHook()
// Command numbers:
// 2285 - RAM Preview
// 2415 - Play (spacebar)
```

*Tags: `aegp`, `scripting`, `ui`, `reference`*

---

## How can you add one composition as a layer into another composition using ExtendScript?

You can add a composition item as a layer to another composition using the `comp.layers.add()` method in ExtendScript. The syntax is `comp.layers.add(otherCompItem);` where `otherCompItem` is the composition you want to add. More details can be found in the official After Effects scripting guide.

```cpp
comp.layers.add(otherCompItem);
```

*Tags: `scripting`, `extendscript`, `layer-checkout`*

---

## What language and tools should I use to develop an After Effects plugin for controlling layer properties and exporting data?

For your use case of creating a GUI, controlling layers and layer properties (content, effects, transform), and exporting data as XML, you don't need to develop a C plugin. Instead, use JavaScript/ExtendScript to create a script that can be encrypted as a .jsxbin file. Refer to the After Effects Scripting Guide PDF for detailed information on manipulating layers, effects, and properties. The AE Scripting forum at https://forums.adobe.com/community/aftereffects_general_discussion/ae_scripting is also a valuable resource for scripting questions.

*Tags: `scripting`, `ui`, `deployment`, `reference`*

---

## What is the difference between scripting and plug-ins in After Effects?

Scripting and plug-ins are two different approaches to extending After Effects. Scripting is done with JavaScript and can perform operations in the project and create palettes, but cannot process layer pixels. It is simpler, does not require compiling, and is cross-platform. Plug-ins are created in C++, can do everything scripting does plus process layer pixels to create effect plug-ins, but require Visual Studio on Windows and Xcode on Mac, require C/C++ knowledge, and must be compiled separately for each platform.

*Tags: `scripting`, `cross-platform`, `build`*

---

## Where can I find the official After Effects SDK and scripting documentation?

The official After Effects SDK, scripting guide, and C API documentation can be found on Adobe's developer network at http://www.adobe.com/devnet/aftereffects.html. The scripting guide walks through JavaScript scripting, and the SDK file contains the C API docs for plug-in development.

*Tags: `reference`, `scripting`, `sdk`*

---

## How can I open a file browser dialog in an After Effects plugin to let users select a file?

There is no direct C API call in the SDK to open a file browser dialog. However, you can use AEGP_ExecuteScript() to execute a script that opens a file browser and retrieves the selected file path, then pass that information back to your plugin.

*Tags: `aegp`, `ui`, `scripting`, `reference`*

---

## How can I replace an image on a layer using the After Effects C API?

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

*Tags: `aegp`, `layer-checkout`, `scripting`, `api`*

---

## How can an AEGP plugin modify a composition's current time indicator?

Use the AEGP_SetItemCurrentTime() function from the SDK. This function allows you to set the current time for a given composition item from an AEGP plugin, as opposed to effect plugins which use PF_MoveTimeStep().

*Tags: `aegp`, `scripting`, `reference`*

---

## How can you hide all shy layers in a composition when AEGP_SetCompFlags is not available in the SDK?

While AEGP_SetCompFlags() does not exist in the SDK, you can use AEGP_ExecuteScript() to execute JavaScript code that accomplishes this. The JavaScript code to hide shy layers in the active composition is: app.project.activeItem.hideShyLayers = true;

```cpp
app.project.activeItem.hideShyLayers = true;
```

*Tags: `aegp`, `scripting`, `sdk`*

---

## How can I retrieve text properties like font, color, and size from a text layer in After Effects?

You can use the JavaScript API through AEGP_ExecuteScript() to access text layer properties. To fetch the font name from a text layer (CS6 and up), use: var textProp = myTextLayer.property("Source Text"); var textDocument = textProp.value; textDocument.font; This approach works because text layers have limited SDK support, so scripting is the recommended method. The script is passed as a char pointer and doesn't require an external script file. For more details on launching AEGP_ExecuteScript() and retrieving results back to the C side, see: http://forums.adobe.com/message/3625857#3625857

```cpp
var textProp = myTextLayer.property("Source Text");
var textDocument = textProp.value;
textDocument.font;
```

*Tags: `aegp`, `scripting`, `params`, `reference`*

---

## What is the recommended approach for accessing text layer data from C/C++ plugins?

Text layers are a blind spot in the After Effects SDK (at least through CS6). The recommended approach is to use AEGP_ExecuteScript() to pull text data using the JavaScript API, as the JavaScript API has better support for text properties than the C SDK. The script can be passed directly as a char pointer without requiring an external script file. See http://forums.adobe.com/message/3625857#3625857 for implementation details on launching AEGP_ExecuteScript() and retrieving results back to the C side.

*Tags: `aegp`, `scripting`, `sdk`, `reference`*

---

## Is it possible to override or modify the After Effects Export to XFL command through the API?

No, the After Effects API does not offer means of overriding or modifying the export to XFL command. However, as an alternative, you could apply a script to the generated XML to correct it automatically after export, or run a script that exports data from AE directly in the format you need.

*Tags: `scripting`, `xfl`, `export`, `aegp`, `api`*

---

## What is the best approach to automate XFL transformation point corrections after exporting from After Effects?

Rather than trying to override the built-in XFL export, you can write a script that post-processes the exported XFL file's XML to correct placement of registration points, transformation points, and anchor/position data. This script can automatically transform the XML structure from the standard DOMFrame format to your desired output format, saving manual editing time.

*Tags: `scripting`, `xfl`, `xml`, `export`, `automation`*

---

## How can you create dependent parameters that work correctly with keyframing and expressions in After Effects plugins?

When a parameter is keyframed or uses expressions, the PF_Cmd_USER_CHANGED_PARAM callback is not called during timeline scrubbing, making it difficult to synchronize dependent parameters. The PF_ParamFlag_SUPERVISE flag works for user-initiated changes but not for time-varying data. If the dependent parameter doesn't affect rendering (PUI_ONLY flag), you can use it. Otherwise, one practical solution is to use Motion Script expressions to express dependencies between sliders and attach these expressions during sequence setup, allowing the effect to use the computed parameters during render.

```cpp
params[SLIDER2]->u.fs_d.value = params[SLIDER1]->u.fs_d.value;
params[SLIDER2]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `keyframing`, `expressions`, `scripting`, `aegp`*

---

## How can I execute scripts from within an After Effects plugin?

You can use the AEGP_ExecuteScript() function call to run scripts directly from a plugin without needing to store them as separate files on disk. This is a built-in AEGP suite function for script execution.

*Tags: `aegp`, `scripting`, `api`*

---

## What is an alternative approach to directly detecting parenting changes in After Effects effects?

Instead of trying to detect parenting events directly, you can use expressions to keyframe parenting behavior. One effective method is to use layer markers' comments as parent layer names within position, rotation, and scale expressions. This allows you to simulate dynamic parenting through expressions without needing to monitor parenting change events.

*Tags: `params`, `scripting`, `ui`*

---

## What AEGP SDK function can be used to apply a preset to a layer from a plugin?

There is no native C API method for applying an effect preset in the AEGP SDK. The recommended approach is to use AEGP_ExecuteScript to execute the Layer.applyPreset() JavaScript method instead.

*Tags: `aegp`, `scripting`, `sdk`, `reference`*

---

## How can I get a Pixel Bender kernel's match name to use in an After Effects native plugin?

Create a new After Effects project with a composition containing a single solid layer. Apply your Pixel Bender plugin to the layer. Then run this ExtendScript to retrieve the match name: alert(app.project.item(1).layer(1).effect(1).matchName); This will return the match name (typically based on the id data in the Pixel Bender script) that you can use in your native plugin to make After Effects treat both as the same plugin for backward compatibility.

```cpp
alert(app.project.item(1).layer(1).effect(1).matchName);
```

*Tags: `scripting`, `params`, `reference`, `pipl`*

---

## How can I retrieve text justification from a text layer using the AEGP C++ API?

There is no direct way to access text justification through the C++ AEGP API. However, you can use AEGP_ExecuteScript to execute JavaScript code that accesses the text justification property. The workaround involves calling AEGP_ExecuteScript with a JavaScript function that retrieves the justification value from the text layer, then parsing the returned result. The justification constants are numeric values (6012, 6013, 6014).

```cpp
#define TEXT_JUSTIFICATION_SCRIPT "function getTextJustificationForLayer(layerId) { \
var comp = app.project.activeItem; \
for (var i = 0; i < comp.numLayers; i++) \
if (i == layerId && comp.layer(i+1) instanceof TextLayer) \
return comp.layer(i+1).property(\"ADBE Text Properties\").property(\"ADBE Text Document\").value.justification; \
} \
getTextJustificationForLayer(%d);"

AEGP_MemHandle resultMemH = NULL;
A_char *resultAC = NULL;
A_char scriptAC[AEGP_MAX_MARKER_URL_SIZE] = {'\0'};
sprintf(scriptAC, TEXT_JUSTIFICATION_SCRIPT, layerIndex);
ERR(suites.UtilitySuite5()->AEGP_ExecuteScript(S_my_id, scriptAC, FALSE, &resultMemH, NULL));
ERR(suites.MemorySuite1()->AEGP_LockMemHandle(resultMemH, reinterpret_cast<void**>(&resultAC)));
int just = atoi(resultAC);
ERR(suites.MemorySuite1()->AEGP_FreeMemHandle(resultMemH));
```

*Tags: `aegp`, `scripting`, `text`, `c++`, `workaround`*

---

## How can I control layer copy-paste operations in an After Effects Effect plugin to restrict pasting within the same composition?

Use an AEGP (After Effects General Plugin) rather than an Effect plugin to intercept copy-paste operations. Register a command hook using AEGP_RegisterCommandHook to be notified whenever the user copies or pastes. This allows you to check the collection being pasted and remove or delete effects as needed. You can use AEGP_Command_ALL with AEGP_RegisterCommandHook to discover the command numbers for copy and paste operations. Attempting this from within an Effect plugin alone is not practical.

*Tags: `aegp`, `pipl`, `layer-checkout`, `scripting`*

---
