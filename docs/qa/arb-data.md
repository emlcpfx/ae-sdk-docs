# Q&A: arb-data

**104 entries** tagged with `arb-data`.

---

## How do you implement a text editor in the Effect Control Window of an AE plugin?

Instead of trying to handle PF_Event_KEYDOWNs directly (which has limitations like missing Tab key and no param index), use a button that runs a script via AEGP_ExecuteScript(). The script string can display a text box dialog. Pass data to it by find+replacing in the script string before execution. Store the text in an arb data parameter. The script can return data back to AE.

*Contributors: [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2023-07-31 · Tags: `custom-ui`, `text-input`, `aegp-execute-script`, `arb-data`, `button-param`*

---

## How does PF_Arbitrary_NEW_FUNC work, and when is it called?

PF_Arbitrary_NEW_FUNC is never called in post-CS6 AE. The way arb data works now: you allocate a default arb handle during ParamSetup. For every new instance, AE sends PF_Arbitrary_COPY_FUNC with your default arb handle. Your arb data won't be zero length because it copies from the default you set up. Follow the ColorGrid SDK example which uses both NEW_FUNC and ParamSetup for this.

*Contributors: [**rowbyte**](../contributors/rowbyte/), [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2023-08-22 · Tags: `arb-data`, `arbitrary-data`, `param-setup`, `new-func`, `copy-func`*

---

## What is the Compute Cache API and how does it differ from sequence data and arb data?

Compute Cache replaces sequence data for MFR-compatible caching. Key differences: (1) Compute Cache can be read/written from any thread (render or UI), (2) it is NOT saved with the project (unlike arb data), (3) it's designed for per-frame computed values. Register it in GlobalSetup with AEGP_ClassRegister. The key function should include effect ref, layer ID, effect position, current time, and time scale. Pass in_data->pica_basicP through the options refcon pointer to access suites inside callback functions. Must be registered during GlobalSetup, first access possible during ParamSetup.

```cpp
#include "AE_ComputeCacheSuite.h"

struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
};

A_Err MyGenerateKeyFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeKeyP out_keyP) {
    access_cache_data* accessCacheData = static_cast<access_cache_data*>(optionsP);
    SPBasicSuite* bsuite = accessCacheData->in_data->pica_basicP;
    // Generate key using AEGP_HashSuite1...
    return A_Err_NONE;
}

static PF_Err GlobalSetup(...) {
    AEFX_SuiteScoper<AEGP_ComputeCacheSuite1> compute_suite(in_data, kAEGPComputeCacheSuite, kAEGPComputeCacheSuiteVersion1);
    AEGP_ComputeCacheCallbacks callbacks = { MyGenerateKeyFunc, MyComputeFunc, MyApproxSizeValueFunc, MyDeleteComputeValueFunc };
    compute_suite->AEGP_ClassRegister("com.mycompany.effect.myComputeCacheClass", &callbacks);
}
```

*Contributors: [**tlafo**](../contributors/tlafo/), [**gabgren**](../contributors/gabgren/), [**Tim Constantinov**](../contributors/tim-constantinov/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2024-02-27 · Tags: `compute-cache`, `mfr`, `sequence-data`, `arb-data`, `caching`, `thread-safety`*

---

## Is 5000+ arb dispose calls on AE shutdown normal?

Yes, this is expected behavior. AE's allocators generally don't dispose until available memory is saturated, thread destruction, or app exit. You can test this by setting AE's available memory artificially low (like 2GB) - you'll see many more destruction calls during normal operation. On exit, AE cleans up everything at once.

*Contributors: [**rowbyte**](../contributors/rowbyte/) · Source: adobe-plugin-devs · 2024-12-22 · Tags: `arb-data`, `memory-management`, `dispose`, `shutdown`*

---

## How does AE's Custom UI / arb params bug in AE 2025 manifest?

In AE 2025 (25.0.1), custom UI and arb params display glitches: one arb param's UI can corrupt other arb UIs, causing flashes or scrambled/duplicated content. The likely culprit is NewImageFromBuffer in DrawBot - every call may be setting pixels of all retained DRAWBOT_ImageRef within the same context. Adobe acknowledged fixing something in arb handling, but it may have introduced this unintended consequence. The issue affects multiple third-party plugins including Trapcode.

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**rowbyte**](../contributors/rowbyte/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2024-12-10 · Tags: `custom-ui`, `arb-data`, `drawbot`, `ae-2025`, `bug`, `regression`*

---

## How do you serialize/capture arb data from arbitrary third-party plugins?

Arb data is serialized via PF_Cmd_Arbitrary_Callback using flatten/unflatten functions, similar to sequence data. The flatten/unflatten versions are not accessible via AEGP functions. For plugins with pure stack-variable arb data (like Curves), you can reinterpret_cast the handle as char* and serialize directly. However, plugins with pointers in their arb structs (like Liquify) cannot be serialized this way because deserialized pointer addresses won't be valid. ExtendScript may offer a path through stream values, but it's at the limit of what's possible.

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**James Whiffin**](../contributors/james-whiffin/) · Source: adobe-plugin-devs · 2025-02-15 · Tags: `arb-data`, `serialization`, `flatten`, `unflatten`, `aegp`*

---

## What causes an error when working with arbitrary parameters?

The error occurs if you don't give the arbitrary (arb) parameter a name.

*Tags: `arb-data`, `params`, `debugging`*

---

## Is it normal to receive thousands of calls to PF_Arbitrary_DISPOSE_FUNC on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls on shutdown is likely due to MFR (Multi-Frame Rendering) duplicating arbitrary parameters for each thread, causing many copy and dispose cycles as parameters change and threads are cleaned up during shutdown.

*Tags: `mfr`, `arb-data`, `threading`, `memory`*

---

## Can we call PF_PROGRESS and PF_ABORT from another thread started in the render function?

It should be possible, but you may need atomic operations or mutexes to work on all threads at once. However, there are challenges: the progress bar is only rendered at the bottom of the UI, and multiple threads may report progress out of order since they complete asynchronously. A better approach is to bake the computation in the backend (like in userchangedParam), update an arb param during the update, store the result in sequence data, and apply it (similar to warp stabilizer).

*Tags: `threading`, `render-loop`, `arb-data`, `ui`*

---

## How can we report progress while a blocking rendering computation is happening in the main thread?

The standard PF_PROGRESS approach won't work if the main thread is blocked. Instead, consider baking the computation in the backend (like in userchangedParam), updating an arb param during processing, storing the result in sequence data, and applying it in the render function (similar to how warp stabilizer works). This avoids blocking the main thread and allows proper progress reporting.

*Tags: `render-loop`, `params`, `arb-data`, `threading`*

---

## Can PF_ADD_ARBITRARY2 be used with PF_PUI_INVISIBLE flag?

PF_ADD_ARBITRARY2 can be used with PF_PUI_INVISIBLE, but there are specific requirements for the ARB param setup. The ARB needs special functions implemented correctly, similar to the ColorGrid example in the SDK.

*Tags: `arb-data`, `params`, `ui`*

---

## Should arbitrary data be stored in a separate hidden param or saved in the popup param itself?

An ARB is an independent param, so it should be a separate hidden parameter, not saved within the popup param.

*Tags: `arb-data`, `params`*

---

## Why does an arbitrary data struct work with int but fail with std::string?

std::string cannot be used in ARB data because it cannot be flattened or serialized to be saved on disk. Use char arrays (char[xxx] where xxx is the max size) instead for storing strings in arbitrary data.

*Tags: `arb-data`, `params`, `memory`*

---

## How can you pass data between a button parameter and a script execution in an After Effects plugin?

Store the script text in an arbitrary data parameter. When a button is clicked, use AEGP_ExecuteScript() to execute the script string. You can pass data by find-and-replacing values in the script string before execution, and the script can return data back to After Effects.

```cpp
AEGP_ExecuteScript()
```

*Tags: `arb-data`, `params`, `ui`, `scripting`, `aegp`*

---

## Should script text be stored in an arbitrary parameter when implementing a button that triggers script execution?

Yes, the script text should be stored in an arbitrary parameter. A button parameter triggers the AEGP_ExecuteScript() function to execute the script string stored in the arb data.

*Tags: `arb-data`, `params`, `ui`, `aegp`*

---

## Is PF_Arbitrary_NEW_FUNC ever called in post-CS6 AE architecture?

No, PF_Arbitrary_NEW_FUNC is not called in post-CS6 architecture. Instead, you allocate a default Arb handle during ParamSetup, and for every new instance AE sends a PF_Arbitrary_COPY_FUNC selector with your default arb handle. The arb data won't be zero length.

*Tags: `arb-data`, `params`, `aegp`*

---

## What is the correct way to initialize arbitrary data in ParamSetup?

In ParamSetup, you fill the default arb data by calling a custom function to create the default arb. You allocate the default Arb handle during ParamSetup, and AE will send PF_Arbitrary_COPY_FUNC selectors with your default arb handle for each new instance.

*Tags: `arb-data`, `params`, `aegp`*

---

## What are the advantages of compute cache compared to arbdata for storing pointers to heap objects that are flattened/unflattened?

Compute cache can be written and read from any thread (render or UI thread) using std::atomic for thread safety. Unlike arbdata, compute cache is not stored with the project. Compute cache is designed to get a value per frame, whereas arbdata requires a global context where you define arrays or vectors to store data for different frames. Compute cache replaced sequence data after MFR was introduced since sequence data can no longer be used.

*Tags: `compute-cache`, `arb-data`, `sequence-data`, `threading`, `memory`, `mfr`*

---

## Can data from compute cache be transferred to arbdata to persist it in the project?

Yes, data from the compute cache can be transferred to arbdata if needed so it gets saved in the project. You can read the compute cache from arb or sequence thread if you want.

*Tags: `compute-cache`, `arb-data`, `sequence-data`*

---

## What are the size limitations for storing custom metadata in AE projects using xmpPacket from ExtendScript?

There is a practical size limit of less than 500kb for xmpPacket data. AE itself stores more than 500kb in its own XMP Packet for reference. Tools like AE Viewer can remove XMP packets to significantly reduce file sizes (e.g., from 50mb down to 50kb).

*Tags: `scripting`, `arb-data`, `memory`*

---

## Is metadata stored via xmpPacket regulated by the undo manager in After Effects?

No, XMP metadata modifications do not appear to be regulated by the undo manager and are not undoable.

*Tags: `scripting`, `arb-data`*

---

## How can instance-specific data be stored in a project file and accessed in global or paramssetup?

Arb data can be used for this purpose. Arb data can be accessed using special group functions for access and save operations, allowing you to store and retrieve instance-specific data that persists in the project file.

*Tags: `arb-data`, `globalsetup`, `params`, `sequence-data`*

---

## Can arb data parameter values be accessed in paramssetup?

Arb data can be accessed in paramssetup using the standard access functions (similar to how it's accessed in other functions like smartrender), allowing you to read arbP member variables.

*Tags: `arb-data`, `params`, `smartfx`*

---

## What performance improvement was achieved after Adobe fixed the arb handling issue?

A x7 speed up was observed after Adobe fixed something in arb handling, which also fixed copy/paste effect times issues that were previously causing performance problems.

*Tags: `arb-data`, `caching`, `performance`*

---

## What is causing custom UI arb parameters to glitch in After Effects 2025?

The issue is not directly related to arb parameters, but rather to drawing image buffers using the DrawBot suite. The problem occurs when the UI composites multiple images per plugin context. The culprit is likely NewImageFromBuffer, which on every call probably sets the pixels of all retained DRAWBOT_ImageRef objects within the same context, causing flashes or scrambled/duplicated UI. The issue can be avoided if the UI either doesn't composite an image at all or only composites a single image per plugin context.

*Tags: `ui`, `arb-data`, `debugging`*

---

## Is it expected to receive 5000+ calls to arb dispose when shutting down After Effects?

Yes, this is expected behavior. After Effects allocators generally do not dispose of memory until either available memory is saturated, threads are destroyed, or the application exits. You can test this behavior by artificially setting the available memory to a low value like 2GB, which will trigger more destruction calls during runtime.

*Tags: `arb-data`, `memory`, `debugging`*

---

## Is there a way to serialize all plugin data including arbitrary data?

No clear answer was provided in this conversation. The question was asked but not addressed by other participants.

*Tags: `arb-data`, `serialization`, `plugin-data`*

---

## How does After Effects store arbitrary data of a plugin to disk when that plugin is not installed?

Arbitrary data is serialized via PF_Cmd_Arbitrary_Callback using the flatten function, similar to sequence data. However, the flatten/unflatten versions are not accessible via AEGP functions. For plugins like 'curves' with stack variable handles, the data can be reinterpreted cast as char* and serialized. However, for plugins like 'liquify' that have pointers in their struct, serialization is problematic because deserialized pointer addresses are invalid on reload and will cause crashes.

*Tags: `arb-data`, `aegp`, `sequence-data`, `params`*

---

## Is there a way to serialize all plugin data including arbitrary data for template/capsule creation?

For first-party plugins where you know the handle, you can serialize the data yourself. However, for third-party and Adobe stock plugins, it is very tricky. The main challenge is accessing the flatten/unflatten versions of arbitrary data which are not exposed via AEGP functions. Some plugins have serializable stack variable handles (like 'curves') while others contain pointers (like 'liquify') that cannot be reliably serialized and deserialized. Extended script investigation may be a possible avenue to explore.

*Tags: `arb-data`, `aegp`, `scripting`, `params`*

---

## What is the equivalent of Global Data for sharing state across multiple plugins?

The answer was not completed in the conversation. Alex Bizeau was asked whether an extra AEGP plugin is needed to handle cross-plugin data or if the SuiteSuite is the mechanism for this, but no definitive answer was provided.

*Tags: `aegp`, `arb-data`, `scripting`*

---

## How can internally computed position values from a plugin render be exposed to users for use in expressions or as parameters?

The recommended approach is to output the internally computed values through a point control parameter. This allows users to access the generated positions either in expressions or by exporting them as nulls. The answer suggests this is a viable solution, with a reference to similar functionality being planned for the Vision plugin on aescripts.com.

*Tags: `params`, `ui`, `scripting`, `arb-data`*

---

## How do you hide an arbitrary parameter banner in the timeline while keeping it visible in the Effect Control Window?

When working with arbitrary parameters in After Effects, the visibility in the timeline versus the Effect Control Window (ECW) is tied to whether the parameter is keyframeable. Static banners that don't need keyframing can be hidden from the timeline while remaining visible in the ECW by ensuring they are not set up as keyframeable streams. The Color Grid effect demonstrates having parameters visible in both locations because they are keyframeable, but for static-only parameters, the visibility behavior differs.

*Tags: `arb-data`, `ui`, `params`, `debugging`*

---

## What causes an error when working with arbitrary parameters in After Effects plugins?

An error occurs when you don't provide a name for an arbitrary parameter (arb param). Always ensure that arbitrary parameters are given explicit names to avoid this error.

*Tags: `arb-data`, `params`, `debugging`*

---

## How do plugins like Trapcode Particular cache multiple frames over time for features like fluid simulation?

Plugins that need to cache multiple frames typically use the Sequence Data API to store frame information persistently. While a basic setup might store only the last frame in Sequence Data, plugins like Trapcode Particular that perform simulations (such as fluid dynamics) across many frames likely store cumulative or checkpoint data in the Sequence Data structure. This allows them to jump to any point in the composition and recalculate or retrieve cached simulation values. The key is leveraging Sequence Data's ability to persist data across multiple frames rather than just the immediate previous frame.

*Tags: `sequence-data`, `caching`, `memory`, `arb-data`*

---

## Is it normal to receive thousands of PF_Arbitrary_DISPOSE_FUNC calls on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls during shutdown is likely due to Multi-Frame Rendering (MFR) duplicating arbitrary parameters for each thread, combined with parameter changes triggering dispose/copy cycles.

*Tags: `arb-data`, `mfr`, `shutdown`, `threading`, `memory`*

---

## How can I safely detect if a layer is available for checkout in a userChangedParam callback?

A workaround is to validate layer data by checking width/height and Lrect values—if they are random, over 100,000, or negative, the layer is not properly available for checkout. Note that Param[0].u.ld.data will be empty if checkout hasn't occurred in that thread. However, this is not a fully reliable solution, and the proper approach may require additional checks during effect duplication and shutdown phases.

*Tags: `layer-checkout`, `params`, `arb-data`, `shutdown`, `threading`*

---

## Can you call PF_PROGRESS and PF_ABORT from a separate thread spawned during the render function?

No, you cannot call PF_PROGRESS and PF_ABORT from another thread. These functions must be called from the main thread. If you need to report progress during a blocking computation, you should consider alternative approaches such as baking the compute in a backend operation (like in userchangedParam), updating an arbitrary parameter during the update phase, storing results in sequence data, and applying them later (similar to how warp stabilizer works). This avoids blocking the main thread and allows proper progress reporting through the UI.

*Tags: `threading`, `render-loop`, `params`, `arb-data`*

---

## Can arbitrary parameters with PF_PUI_INVISIBLE flag be used to store custom data in After Effects plugins?

Yes, arbitrary (Arb) parameters can be used to store custom data invisibly in plugin parameters. However, there are important constraints: the data structure must be serializable and cannot contain C++ std::string types. Instead, use fixed-size char arrays (char[size]) for string data, as std::string cannot be flattened or saved to disk. Refer to the ColorGrid example in the SDK for proper Arb parameter setup including special functions for initialization and updates.

*Tags: `arb-data`, `params`, `ui`, `reference`*

---

## Why does using std::string in an arbitrary parameter data structure cause errors in After Effects plugins?

std::string cannot be flattened or serialized to disk, which is required for After Effects to save and restore arbitrary parameter data when projects are saved and reopened. Use fixed-size char arrays (char[max_size]) instead to store string data in Arb parameter structures.

*Tags: `arb-data`, `params`, `memory`*

---

## How can you implement a button in an After Effects plugin that triggers a script and passes data to it?

Use an arbitrary data parameter to store text/script content, then trigger AEGP_ExecuteScript() via a button event to execute the script string. You can manipulate the script string (e.g., via find-and-replace) before execution to pass data into it, and the script can return data back to After Effects.

*Tags: `aegp`, `arb-data`, `ui`, `scripting`*

---

## What is the recommended way to store plugin scripts as part of the project rather than in external folders?

Store scripts in arbitrary data parameters within the plugin rather than maintaining separate script files in external folders. This keeps scripts bundled with the project/plugin and avoids the clunky workflow of managing scripts outside the After Effects project structure.

*Tags: `arb-data`, `scripting`, `params`*

---

## How can you retrieve custom data computed from a previous frame in After Effects plugins?

You can use checkout Param during compute cache to access data from previous frames. However, this approach has limitations as it doesn't account for previous effects in the chain. For frame-sized custom data, you need to handle the data flow through the compute cache mechanism while being aware that parameter checkout alone may not fully solve the problem if previous effects need to be considered.

*Tags: `compute-cache`, `params`, `render-loop`, `arb-data`*

---

## Why is PF_Arbitrary_NEW_FUNC selector never received, causing arbitrary data to always be zero length initially?

This is a known behavior where the PF_Arbitrary_NEW_FUNC selector is not consistently triggered during arbitrary data initialization. The user reported that their arbitrary data always starts with zero length despite expecting the NEW_FUNC selector to be called. This appears to be an obscure aspect of the arbitrary data lifecycle that requires alternative initialization strategies.

*Tags: `arb-data`, `aegp`, `debugging`, `params`*

---

## Is the PF_Arbitrary_NEW_FUNC selector called in After Effects post-CS6 architecture?

No, PF_Arbitrary_NEW_FUNC is not called in the post-CS6 architecture. Instead, arbitrary data is initialized by allocating a default Arb handle during ParamSetup by calling a custom function to create the default arb data. For every new instance, After Effects sends a PF_Arbitrary_COPY_FUNC selector with your default arb handle. While the NEW_FUNC code can be left in place just in case, it will not be triggered in practice.

*Tags: `arb-data`, `params`, `aegp`, `reference`*

---

## What are the advantages of compute cache compared to arbdata for storing pointers to heap data?

Compute cache offers several advantages over arbdata: (1) it can be written/read from any thread (render or UI) with optional thread-safety using std::atomic, (2) it's designed per-frame rather than requiring a global context like sequence or arb data, (3) it provides functions to specify whether to wait if another thread is computing the same key. However, arbdata gets saved with the project while compute cache does not. Compute cache replaces sequence data now that MFR is introduced and sequence data can no longer be used.

*Tags: `compute-cache`, `arb-data`, `threading`, `sequence-data`, `mfr`, `memory`*

---

## Can compute cache data be transferred to arbdata for project persistence?

Yes, data from the compute cache can be transferred to arbdata if needed so it gets saved in the project. Additionally, you can read the compute cache from the arb or sequence thread if required.

*Tags: `compute-cache`, `arb-data`, `sequence-data`, `params`*

---

## What are the size limitations for storing custom metadata in After Effects projects using xmpPacket from ExtendScript?

Custom metadata stored via xmpPacket in ExtendScript has a practical size limit of less than 500kb. After Effects itself stores significantly more than 500kb of data in its XMP packet for reference. The metadata is not regulated by the undo manager and does not appear to be undoable.

*Tags: `arb-data`, `scripting`, `reference`*

---

## How can you reliably assign a persistent ID to an effect instance that survives effect reordering?

One approach is to hash the layer ID, composition ID, and index of the effect together. However, this has the limitation that if the effect's index changes (e.g., when effects are reordered), the ID must be recalculated. An alternative suggestion from Adobe forums is to store a copy of the ID in a hidden arbitrary parameter and use the UI/ARB thread to update the value from sequence data, though this approach has not been thoroughly tested and may not be intuitive to implement.

*Tags: `arb-data`, `params`, `ui`, `aegp`*

---

## What performance improvements were observed after Adobe fixed arb handling?

Adobe recently fixed an issue in arb (arbitrary data) handling that resolved copy/paste effect performance issues. A notable case involved applying Trapcode, which showed approximately 7x speed improvement after the fix, though this may have been an unintended consequence of the arb handling changes.

*Tags: `arb-data`, `performance`, `debugging`*

---

## What causes custom UI artifacts and glitching in After Effects 2025 when using DrawBot with arbitrary data parameters?

The issue is not directly related to arbitrary data (arb) parameters, but rather to how DrawBot's Image buffer compositing works. The problem occurs when the UI composites multiple images per plugin context. The culprit appears to be NewImageFromBuffer, which on every call sets the pixels of all retained DRAWBOT_ImageRef objects within the same context, causing visual artifacts like flashes, scrambled, or duplicated UI elements. The issue can be avoided by either not compositing any image in the UI, or by only compositing a single image per plugin context.

*Tags: `ui`, `arb-data`, `drawbot`, `ae-2025`, `debugging`*

---

## Is it expected to receive thousands of arb dispose calls when shutting down After Effects?

Yes, this is expected behavior. After Effects' allocators don't dispose of resources until available memory is saturated or on thread destruction/exit. You can test this behavior by artificially setting the available memory to a low value like 2GB, which will trigger more destruction calls during runtime.

*Tags: `memory`, `arb-data`, `aegp`, `debugging`*

---

## How can you handle migration of sequence data when the plugin format changes over versions?

Add an extra hidden parameter that stores the version number when sequence data is saved. When loading, parse this version information and reset values accordingly. This approach becomes especially useful when you have complex migration logic to handle multiple format versions.

*Tags: `sequence-data`, `params`, `arb-data`, `versioning`*

---

## How does After Effects serialize arbitrary data from plugins to disk?

Arbitrary data is serialized using PF_Cmd_Arbitrary_Callback with the flatten function, similar to sequence data. However, the flatten/unflatten versions are not accessible via AEGP functions. For plugins with stack-variable handles (like "curves"), the handle can be reinterpreted cast as char* and serialized. Plugins with pointers in their structures (like "liquify") cannot be easily serialized this way because deserialized pointer addresses are invalid on reload, causing crashes.

*Tags: `arb-data`, `serialization`, `aegp`, `params`*

---

## What is Maxon Studio and how does it relate to project templates?

Maxon Studio is a template tool that transforms compositions into capsules which can be reused as templates to regenerate projects. It is similar to the AEGP ProjDumper sample but with significantly more features. The tool has been internal since inception, with initial release delayed due to lacking features and UX. It is being prepared for wider studio release with plans to capture arbitrary plugin data into capsules for reuse across 3rd party and Adobe stock plugins.

*Tags: `aegp`, `arb-data`, `tool`, `reference`*

---

## How can internally computed positions from a plugin be exposed to users in After Effects?

Internally computed positions can be exposed to users by outputting them through a point control parameter. This allows users to index the values in expressions or access them programmatically. One practical example is exporting bounding boxes as nulls, which is a technique used in projects like Vision (https://aescripts.com/vision).

*Tags: `params`, `ui`, `scripting`, `expressions`, `arb-data`*

---

## How can you retrieve the string value of an arbitrary parameter in an After Effects effect using ExtendScript?

One approach is to use a hidden checkbox parameter that is supervised. From the JavaScript side, leave a message in global scope that the C side can read using AEGP_ExecuteScript(). Toggle the checkbox to trigger a USER_CHANGED_PARAM call. From the C side, use AEGP_ExecuteScript() to check for the message, read the string value from the arb data, and pass it back to JavaScript via AEGP_ExecuteScript(). The execution then returns to JavaScript where the string value is available in global scope. An alternative simpler approach is to use hidden UI elements and send the string to their controller name, then retrieve the property name and combine them in the JSX side. Note that there may be a limit on parameter name length, so avoid exceeding it to prevent crashes.

*Tags: `scripting`, `arb-data`, `aegp`, `params`*

---

## How can two different plugin types share arbitrary data through their arb parameters?

AE blocks direct expression connections between arb parameters of different plugin types, as it assumes different effects cannot be guaranteed to have the same structure. However, you can read arb values from other effects using AEGP_GetNewStreamValue on the C/C++ side. To reference another plugin instance, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH and effect index. For consistent identification across sessions, store an ID in sequence data and access it via AEGP_EffectCallGeneric, though this has limitations with duplicated effects. Alternatively, rename an invisible parameter, though this won't survive project reloads.

*Tags: `arb-data`, `aegp`, `params`, `cross-plugin`*

---

## What AEGP functions can be used to reference and read arbitrary data from other effect instances?

Use AEGP_GetLayerEffectByIndex to get an effect reference from a layer using AEGP_LayerH and the effect index. Then use AEGP_GetNewStreamValue to read arb values from that effect. For consistent effect identification, store an ID in sequence data and retrieve it using AEGP_EffectCallGeneric.

*Tags: `aegp`, `arb-data`, `sequence-data`*

---

## How can an effect instance identify itself given an AEGP_EffectRefH handle?

Only an effect instance has access to its own sequence data handle when AE passes it during calls, and AE may move that memory between calls. For identifying specific instances, experts recommend using a mechanism where each effect instance stores a unique identifier (like a 64-bit ID generated at instantiation). Since direct inter-instance communication via AEGP_EffectCallGeneric() is problematic for same-type effects, alternative approaches include: (1) using a separate AEGP plugin to mediate calls, or (2) for non-persistent identification, changing a hidden parameter's name to encode the identifier and reading it via the stream suite.

*Tags: `aegp`, `sequence-data`, `params`, `arb-data`*

---

## How can I store custom AEGP plugin data in a comp or project?

There is no official facility for storing AEGP data in a comp or project, but there are several workarounds: (1) Use layer and comp comments fields, which are rarely used by other plugins and not prominently displayed in the UI; (2) Add a null layer to the comp with a clear name like "my stuff, do not delete", lock it, and store data in its comments field while documenting its purpose to users; (3) Create a folder in the project with a clear name and store subfolders named with comp IDs containing the data, minimizing visible clutter.

*Tags: `aegp`, `arb-data`, `project-structure`*

---

## How can sequence data be safely shared and synchronized between PF_Cmd_EVENT and SmartPreRender calls?

Sequence data is intentionally separate between the render and UI threads by design, allowing render threads to execute asynchronously without data changes from other threads. To share data between them, use global data (shared across threads but not instance-specific) or store render instance data with an identifier such as comp item ID + layer ID + effect index on layer, then have the UI thread look up the correct data using a mutex. Note that arb/hidden params can only be written from the UI thread; render threads cannot modify project data.

*Tags: `sequence-data`, `threading`, `render-loop`, `aegp`, `arb-data`*

---

## Which identifiers (comp item ID, layer ID, or effect index) remain stable when layers are reordered in After Effects?

Comp item ID doesn't change during a session but may change between sessions or when projects are imported. Layer ID doesn't change when layers are reordered, but can change when layers are copied, cut/pasted, or duplicated to another comp, and may be re-used after deletion. Layer IDs are only unique within a single comp. Effect index changes when the effect moves in the effect stack but not when the layer is reordered; multiple instances of the same effect on one layer can have the same index.

*Tags: `aegp`, `sequence-data`, `arb-data`*

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

## How should I store time-dependent curve data for a custom parameter and ensure the UI updates as the user draws?

Use an arbitrary parameter (PF_ADD_ARBITRARY2) to store your data. The workflow is: user interacts with UI → data is stored in a LOCAL array → local array is serialized and stored in the arb parameter → AE automatically invalidates cached frames and re-renders. Before displaying the UI, read the value from the arb param, deserialize it into your local array, and draw accordingly. This also ensures undo/redo operations reflect correctly in the UI. You don't need to convert data to a string; serialize it into one continuous block of memory using whatever method works for your data type.

*Tags: `params`, `ui`, `arb-data`, `caching`, `render-loop`*

---

## Should I use global variables to store custom parameter data that needs to be accessed from multiple functions?

No. Instead of using global variables, store user input in a LOCAL array during UI interactions (like DrawEvent), then serialize that local array into an arbitrary parameter. This allows the data to persist, be invalidated properly by AE, and trigger re-renders automatically without relying on global state.

*Tags: `params`, `arb-data`, `ui`, `memory`*

---

## How can I store per-effect instance data and ensure unique instance IDs when effects are copied, duplicated, or loaded from disk?

There is no built-in mechanism for per-instance unique IDs. A practical workaround is to store an instance ID in sequence data and use global data to track instances. On SEQUENCE_RESETUP, flag the sequence data as 'requires checking the id', then scan the project (or layer) for conflicting IDs. If duplicates are found, deduce which is the original instance (by tracking where it was last seen) and assign a new ID to the copy. This approach requires careful state management across copy/paste, duplication, and file loading operations. Note that during SEQUENCE_RESETUP, you may receive either flattened or unflattened data, so tag your data structure to track its state.

*Tags: `sequence-data`, `arb-data`, `aegp`, `threading`*

---

## What is the best practice for spawning a window with text input UI from effect parameters?

There are several approaches: (1) Use AEGP_ExecuteScript to launch a JavaScript prompt for easy cross-OS compatibility—it takes text input from the user and you can check the returned value to see if the user hit 'ok' or 'cancel'. (2) Use OS-level windows for more advanced UI, which works on both macOS and Windows but requires more documentation review. (3) Store the text data using either sequence data or an arbitrary (arb) parameter—the arb route is preferred as it allows undo/redo. You can add a custom UI to the arb parameter that both displays the current string value and is clickable, or use a simple button parameter to trigger the window while keeping the arb parameter hidden.

*Tags: `ui`, `params`, `arb-data`, `cross-platform`, `aegp`, `scripting`*

---

## How should text input be stored with a layer when using arbitrary parameters?

Arbitrary parameters can store the text string and are preferred over sequence data because they support undo/redo operations. The arb parameter itself can be hidden from the user if desired. You can implement a custom UI on the arb parameter that displays the current text value and is clickable, or alternatively use a separate button parameter to trigger the input dialog while the arb parameter stores the data invisibly.

*Tags: `params`, `arb-data`, `ui`, `layer-checkout`*

---

## What is the purpose of the PF_Cmd_SEQUENCE_FLATTEN selector in After Effects plugins?

PF_Cmd_SEQUENCE_FLATTEN is sent by After Effects when saving or duplicating sequences. It requires the plugin to flatten sequence data containing pointers or handles so the data can be correctly written to disk and saved with the project file. The flattened data must be properly byte-ordered for file storage. When this command is received, the plugin should free unflat data and set out_data->sequence_data to point to the new flattened data. As of SDK 6.0, if an effect's sequence data has been flattened, the effect may be deleted without receiving PF_Cmd_SEQUENCE_SETDOWN, and After Effects will dispose of the flat sequence data.

*Tags: `sequence-data`, `sdk`, `arb-data`, `deployment`*

---

## How can you create independent keyframe streams for individual parameters in an arbitrary data effect instead of concatenating all parameters into one keyframe stream?

Adobe deliberately designs some effects with concatenated keyframes (all parameters share one keyframe stream) and others with individual controls (each parameter has independent keyframes). The Levels effect has two versions shipped with After Effects: the standard Levels (concatenated) and Levels (Individual Controls) (independent parameters). This is a design choice rather than a technical constraint. If you need independent animations and keyframing, you can either use the individual controls variant pattern or calculate effects on-the-fly during the Render event without storing interpolation data in ArbData, similar to how Levels works without relying on ArbData for histogram display.

*Tags: `arb-data`, `params`, `ui`, `animation`, `plugin-design`*

---

## Why does the Hue/Saturation effect concatenate all parameters into a single keyframe stream instead of allowing independent parameter animation like Levels (Individual Controls)?

The Hue/Saturation effect uses USER_CHANGED_PARAM to create keys on arbitrary parameters, which results in all slider changes being keyed together rather than independently. This design choice means you cannot animate individual sliders (like Lightness) separately if another slider (like Saturation) is already animated, as all changes are stored in a single ArbData keyframe stream. The Levels (Individual Controls) variant demonstrates the alternative approach where parameters are independent, suggesting that Hue/Saturation's design prioritizes a different workflow, though the constraint could be overcome by redesigning parameter architecture to use separate animation streams.

*Tags: `arb-data`, `params`, `animation`, `ui`, `plugin-design`*

---

## What is the correct way to extract arbitrary data from a stream value in After Effects plugins?

The correct approach is to cast the stream value's arbH handle to a PF_Handle, then use the HandleSuite1 to lock the handle and access the arbitrary data. However, it's critical to ensure you either complete all data operations before releasing the stream value, or copy the data before release. Otherwise, you may be left with an invalidated pointer that could cause crashes or undefined behavior. The stream value should be disposed with AEGP_DisposeStreamValue after you're done with it.

```cpp
PF_Handle arbH = reinterpret_cast<PF_Handle>(streamVal.val.arbH);
arbData *arbP = reinterpret_cast<arbDataP>(suites.HandleSuite1()->host_lock_handle(arbH));
// Use arbP data here before disposing
suites.StreamSuite5()->AEGP_DisposeStreamValue(&streamVal);
```

*Tags: `arb-data`, `aegp`, `memory`, `stream-data`*

---

## How can I hide a custom arbitrary parameter from the timeline while keeping it visible in the Effect Controls Window?

If the custom arbitrary parameter is purely for UI purposes and doesn't need to vary over time, use PF_ADD_NULL instead of creating an arbitrary parameter. This approach avoids the <error> display in the timeline while still allowing UI controls in the ECW. The HistoGrid example demonstrates this pattern effectively.

```cpp
PF_ADD_NULL("parameter_name");
```

*Tags: `ui`, `params`, `arb-data`, `reference`*

---

## What causes a custom arbitrary parameter to display as <error> in the timeline?

The <error> message occurs when an arbitrary parameter doesn't have a proper name assigned to it. Ensure that the custom arb param is given a descriptive name string in the parameter definition.

```cpp
PF_ADD_ARBITRARY2("Banner Name",
  A_long(PF_FpLong(BANNER_WIDTH) * 0.5),
  A_long(PF_FpLong(BANNER_HEIGHT) * 1.0),
  PF_ParamFlag_CANNOT_TIME_VARY,
  PF_PUI_CONTROL,
  def.u.arb_d.dephault,
  BANNER_ID,
  0);
```

*Tags: `params`, `arb-data`, `ui`, `debugging`*

---

## What should the refconPV value be when creating arbitrary data parameters in After Effects plugins?

The refconPV is a pointer to whatever you'd like, and that pointer is passed back to the plugin during arbitrary data calls for that parameter. If you have different handler classes for each arbitrary parameter, you can pass a pointer to an instance of that class. If unused, you can pass null. The value 0xDEADBEEFDEADBEEF seen in examples is just programmer's leetspeak and not significant. This allows you to maintain separate state or handlers for multiple arbitrary data parameters in the same plugin.

```cpp
#define ARB_REFCON (void*)0xDEADBEEFDEADBEEF
PF_ADD_ARBITRARY2( "Color Grid",
UI_GRID_WIDTH,
UI_GRID_HEIGHT,
PF_PUI_CONTROL | PF_PUI_DONT_ERASE_CONTROL,
def.u.arb_d.dephault,
COLOR_GRID_UI,
ARB_REFCON);
```

*Tags: `arb-data`, `params`, `ui`, `reference`*

---

## How do you properly handle undo/redo with sequence data and arbitrary data in After Effects plugins?

Sequence data does not participate in undo/redo, which makes it problematic as a base for calculations that depend on parameter changes. The recommended approach is to store a state identifier in both arbitrary data (arb data) and sequence data. When using sequence data, compare the state identifier in the arb to the one in the sequence data—if they mismatch, regenerate the sequence data and store the corresponding state identifier. Sequence data is best used only when other storage methods don't meet needs, such as for data that should survive a reset button, data that shouldn't trigger undo entries, or temporary data passed between modules.

*Tags: `sequence-data`, `arb-data`, `params`, `debugging`*

---

## What are the appropriate use cases for sequence data in After Effects plugin development?

Sequence data has limited but specific use cases: (1) flagging new instances during unique sequence setup calls, (2) storing data that survives the reset button (unlike arb data), (3) storing data that should not undo/redo or trigger re-renders, and (4) maintaining session data without bloating the project file. It should not be used as a base for parameter-dependent calculations or processed image data, as there is no reliable way to tell when param values have changed and different frames may be processed simultaneously.

*Tags: `sequence-data`, `arb-data`, `params`*

---

## How should arbitrary data reallocation be handled when it depends on non-animatable parameters like 'rows' and 'cols' in a C++ effect plugin?

Arbitrary parameter handlers in After Effects are stateless functions that operate without access to a specific effect instance, effect reference, or sequence data. Instead of trying to fetch parameter values during HandleArbitrary() calls (where effect_ref is usually null and checkout/checkin rarely work), you should reallocate and store allocation-related metadata like 'rows' and 'cols' in your arbitrary data structure itself during UserChangedParam() or the render function. This way, HandleArbitrary() can process the data independently without querying system or instance state. The render function should fetch raw arbitrary data and other relevant pieces available during render time, then process them there rather than during arbitrary parameter handling calls.

*Tags: `arb-data`, `params`, `skeleton`, `memory`*

---

## How does sequence data get synchronized between the UI thread and render thread in Smart Rendering?

Sequence data synchronization between UI and render threads happens at specific times only: when a new instance is created and during specific UI events. Synchronization occurs when PF_Cmd_USER_CHANGED_PARAM is triggered, and in CLICK and DRAG events if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented. The mechanism involves PF_Cmd_GET_FLATTENED_SEQUENCE_DATA and PF_Cmd_SEQUENCE_RESETUP for flattening and unflattening respectively. However, PreRendering and SmartRendering do not automatically trigger this flattening. For reliable data persistence, Adobe recommends using invisible arb parameters which sync on every change, or implementing a message-passing mechanism through global data structures accessible to both threads.

*Tags: `sequence-data`, `smart-render`, `arb-data`, `threading`, `render-loop`*

---

## What is the best way to persist custom objects from the UI to the rendering thread in After Effects plugins?

The recommended approach is to use invisible arb parameters, which create undo entries and provide immediate invalidation of cached frames on changes. Alternatively, if arb parameters don't fit your design, you can devise a message-passing mechanism by placing instructions in a global data structure common to both threads, with the render thread checking for messages. Direct use of sequence data flattening outside of USER_CHANGED_PARAM or UI click/drag events is not reliable and should be avoided, as it can lead to sync issues with SmartRendering and PreRendering.

*Tags: `arb-data`, `threading`, `sequence-data`, `smart-render`, `params`*

---

## How can I force an overlay to refresh when ARB parameter data changes without requiring mouse movement?

Use the PF_Event_IDLE event which is automatically sent to composition UIs. In the idle event handler, check system time or other factors to determine if a refresh is needed. If so, set PF_OutFlag_REFRESH_UI or call PF_InvalidateRect(), which will trigger a PF_Event_DRAW call to redraw the overlay and reflect the updated ARB data.

*Tags: `ui`, `arb-data`, `aegp`, `render-loop`*

---

## Why is arbitrary data always showing default values in PF_cmd_Render and PF_cmd_SmartRender, but correct values in PF_cmd_User_changed_params?

The issue is typically in how the arbitrary data handle is being allocated and passed back to After Effects. Arbitrary data should be flat—the entire data should be one big handle, not a handle containing other handles. When allocating new arbitrary data, create a single handle of the required size (data size + null termination), copy the data into it directly, and pass it to AE. When you pass the handle as an arb value, it becomes owned by After Effects, so do not unlock or release it. Also, use AEGP_GetNewStreamValue instead of PF_checkout_param to retrieve parameter values.

```cpp
PF_Handle newArbH = PF_NEW_HANDLE(newString.length() + 1);
A_char* newExpMemP = (char*)newArbH;
strcpy(newExpMemP, newString.c_str());
params[SKELETON_ARBITRARY_EXPRESSION]->u.arb_d.value = newArbH;
// Do not unlock/release - handle now belongs to AE
```

*Tags: `arb-data`, `params`, `render-loop`, `memory`*

---

## Does arbitrary data support undo/redo operations in After Effects plugins?

Yes, arbitrary data supports undo/redo operations, unlike sequence data which does not. This makes arbitrary data suitable for storing user-modified parameter state that needs to be reversible.

*Tags: `arb-data`, `params`*

---

## How can I access data from another plugin in After Effects?

Getting parameter data is relatively straightforward using both the C and JavaScript APIs, and the 'project dumper' sample project demonstrates this. However, there are two significant limitations: (1) accessing arbitrary parameter data (custom type params) is difficult—while you can retrieve the data chunk, deciphering its structure is challenging without knowing the format; (2) accessing sequence data (data stored internally by the plugin rather than in params) is only possible if the plugin provides specific custom infrastructure to expose it, otherwise only the plugin itself can access it.

*Tags: `arb-data`, `params`, `sequence-data`, `scripting`, `reference`*

---

## How should an importer plugin interact with an effect plugin to share custom file data?

An importer plugin allows custom file types to be imported into After Effects and placed in compositions. There are two main scenarios for importer-to-effect interaction: (1) The importer parses the file on demand and populates an image buffer, which can then be read by the effect like any other layer source. After Effects can cache the result for faster access on subsequent frame renders. This approach expands usability for custom file types. (2) The effect uses the layer selector only to get the selected layer's ID, retrieves the source project item and file path from it, and then parses the file directly in the effect. This approach allows for data files without graphic interpretation and enables selective storage of data rather than loading all file data into RAM. Both scenarios are valid, and you can combine both approaches—parsing in the importer while also allowing the effect to fetch and parse the file path independently.

*Tags: `importer`, `aegp`, `layer-checkout`, `caching`, `arb-data`*

---

## How can I pass data from a PopupDialog function to the Render function in an After Effects plugin?

There are two recommended approaches: (1) Store the message in global_data along with an effect identifier (comp itemID, layerID, and effect index), so the render thread can check if its matching identifier has a message waiting. (2) Write the data to an arbitrary parameter using SetStreamValue(), which gets synchronized immediately and reliably between threads. The arb param approach also allows undo/redo functionality for the user.

*Tags: `sequence-data`, `threading`, `aegp`, `arb-data`*

---

## How should arbitrary data parameters be saved and loaded in After Effects plugins?

When dealing with an ARB (arbitrary) parameter, you should honor the PF_Arbitrary_UNFLATTEN_FUNC and PF_Arbitrary_FLATTEN_FUNC callbacks. The sequence data commands (PF_Cmd_SEQUENCE_FLATTEN and PF_Cmd_SEQUENCE_RESETUP) have nothing to do with ARB parameter persistence. ARB data may get flattened/unflattened during various events such as copy/paste, not only during save/load. If data doesn't reconstruct correctly after honoring these callbacks, the issue is likely with how the flatten/unflatten operations are implemented.

*Tags: `arb-data`, `params`, `aegp`, `memory`*

---

## Is there a sample project demonstrating how to implement arbitrary data parameters in After Effects?

The 'ColorGrid' sample project is a good reference for understanding how to properly handle arbitrary parameters. It demonstrates the correct implementation of ARB param flatten and unflatten operations.

*Tags: `arb-data`, `params`, `reference`, `open-source`*

---

## Should arbitrary parameter data be stored in a single struct for all arb params or separate structs per parameter?

You can use either approach, but a single generalized arb handler that is reusable across multiple arb params is more efficient. Store the data in AE rather than in the handler class itself. During global setup, create and store the handlers in the global data structure, then pass pointers to them during param setup. Keep handlers locked throughout the session using placed new allocation. The key is that except for the 'create new' function, most handler functions remain the same across different arb types.

```cpp
// C++ style approach with reusable handler
// Store handler pointer in arb definition
// Handler class with virtual methods for each arb type
class ArbHandler {
public:
  virtual PF_Err CreateDefault(PF_Handle *arbH) = 0;
  virtual PF_Err Handle(PF_Cmd cmd, PF_Handle arbH) = 0;
};

// During global setup:
ArbHandler *handler = new ArbHandler();
ArbHandler **handlerH = (ArbHandler**)AE_Allocate(sizeof(ArbHandler*));
*handlerH = handler;
// Lock handle and store in global data
```

*Tags: `arb-data`, `params`, `plugin-architecture`, `c++`, `aegp`*

---

## How can you determine which arbitrary parameter is being referenced in callback handlers that are shared across multiple arb params?

Use the C style approach with a switch statement on the disk_id that AE transfers with each PF_Cmd_ARBITRARY_CALLBACK call, or use the C++ style approach where you place a pointer to an arb handling class in the arb definition parameter. This pointer is sent back with every callback, allowing you to use virtual methods to handle different arb types. The C++ approach with virtual methods is cleaner and easier to maintain than switch statements.

```cpp
// C style with disk_id switch
case PF_Cmd_ARBITRARY_CALLBACK:
  switch(params->arb_d.disk_id) {
    case ARB_PARAM_1:
      return Handle_Arb1(arbH);
    case ARB_PARAM_2:
      return Handle_Arb2(arbH);
  }

// C++ style - pointer passed back in callback
// Arb definition includes: (void*)arbHandlerPtr
// In callback, cast back and call virtual methods
```

*Tags: `arb-data`, `params`, `aegp`, `callback`*

---

## How can I share data between the render thread and UI event handlers?

Since After Effects 13.5, sequence_data modified in the render thread cannot be accessed in UI threads. Two workarounds are: (1) add a pointer to sequence_data that is shared between threads (requiring logic to distinguish live pointers from saved invalid ones), or (2) store data in the global_data handle which is shared between threads and instances (requiring logic to identify which data belongs to which instance). The recommended approach is caching values in sequence_data during render and reading them back in event handlers from the UI thread.

*Tags: `sequence-data`, `threading`, `arb-data`, `ui`, `render-loop`*

---

## Why do project files crash after adding new parameters to an effect plugin?

Project files crash when parameters are added because After Effects uses DISK_ID values to associate saved parameter data with plugin parameters. If you change the order of parameter definitions in your enum or reassign DISK_IDs, After Effects cannot correctly map the old saved data to the new parameters. For example, if a slider parameter with DISK_ID=3 is replaced by a checkbox parameter in the new version, the saved slider data will be incorrectly applied to the checkbox, causing crashes. The solution is to ensure that existing parameters retain their exact same DISK_ID values from the old version, and only assign new DISK_IDs to newly added parameters. The DISK_ID order has nothing to do with the visual parameter order in the UI.

*Tags: `params`, `arb-data`, `deployment`, `aegp`*

---

## What is the relationship between parameter definition order and DISK_IDs in After Effects plugins?

There are two separate enums in After Effects plugin parameter setup: one that correlates with the order of definitions in param_setup (used to access params by index), and another for DISK_IDs (used as a unique identifier for each parameter). These two do not need to match. A parameter can be in index #5 in the UI but still have DISK_ID=3 if it had DISK_ID=3 in the previous version. The DISK_ID is what matters for backward compatibility and project file loading, not the parameter order or index.

*Tags: `params`, `arb-data`, `aegp`*

---

## How should you handle sequence data structure changes when adding new parameters?

When modifying sequence data structure, if you extend or modify the structure, old project files loaded with the new plugin will attempt to match the old sequence data to the new structure, potentially causing conflicts. Best practice is to only add new parameters with new IDs at the end of the current parameter set without changing the order of existing parameters. Do not replace existing parameter definitions, only append new ones. This ensures backward compatibility with projects saved using older plugin versions.

*Tags: `params`, `arb-data`, `sequence-data`, `deployment`*

---

## How should you properly manage and copy pixel buffer data when passing worlds between effects using AEGP_EffectCallGeneric?

When passing world data between effects, you need to understand ownership. AE-owned worlds (input/output buffers during render) will be released after the render call returns, so you must copy the pixel data using PF_COPY(). For plugin-owned worlds, you can pass the AEGP_WorldH directly. To copy: (1) create a new AEGP_WorldH with WorldSuite3()->AEGP_New() at the same dimensions, (2) wrap it with WorldSuite3()->AEGP_FillOutPFEffectWorld(), (3) copy pixels using PF_COPY(). In the receiving plugin, assign the AEGP_WorldH to sequence data and call WorldSuite3()->AEGP_Dispose() when done. Only dispose once, and let wrapped PF_EffectWorld structures go out of scope naturally.

```cpp
// Create and copy a world
AEGP_WorldH newWorldH;
suites.WorldSuite3()->AEGP_New(inputP->width, inputP->height, &newWorldH);

PF_EffectWorld newWorld;
suites.WorldSuite3()->AEGP_FillOutPFEffectWorld(newWorldH, &newWorld);

PF_COPY(&inputWorld, &newWorld, NULL, NULL);

// Send to other plugin via AEGP_EffectCallGeneric
suites.EffectSuite2()->AEGP_EffectCallGeneric(pluginID, effectH, &timeT, (void*)newWorldH);

// In receiving plugin's sequence data
seqDataWorldH = receivedWorldH;
```

*Tags: `aegp`, `arb-data`, `memory`, `layer-checkout`*

---

## Why does memory usage increase continuously when using AEGP_EffectCallGeneric to send mesh data between effects?

Memory leaks when using AEGP_EffectCallGeneric typically stem from improper ownership and disposal of allocated data structures. If you're allocating worlds or buffers to send data and not properly disposing of them after use, AE's memory footprint will continuously grow. Ensure that for any AEGP_WorldH you create with WorldSuite3()->AEGP_New(), you call WorldSuite3()->AEGP_Dispose() exactly once when done. If the issue persists even with empty data, verify that your sequence data management isn't holding references to disposed memory and that you're not allocating new structures on every frame without cleanup.

*Tags: `aegp`, `memory`, `caching`, `arb-data`*

---

## How can you save and restore arbitrary data from a 3rd party plugin's property without using presets?

You can use the AEGP (After Effects General Plugin) approach: have an effect plugin communicate with an AEGP via a special suite (see "checkout" and "sweetie" samples). When the effect needs to apply changes, it sends data to the AEGP and stores it there without executing changes immediately. Let the effect finish execution and return. Then, during the AEGP's idle_hook call, check for messages from the effect and execute the changes after the effect is no longer in mid-call. This avoids crashes from modifying the scene while an effect is operating on it. Alternatively, you can bundle a preset file within your plugin and apply it via applyPreset() with a file path, avoiding the need to keep presets in Adobe's preset directory.

```cpp
var thePreset = File("C:/Program Files/Adobe/Adobe After Effects CS5.5/Support Files/Presets/Behaviors/wigglerama.ffx");
theLayer.applyPreset(thePreset);
```

*Tags: `aegp`, `arb-data`, `params`, `plugin-communication`*

---

## What is the recommended way to store large numbers of parameters that need to be saved in a project file?

For storing many parameters (e.g., 1000+ values) that need to be persisted in the project file, use sequence data or arbitrary data rather than individual built-in parameter types. These are automatically saved and recalled with the project. Ensure you correctly handle copying and flattening/deflattening of values in the appropriate functions, especially if using pointers. The SDK examples demonstrate the correct implementation.

*Tags: `arb-data`, `sequence-data`, `params`, `deployment`*

---

## How should I store fixed pixel coordinates in an After Effects plugin preset?

You can store x,y coordinates using several methods: (1) Point params - simple approach, can be hidden from UI; (2) Arbitrary type params - allows custom data structures, supports undo, triggers re-render; (3) Sequence data - single instance per plugin, handles data changes manually, doesn't trigger re-render. Data is stored with the project file. Refer to the ColorGrid sample for arbitrary params and the Supervisor sample for sequence_data.

*Tags: `params`, `arb-data`, `sequence-data`, `ui`*

---

## What is the correct way to create custom hidden streams and store data on layers using AEGP?

There are limited options for storing custom data on layers with AEGP. Dynamic streams (like strokes from the paint tool and text animators) exist but are very specific. For persistent data across sessions, the recommended approach is to keep the data within the AEGP itself rather than on the layer, storing each layer's ID and its comp's project item ID to identify and retrieve the associated data. For temporary data that can be regenerated, storing it in the AEGP is also preferred. Alternative approaches include using effect parameters, layer comments, or layer marker comments, though these have limitations with the SDK.

*Tags: `aegp`, `arb-data`, `layer-checkout`, `sdk`*

---

## How can non-layer effect data be persisted and saved with an After Effects project?

Refer to the Adobe forum discussion at http://forums.adobe.com/message/2837630#2837630 which discusses techniques for saving non-layer effect data with the project file. This resource addresses the challenge of maintaining custom data across sessions in AEGP plugins.

*Tags: `aegp`, `arb-data`, `reference`*

---

## Can I draw custom graphics and keyframe indicators on the After Effects TimeGraph?

No, the After Effects SDK does not provide buffers or APIs for drawing on the TimeGraph or timeline. You can only draw into buffers that AE provides for the composition window, layer window, and effects controls window. Custom parameter UIs created in the Effects Controls Window will not appear in the timeline. As an alternative, you can create a custom UI using an arbitrary data type parameter that displays a synchronized alternative timeline view centered on the current time cursor, allowing users to edit your special keyframes with custom interpolation behavior.

*Tags: `ui`, `params`, `arb-data`, `aegp`, `sdk`*

---
