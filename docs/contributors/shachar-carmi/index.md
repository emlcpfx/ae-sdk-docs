# shachar carmi

**59 contributions** to AE SDK community knowledge.

Top topics: `aegp`, `custom-ui`, `arb-param`, `idle-hook`, `update-params-ui`, `sequence-data`, `performance`, `threading`, `bit-depth`, `smart-render`

---

## How can I make a plugin for 8-bit, 16-bit, and 32-bit projects share the same internal codebase?

Use C++ templates to create 3 instances of your algorithm with different data types and range constants. This allows you to write the core logic once and instantiate it for PF_Pixel8, PF_Pixel16, and PF_PixelFloat with appropriate range constants for each bit depth.

*Source: adobe-forum-sdk · 2025-11-01 · Tags: `bit-depth`, `templates`, `pixel-format`, `smart-render`, `code-reuse` · [View in Q&A](../qa/bit-depth/)*

---

## How to avoid noise artifacts when AE automatically converts 16/32-bit project input to 8-bit for an 8-bit-only plugin?

AE's automatic bit-depth conversion can introduce dithering/noise artifacts. The recommended approach is to let your plugin accept 16 and 32 bpc inputs as well and do the conversion to 8-bit yourself internally. This avoids the warning sign next to your plugin and gives users the benefit of having the output back in 16/32 bpc. You could also report the noise issue as a bug to Adobe.

*Source: adobe-forum-sdk · 2025-11-01 · Tags: `bit-depth`, `dithering`, `noise`, `pixel-conversion`, `8-bit` · [View in Q&A](../qa/bit-depth/)*

---

## What does AEGP_GetLayerToWorldXform actually represent?

AEGP_GetLayerToWorldXform translates XY coordinates from the layer origin to XYZ in composition space. 'World' means composition space. You can use the returned matrix for simple layer-to-comp coordinate conversion, or use it to construct a full 'render view' matrix from the camera's point of view.

*Source: adobe-forum-sdk · 2025-11-01 · Tags: `aegp`, `transform`, `matrix`, `coordinate-space`, `layer-to-comp` · [View in Q&A](../qa/aegp/)*

---

## How can I continuously update an ECW (Effect Control Window) custom UI drawing while dragging a 2D point parameter?

ECW custom UIs get idle calls on regular intervals only when the cursor is within their perimeter. However, if the point parameter is supervised, you can call PF_RefreshAllWindows during interactions, which will force a redraw. It's somewhat wasteful, but it works.

*Source: adobe-forum-sdk · 2025-11-01 · Tags: `ecw`, `custom-ui`, `arb-param`, `refresh`, `drag-interaction` · [View in Q&A](../qa/ecw/)*

---

## How can an AE plugin save a file (like CSV) when a button is clicked?

The AE SDK only deals with importing files into a project or rendering to custom file types. If you want to save arbitrary data to a file at the click of a button, just use standard C/C++ file I/O directly: fopen/fwrite/fclose. There's no need for any special SDK mechanism.

*Source: adobe-forum-sdk · 2025-09-01 · Tags: `file-io`, `button-param`, `csv`, `export`, `fopen` · [View in Q&A](../qa/file-io/)*

---

## How can I checkout paths or masks from a different layer in the AE SDK?

You can fetch any mask using AEGP_MaskSuite6 and read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape param, you need to traverse the layer's dynamic streams. However, there are limitations: mask vertices are delivered unmodified (not expanded by the 'expand' param), shapes won't arrive 'rounded', and parametric shapes like 'star' have no shape param to read.

*Source: adobe-forum-sdk · 2025-09-01 · Tags: `masks`, `paths`, `shape-layer`, `aegp`, `dynamic-streams`, `vertices` · [View in Q&A](../qa/masks/)*

---

## How can I get the string value of an effect's ARBITRARY parameter data through ExtendScript?

One approach: (1) Add a hidden supervised checkbox parameter. (2) From JavaScript, leave a message in global scope, then toggle the checkbox to trigger USER_CHANGED_PARAM. (3) From C side, use AEGP_ExecuteScript() to check for the JavaScript message, read the string from the arb, and pass it back via AEGP_ExecuteScript(). (4) The string value is then available in JavaScript global scope. An alternative trick is to use hidden UI parameters and encode the string into their controller names, then read the property names from ExtendScript.

*Source: adobe-forum-sdk · 2025-09-01 · Tags: `arb-param`, `extendscript`, `aegp-execute-script`, `string`, `interop` · [View in Q&A](../qa/arb-param/)*

---

## How do I expand the output buffer beyond the layer size using SmartFX?

Put the expanded buffer rect in extra->output->result_rect during PreRender. AE will use that rect for the render call. Expand the in_result.max_result_rect and put the expanded result in extra->output->max_result_rect. When implementing SmartRender, account for the origin offsets between input and output worlds (input_worldP->origin_x - output_worldP->origin_x). The key consideration is what you expand relative to: the cropped buffer, the original layer size, or the max rect of the input.

```cpp
PF_LRect expanded_rect = in_result.result_rect;
expanded_rect.left -= expansion;
expanded_rect.top -= expansion;
expanded_rect.right += expansion;
expanded_rect.bottom += expansion;
extra->output->result_rect = expanded_rect;

// Also expand max_result_rect from in_result.max_result_rect
PF_LRect expanded_max = in_result.max_result_rect;
expanded_max.left -= expansion;
expanded_max.top -= expansion;
expanded_max.right += expansion;
expanded_max.bottom += expansion;
extra->output->max_result_rect = expanded_max;
```

*Source: adobe-forum-sdk · 2025-07-01 · Tags: `smartfx`, `buffer-expansion`, `pre-render`, `smart-render`, `result-rect`, `origin-offset` · [View in Q&A](../qa/smartfx/)*

---

## How to compile unicode characters (e.g., Chinese) in C++ plugin parameter names and category names for Mac AE plugins?

The name field is defined as A_char[32]. Convert your string to UTF-8 and feed the param macro the resulting string. Make sure the length of the resulting string is no more than 31 chars long (including multibyte characters), as the last char must be used for null termination.

*Source: adobe-forum-sdk · 2025-06-01 · Tags: `unicode`, `utf8`, `mac`, `xcode`, `parameter-name`, `localization` · [View in Q&A](../qa/unicode/)*

---

## How can I invoke AEGP functionality from ExtendScript (JSX) without creating a menu entry?

There is no direct way to invoke an AEGP from JavaScript without aid from a C external object. The simplest approach is to leave a flag in the JavaScript global scope and have the AEGP check for that flag on idle_hook calls (which happen 20-50 times per second). It's not immediate and synchronous, but it works easily. An alternative is using a C external object (ExternalObject in ExtendScript), though it's more complex to set up.

*Source: adobe-forum-sdk · 2025-06-01 · Tags: `aegp`, `extendscript`, `idle-hook`, `external-object`, `interop` · [View in Q&A](../qa/aegp/)*

---

## How can I synchronize two parameters so that changing one updates the other (e.g., radius and area)?

Full sync is not entirely possible due to AE's architecture. Problems include: (1) keyframable params may have contradictory interpolation settings, (2) 'current time' is ambiguous with nested comps and multi-frame rendering, (3) since AE 2015, you cannot change param values from the render thread. Workarounds: (1) Use an expression on one param, set programmatically via AEGP_SetExpression during UPDATE_PARAMS_UI. (2) Use PF_PUI_STD_CONTROL_ONLY to make a param non-keyframable. (3) Use PF_PUI_DISABLED to prevent manual changes while still allowing expressions. To get param values without downsample scaling, use AEGP_GetNewStreamValue instead of regular checkout.

*Source: adobe-forum-sdk · 2025-06-01 · Tags: `parameter-sync`, `expressions`, `update-params-ui`, `downsample`, `keyframes` · [View in Q&A](../qa/parameter-sync/)*

---

## Why does transform_world give bad results when dragging a slider, and what is the correct first argument?

The SDK documentation is wrong about putting in_data as the first argument to transform_world. The first argument is actually the effect_ref (in_data->effect_ref), but passing NULL instead works better. When using effect_ref, transform_world appears to check for interrupts, which causes it to halt the transformation during slider dragging for smoother interactive experience, but this results in incorrect output. Passing NULL avoids this interrupt check. Many suite callbacks accept NULL instead of effect_ref.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `transform-world`, `documentation-bug`, `effect-ref`, `interrupt`, `rendering` · [View in Q&A](../qa/transform-world/)*

---

## How does the AddArc function work in the Drawbot Path Suite, and why do two consecutive arcs produce unexpected shapes?

When drawing multiple arcs/circles in the same path, you need to call close() on the path after each circle. Without closing, the entire sequence is treated as one continuous path, and the renderer draws a connecting line from the end point of one arc back to the beginning of the next, creating unexpected shapes. You can have multiple closed shapes in one path, or alternatively draw each shape in separate paths with separate drawing calls.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `drawbot`, `path-suite`, `addarc`, `custom-ui`, `drawing` · [View in Q&A](../qa/drawbot/)*

---

## Why does AEGP_GetLayerNumMasks return 0 even though the layer appears to have masks?

Track mattes and masks are two different things. A track matte is one layer operating with the alpha/luma of the layer above it. A mask is a vector shape drawn on a layer with the pen tool. If you're using a track matte, AEGP_GetLayerNumMasks correctly returns 0 because there are no actual masks on that layer. The mask suite only works with vector masks drawn with the pen tool, not track mattes.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `masks`, `track-matte`, `aegp`, `mask-suite`, `layer` · [View in Q&A](../qa/masks/)*

---

## How can I chain multiple transform_world calls, using the output of one as input for the next?

Never use the same buffer as both input and output of transform_world, as it reads and overwrites simultaneously, causing corrupted output. Never overwrite the input buffer (AE caches it). For repeated transforms: (1) Create a temp buffer, (2) transform input to temp, (3) transform temp to output, (4) transform output to temp, (5) repeat as needed, (6) copy to output if last result is in temp. However, matrix multiplication is extremely cheap (20-something operations for 3x3), while each transform_world render costs millions of operations per image. Always prefer multiplying matrices and doing a single render.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `transform-world`, `matrix-multiplication`, `buffer-management`, `performance`, `rendering` · [View in Q&A](../qa/transform-world/)*

---

## How can two different plugin types share arbitrary (ARB) parameter data?

AE blocks expression connections between ARB parameters of different effect types, assuming they can't be guaranteed to have the same structure. On the C side, use AEGP_GetNewStreamValue to read ARB values from other effects. To find the other effect, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH. For consistent identification of a specific effect instance, store an ID in sequence data and access it via AEGP_EffectCallGeneric, or rename an invisible param (though that doesn't survive project reload).

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `arb-param`, `inter-plugin`, `aegp`, `stream-value`, `effect-identification` · [View in Q&A](../qa/arb-param/)*

---

## What is the difference between thread_indexL and i in iterate_generic?

The 'i' parameter is the iteration index (not sequential - may be reordered to minimize CPU cache conflicts), while thread_indexL is the thread index. With 4 threads and 8 iterations, i values may arrive as 6,0,2,7,4,5,1,3 on threads 3,0,1,3,2,2,0,1. They only match when iterations equals threads (e.g., PF_Iterations_ONCE_PER_PROCESSOR). The thread count doesn't change during an AE session. For image processing, it's better NOT to use PF_Iterations_ONCE_PER_PROCESSOR because some threads will finish early and sit idle. Using N iterations ensures minimal CPU power loss. Never call iterate_generic from within iterate_generic, and don't call suites from non-thread-0 threads.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `iterate-generic`, `multithreading`, `thread-index`, `iteration`, `performance` · [View in Q&A](../qa/iterate-generic/)*

---

## How do I implement matrix transformation with a custom pivot/anchor point in the AE SDK?

The anchor point goes in the place of position with negative values, but you can't do it all in one matrix. Create three separate matrices: (1) anchor point translation matrix (negative of anchor position), (2) rotation/scale matrix, (3) position translation matrix. Multiply them together: anchor * rotation * position. AE doesn't offer built-in matrix handling functions, but the 'Artie' sample contains code for 4x4 matrix multiplication. For 3x3, use standard nested loop multiplication.

```cpp
for (int i = 0; i < a; i++)
    for (int j = 0; j < d; j++) {
        Mat3[i][j] = 0;
        for (int k = 0; k < c; k++)
            Mat3[i][j] += Mat1[i][k] * Mat2[k][j];
    }
```

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `matrix`, `transform`, `pivot-point`, `anchor`, `rotation`, `scale` · [View in Q&A](../qa/matrix/)*

---

## What are the best practices for using global variables in AE plugins?

Global variables are generally frowned upon but widely used. Considerations: (1) std::vectors allocate system memory directly rather than through AE's RAM management, which can cause AE to exceed desired memory usage. (2) Globals are shared by ALL instances of your effect. (3) Memory is only deallocated when AE shuts down, even after global_setdown, so you can't release it through AE's suites. The recommended approach is to allocate memory and store it in a pointer on your global_data handle. For std::vector, you can write a custom allocator that uses AE memory allocation. If the data is constant and read-only, globals are more acceptable.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `global-variables`, `memory-management`, `global-data`, `std-vector`, `allocator` · [View in Q&A](../qa/global-variables/)*

---

## How can I debug a plugin when it's rendered through AE Render Queue or Adobe Media Encoder?

Adobe Media Encoder runs an invisible/separate AE instance for rendering. You can see these processes in the task manager. To debug, attach your debugger to the already running AE process spawned by AME. In Visual Studio, use Debug > Attach to Process. In Xcode, use Debug > Attach to Process by PID or Name.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `debugging`, `render-queue`, `media-encoder`, `attach-debugger`, `ame` · [View in Q&A](../qa/debugging/)*

---

## Can I write a 3D importer plugin for After Effects that behaves like the .obj importer with 3D transforms and depth buffer?

No, this is not possible. Importer AEGPs (known as IO and FBIO) only parse a requested frame and pass it to AE as an image. 3D behavior is handled by the Artizan plugin which does full comp rendering. You could try writing an Artizan plugin, but you'd have to handle all other rendering as well, making it impractical.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `3d-import`, `io-plugin`, `fbio`, `artizan`, `importer` · [View in Q&A](../qa/3d-import/)*

---

## How can I create a progress bar or continuously animated element in the ECW (Effect Control Window)?

ECW UI elements get idle calls only when the cursor is within their borders. For updates regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows() during that event. It's a brute-force approach but gets the job done. For softer redraws when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `ecw`, `progress-bar`, `idle-hook`, `refresh`, `custom-ui`, `animation` · [View in Q&A](../qa/ecw/)*

---

## How do I get the name of the currently selected layer using the AE SDK?

Use AEGP_GetNewCollectionFromCompSelection() to get the selection, then traverse it with AEGP_GetCollectionNumItems() and AEGP_GetCollectionItemByIndex(). Each item returns an AEGP_CollectionItemV2 struct with a type field. If type is layer, the 'layer' union member contains a valid AEGP_LayerH. Then use AEGP_GetLayerName() with that handle. Note there may be multiple selected layers.

```cpp
typedef struct {
    AEGP_CollectionItemType type;
    union {
        AEGP_LayerCollectionItem layer;
        AEGP_MaskCollectionItem mask;
        AEGP_EffectCollectionItem effect;
        AEGP_StreamCollectionItem stream;
        AEGP_MaskVertexCollectionItem mask_vertex;
        AEGP_KeyframeCollectionItem keyframe;
    } u;
    AEGP_StreamRefH stream_refH;
} AEGP_CollectionItemV2;
```

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `aegp`, `selection`, `layer-name`, `collection`, `layer-handle` · [View in Q&A](../qa/aegp/)*

---

## How can I set a parameter value when an effect is first applied (during ParamSetup or SequenceSetup)?

ParamSetup is called once per session per effect type, not per instance, so you can't set instance-specific values there. SequenceSetup also lacks a specific instance association - the params array contains junk data and acquiring an AEGP_EffectRef will crash. The solution is to set a flag in new sequence data saying 'this is a brand new instance', then check that flag during idle_hook or UPDATE_PARAMS_UI and make changes accordingly. For unique instance IDs, use sequence data as each sequence setup call is triggered only once per instance when created.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `param-setup`, `sequence-setup`, `instance-id`, `initialization`, `sequence-data` · [View in Q&A](../qa/param-setup/)*

---

## Is there a way to set custom layer icons in the AE timeline via AEGP?

There is no API for changing layer icons. The AE interface is an OS window, so theoretically hacky approaches could work, but there's no supported way to do this through the SDK.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `aegp`, `layer-icon`, `ui-customization`, `unsupported` · [View in Q&A](../qa/aegp/)*

---

## Why does my std::map of install keys and matchnames return wrong effect keys?

The map was declared as std::map<A_char, A_long> which stores only a single A_char character, not the full string. The map key type should be a string type (like std::string) or at least an array of A_char to store the full matchname. Using just A_char means all matchnames starting with the same character would overwrite each other.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `aegp`, `matchname`, `install-key`, `string-handling`, `std-map` · [View in Q&A](../qa/aegp/)*

---

## What are best practices for AE SDK string conversions between A_char, A_UTF16Char, and std::string?

Use MultiByteToWideChar with NULL as the last parameter to predict the required buffer size. For error handling, only use PF_Err_OUT_OF_MEMORY as the error return, because PF_Err_INTERNAL_STRUCT_DAMAGED tells AE the effect instance is bad and AE won't talk to it again. PF_Err_OUT_OF_MEMORY is the polite way of telling AE a certain operation has failed and its results should be ignored.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `string-conversion`, `utf16`, `utf8`, `error-handling`, `buffer-size` · [View in Q&A](../qa/string-conversion/)*

---

## Is AEGP_RenderAndCheckoutLayerFrame_Async safe to use for rendering multiple frames?

There is a known bug with the async version of RenderAndCheckout related to memory releasing. The issue was discussed between several developers and the AE team. All developers reverted back to the synchronous version, doing their best to render one frame at a time on each idle cycle to avoid making the UI laggy. The frame receipt cannot be properly checked in when using the async version.

*Source: adobe-forum-sdk · 2024-03-01 · Tags: `async-render`, `render-suite`, `memory-leak`, `bug`, `frame-checkout` · [View in Q&A](../qa/async-render/)*

---

## Can I use iterate_generic to parallelize transform_world calls?

No, transform_world is not safe to call from utility threads. AE has main threads (UI and render) and utility threads (used by iteration suite). Interaction with AE is only allowed from main threads. transform_world is already internally multithreaded, so calling it from parallel utility threads won't help and may cause bugs/crashes. Very few API callbacks are safe from non-main threads (e.g., subpixel sample callbacks, but suites must be acquired in advance from main threads). If you need parallel image transformations, write your own transform function with nearest-neighbor sampling that can safely run on multiple threads.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `iterate-generic`, `transform-world`, `multithreading`, `thread-safety`, `performance` · [View in Q&A](../qa/iterate-generic/)*

---

## How can I store custom AEGP data persistently in a comp or project?

There's no facility designed for persistent AEGP data storage, but several workarounds exist: (1) Use layer and comp comments - rarely displayed in UI and almost never used by users. (2) Add a locked null layer named descriptively (e.g., 'my stuff, do not delete') and use its comments field or marker comments. (3) Create a folder in the AE project with sub-folders whose names encode comp IDs and data. This creates minimal project clutter.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `aegp`, `persistent-data`, `project-storage`, `comments`, `workaround` · [View in Q&A](../qa/aegp/)*

---

## How can I make an AEGP plugin invisible (not appear in the Window menu)?

Simply don't use AEGP_InsertMenuCommand or AEGP_RegisterCommandHook. These are only needed to create a menu entry. Without them, the AEGP will load and run in the background without any UI presence.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `aegp`, `invisible-plugin`, `menu`, `background-plugin` · [View in Q&A](../qa/aegp/)*

---

## Why is sequence data different between PF_Cmd_EVENT (UI thread) and SmartPreRender (render thread)?

Sequence data is separate on the render and UI threads by design. The render threads are free to render asynchronously without having their data change from another thread. Global data IS shared between render and UI threads, though it's not instance-specific. To share per-instance data: store data in global data keyed by instance identifiers (comp item ID + layer ID + effect index on layer), protected by a mutex. Comp item IDs don't change during a session. Layer IDs don't change on reorder but may change on copy/paste. Effect index changes when moved in the stack.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `sequence-data`, `threading`, `ui-thread`, `render-thread`, `global-data`, `mutex` · [View in Q&A](../qa/sequence-data/)*

---

## Why doesn't my Arbitrary parameter respect CANNOT_TIME_VARY and COLLAPSE_TWIRL flags?

The flags are placed in the wrong argument position in PF_ADD_ARBITRARY2. PF_ParamFlag_CANNOT_TIME_VARY and PF_ParamFlag_COLLAPSE_TWIRLY are param flags, not param UI flags. They should go in the param_flags argument (one position before where you placed them), not in the ui_flags argument where PF_PUI_CONTROL belongs.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `arb-param`, `param-flags`, `ui-flags`, `cannot-time-vary`, `collapse-twirl` · [View in Q&A](../qa/arb-param/)*

---

## Is it possible to hot-reload AE plugins during development without restarting AE?

Hot reload (called 'Edit and Continue' in Visual Studio) was common on 32-bit builds but became less reliable on 64-bit. Even with hot reloading, there are limitations: you can't change ParamSetup because AE calls it only once per session when the plugin is first loaded. Starting AE via the debugger (Visual Studio or Xcode) tends to launch faster than normal startup. Each time you rebuild C++ code, you generally need to restart AE, unlike ExtendScript or CEP where reloads are possible without full restart.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `hot-reload`, `development-workflow`, `debugging`, `visual-studio`, `xcode` · [View in Q&A](../qa/hot-reload/)*

---

## How can I set up an AE plugin to asynchronously receive video frames from an external pipe or ML model?

AE's rendering pipeline is built around random frame access, not sequential processing. Set up a separate C++ thread (nothing to do with AE SDK) for the async pipe, and use a mutex to share data between the pipe thread and AE's render calls. Use AEGP_CauseIdleRoutinesToBeCalled() to trigger immediate communication. For sequential ML models requiring multiple frames: (1) Have an AEGP monitor the project and pre-fetch frames using AEGP_CheckoutOrRender_ItemFrame_AsyncManager, caching results to disk. (2) If the user jumps past cached results, stall the render until caching catches up. Consider the tradeoff between fast sequential pre-processing (like AE's camera tracker) vs slower random-access processing.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `async`, `pipe`, `ml-model`, `threading`, `frame-caching`, `sequential-processing` · [View in Q&A](../qa/async/)*

---

## How do I work with multiple arbitrary (ARB) parameters in an AE effect?

Each ARB param is its own thing with its own routines. Use ARB_REFCON to route handling: it's pointer-sized and can point to a function or class instance that handles that specific param. This eliminates the need for switch statements. Each ARB needs a unique disk ID and a separately allocated default handle. PF_CustomUIInfo registration only needs to be done once per PARAM_SETUP call (not per param). For a scalable approach, create a base class for generic ARB handling and override specific methods per param type. The refconPV content is entirely up to you - AE is indifferent to it.

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

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `arb-param`, `multiple-params`, `refcon`, `custom-ui`, `class-design`, `param-routing` · [View in Q&A](../qa/arb-param/)*

---

## Can the AE SDK provide post-processed shape path vertices (after Offset Path, Zig Zag, Wiggle modifiers)?

AE doesn't provide post-processed paths. You can only read the raw parameter streams and then try to emulate the processed path yourself. There is no way to get the final computed path after shape modifiers like Offset, Zig Zag, or Wiggle have been applied.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `shape-layer`, `path-vertices`, `shape-modifiers`, `aegp`, `limitation` · [View in Q&A](../qa/shape-layer/)*

---

## How can I get decoded video frames in sequential order for an AE plugin that needs sequential processing (e.g., face detection)?

Use AEGP_RenderAndCheckoutFrame (from AEGP_RenderSuite4) to request any project item's image at any time in any order. AE's work scheme revolves around random frame access, so sequential processing works against the user experience. However, plugins like AE's camera tracker handle this by doing sequential pre-processing in the background while showing progress without blocking the UI. Consider the tradeoff: (1) Fast sequential pre-processing at the cost of usage intuitiveness, or (2) Slower random-access processing with the benefit of not waiting for the whole pre-process to finish.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `sequential-frames`, `render-suite`, `face-detection`, `pre-processing`, `background-processing` · [View in Q&A](../qa/sequential-frames/)*

---

## What is the correct workflow for storing custom data in an arbitrary (ARB) parameter for custom UI interactions?

The workflow should be: user interacts with UI -> data is stored in a LOCAL array -> local array is serialized and stored in an ARB -> AE sees new value and invalidates cached frames. Before displaying UI, read the value from the ARB, deserialize into your local array, and draw accordingly (this ensures undo/redo reflects correctly). Serializing means writing all data into one continuous block of AE-allocated memory. You don't need to convert to a string - use whichever method works for collecting data in and reconstructing out. This is referred to as 'flattening' and 'unflattening' in the AE SDK.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `arb-param`, `serialization`, `custom-ui`, `undo-redo`, `flattening`, `ecw` · [View in Q&A](../qa/arb-param/)*

---

## Why is PF_Cmd_SEQUENCE_FLATTEN called when an effect is first applied, and why are Sequence Resetup/Setdown called multiple times on different threads?

This is correct behavior when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set. AE flattens sequence data so it can be copied (for render thread copies). Since AE 2015, rendering uses separate threads with separate project copies. Resetup may receive unflat data, so tag your data to indicate flat/unflat state. Use PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA to allow AE to get the flat copy without losing the original unflat one, reducing unnecessary resetup calls. During sequence setup/resetup, don't assume the plugin can find its own layer/comp - AE sometimes constructs the instance before associating it to a layer. For unique instance IDs, flag sequence data as 'requires checking' on resetup, then scan for duplicates when the instance can be located.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `sequence-data`, `flatten`, `resetup`, `threading`, `mfr`, `instance-id` · [View in Q&A](../qa/sequence-data/)*

---

## Should I use multithreading (iterate_generic) or MFR (Multi-Frame Rendering) in my plugin, or both?

Use both. MFR and multithreading can coexist because not all steps in the single-frame rendering pipeline are multi-threadable. MFR shines for sequential parts, while multithreaded parts (via iterate callbacks) are critical for performance. Mixing MFR with MT doesn't necessarily cause competition because MT-friendly code tends to be CPU cache friendly, while other parts may leave the CPU waiting for RAM. When using iteration callbacks, AE may prioritize between MFR and MT threads. The AE engineering team tested these scenarios extensively.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `mfr`, `multithreading`, `multi-frame-rendering`, `iterate-generic`, `performance` · [View in Q&A](../qa/mfr/)*

---

## What is the best practice to spawn a window with text input from an effect's parameters?

Two separate concerns: (1) Getting user input: Use AEGP_ExecuteScript to launch a JavaScript prompt - it takes text input and returns whether the user hit OK or Cancel. For fancier UI, use OS-level windows (no third-party library needed). (2) Storing the data: Use an ARB parameter which allows undo/redo. You can have a button param launch the prompt and store the string in a hidden ARB, or combine both into one ARB with a custom UI that shows the current string AND is clickable.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `text-input`, `dialog`, `aegp-execute-script`, `arb-param`, `custom-ui`, `button-param` · [View in Q&A](../qa/text-input/)*

---

## Why doesn't PF_Cmd_UPDATE_PARAMS_UI properly update parameter values?

Two issues: (1) During UPDATE_PARAMS_UI, you should only change appearance (hidden, disabled, etc.), not values. Value changes don't play well with AE's instance application scheme. Set proper default values during PARAM_SETUP. (2) Never modify values directly on the original params array. Instead, make a copy of the param struct, modify the copy, and pass it back via PF_UpdateParamUI. Also ensure out_data->out_flags includes PF_OutFlag_REFRESH_UI. See the 'Supervisor' SDK sample project for correct implementation.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `update-params-ui`, `param-update`, `refresh-ui`, `supervisor-sample`, `pf-update-param-ui` · [View in Q&A](../qa/update-params-ui/)*

---

## Is there a way to enable the alpha channel in a COLOR parameter's color picker?

The standard AE color parameter does not support alpha. PF_AppColorPickerDialog might show the system dialog which could have an alpha component, but probably doesn't. The practical solution is to implement your own color picking dialog launched via param supervision or a custom UI, or simply add a separate opacity parameter alongside the color parameter.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `color-param`, `alpha`, `color-picker`, `custom-dialog`, `opacity` · [View in Q&A](../qa/color-param/)*

---

## How can I programmatically set keyframes on effect parameters when the effect is first applied?

Don't set keyframes during GlobalSetup - there's no effect instance yet (effect_ref is garbage). Instead, try setting keyframes on the first call to UPDATE_PARAMS_UI. To set a value, first get the current stream value using AEGP_GetNewStreamValue, modify the value in the returned AEGP_StreamValue2 struct (e.g., valueP.val.one_d = 20), then use the keyframe suite to add keyframes. If the effect is added via script, it's simpler to set keyframes from the script instead.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `keyframes`, `stream-value`, `global-setup`, `update-params-ui`, `initialization` · [View in Q&A](../qa/keyframes/)*

---

## How can I detect when an effect is removed from a layer?

There's no direct API for detection. PF_Cmd_SEQUENCE_SETDOWN is called when AE destroys an instance, but only when the deletion is out of the undo stack (100 steps later) or on project close/purge. Approaches: (1) Scan the project during idle_hook, catalog effect instances, and detect when one disappears. Not immediate. (2) Monitor 'cut' and 'delete' via command hooks, but this is unreliable - redo operations don't trigger menu commands. Important considerations: 'cut' puts the effect in limbo (can be pasted back or undone). For the idle_hook approach, note that in_data is not available in IdleHook - you need to acquire suites differently, using the SPBasicSuite passed during AEGP initialization.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `effect-removal`, `sequence-setdown`, `idle-hook`, `command-hook`, `undo-stack` · [View in Q&A](../qa/effect-removal/)*

---

## How can I make the idle_hook get called more frequently for near-real-time communication?

Use AEGP_CauseIdleRoutinesToBeCalled() to trigger more frequent idle calls. Setting *max_sleepPL directly doesn't reliably work as it gets reset to its default value. Note that on macOS the idle hook caps at around 30 calls per second, while on Windows it can reach 60.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `idle-hook`, `max-sleep`, `real-time`, `cause-idle`, `frame-rate` · [View in Q&A](../qa/idle-hook/)*

---

## Can I save sequence data during PF_Cmd_SEQUENCE_SETDOWN?

No. Sequence setdown is AE's signal to destruct and free the data, not to save it. The data saved with the project is the last handle used or flattened. To have data saved with the project, set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING during global setup to receive PF_Cmd_SEQUENCE_FLATTEN calls, where you serialize your data into a flat handle that AE will store.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `sequence-data`, `sequence-setdown`, `flatten`, `project-save`, `persistence` · [View in Q&A](../qa/sequence-data/)*

---

## What is the simplest way to make a settings-only effect that doesn't modify the layer visually?

Several options: (1) Use AE's utils->Copy() function to copy the input buffer to the output buffer - it's fast. (2) Use PF_OutFlag_AUDIO_EFFECT_ONLY to make AE skip image rendering entirely (but you'll need to implement audio pass-through). (3) Setting the output rect size to 0 or negative during pre-render might cause AE to skip the render, though this isn't fully reliable.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `pass-through`, `settings-only`, `copy-buffer`, `audio-effect-only`, `identity` · [View in Q&A](../qa/pass-through/)*

---

## How can I prevent my effect from being added more than once to the same layer?

There's no outflag for this. The approach: (1) During global setup, find your effect's AEGP_InstalledEffectKey by iterating installed effects with AEGP_GetNumInstalledEffects, AEGP_GetNextInstalledEffect, and AEGP_GetEffectMatchName. (2) During UPDATE_PARAMS_UI (guaranteed to be called after applying), scan the effect's layer and check each effect's install key using AEGP_GetInstalledKeyFromLayerEffect. (3) If duplicates exist, alert the user, delete/disable the redundant, or skip rendering by copying input to output. You can also try AEGP_DisableCommand, but users can still duplicate or copy/paste the effect.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `single-instance`, `effect-detection`, `install-key`, `matchname`, `update-params-ui` · [View in Q&A](../qa/single-instance/)*

---

## Why does memory usage spike to 6+ GB when I add PF_OutFlag_NON_PARAM_VARY to my plugin?

This is normal AE behavior - it's hoarding cached frames. PF_OutFlag_NON_PARAM_VARY tells AE the output varies independently of parameters, so AE caches each unique frame. Verify by purging AE's memory (Edit > Purge > All Memory) - if RAM consumption drops to expected levels, everything is working correctly. AE will release cached memory when it's needed for new frame renders. PF_OutFlag_WIDE_TIME_INPUT is only needed if you use a layer selector and sample it at different times. Implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `memory`, `caching`, `non-param-vary`, `frame-setup`, `frame-setdown`, `purge` · [View in Q&A](../qa/memory/)*

---

## How do I get the mouse position in the comp window from a plugin?

The mouse coordinates are part of the data passed to the plugin on a UI event call. Look at the 'CCU' SDK sample project - it shows how to get the mouse click location in comp window coordinates and how to convert those into layer source coordinates. This is only available during custom UI event callbacks.

*Source: adobe-forum-sdk · 2022-01-01 · Tags: `mouse-position`, `comp-window`, `custom-ui`, `ccu-sample`, `coordinates` · [View in Q&A](../qa/mouse-position/)*

---

## How can I force a re-render from a separate thread (e.g., when receiving WebSocket messages)?

It's only possible via idle_hook, which runs on the UI thread (30-50 times per second). Check your other thread for messages during idle events. Options to force re-render: (1) Change a hidden/invisible parameter value - this forces AE to re-render. (2) Call the command number to purge the cache. (3) Call AEGP_EffectCallGeneric to have the effect instance act on itself. You cannot set out_data->out_flags from a separate thread - it must happen on the main UI thread during an AE callback.

*Source: adobe-forum-sdk · 2022-01-01 · Tags: `force-rerender`, `idle-hook`, `websocket`, `threading`, `ui-thread`, `hidden-param` · [View in Q&A](../qa/force-rerender/)*

---

## After upgrading a plugin from CS5 SDK to the 2021 SDK with MFR support, AE crashes when saving the project. How to debug?

The crash is likely related to sequence data flattening. Check if PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set in global setup - if so, you must handle PF_Cmd_SEQUENCE_FLATTEN properly. When modifying sequence data, be careful about whether you're changing data in the same handle or allocating a new one. Debug steps: (1) Purge Xcode intermediates and clean rebuild. (2) Test SDK sample projects to verify they work. (3) Comment out parts of your plugin to isolate the issue. (4) Copy your code onto a fresh sample project.

*Source: adobe-forum-sdk · 2021-01-01 · Tags: `sdk-upgrade`, `crash`, `save`, `sequence-flatten`, `mfr`, `debugging` · [View in Q&A](../qa/sdk-upgrade/)*

---

## How can I identify a specific effect instance using AEGP_EffectCallGeneric and PF_Cmd_COMPLETELY_GENERAL?

AEGP_EffectCallGeneric has a limitation: you cannot call it from an effect to another instance of the same effect type. Even bouncing through a separate AEGP via a custom suite doesn't work if the call chain originates from the same effect type. For instance identification, alternatives include: (1) Put an ID in sequence data and access via AEGP_EffectCallGeneric from a different effect type. (2) Change a hidden param's name to serve as identifier using AEGP_SetStreamName() (doesn't survive save/load). (3) Use a separate AEGP to handle identification during idle processing.

*Source: adobe-forum-sdk · 2018-01-01 · Tags: `effect-instance`, `identification`, `aegp`, `completely-general`, `sequence-data` · [View in Q&A](../qa/effect-instance/)*

---

## Is there an API to get the layer comment field from C++ code?

There is no C API for getting layer comments. Use AEGP_ExecuteScript() to run ExtendScript that reads the layer comment and retrieve the result back to the C side.

*Source: adobe-forum-sdk · 2015-01-01 · Tags: `layer-comment`, `aegp`, `execute-script`, `extendscript` · [View in Q&A](../qa/layer-comment/)*

---

## How can I detect when an AE project is saved, from a plugin?

There's no reliable pre-save notification. Two approaches: (1) Use command_hook to catch save events, but it's not 100% bulletproof (sometimes saves happen without triggering the hook). (2) Implement an idle_hook and check if the project dirty flag changes from dirty to clean - if so, the user has saved. For effect plugins specifically, use sequence data with PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING, which causes AE to call your effect with PF_Cmd_SEQUENCE_FLATTEN on save.

*Source: adobe-forum-sdk · 2015-01-01 · Tags: `project-save`, `idle-hook`, `command-hook`, `dirty-flag`, `sequence-flatten` · [View in Q&A](../qa/project-save/)*

---

## How can I hide an effect parameter in After Effects SDK?

Set the AEGP_DynStreamFlag_HIDDEN flag during PF_Cmd_UPDATE_PARAMS_UI. It should get called when a project loads and the effect is shown for the first time. Note that PF_PUI_NO_ECW_UI in ui_flags only removes it from the effect window but not from the timeline. There is a known AE bug where duplicating an effect may cause the hidden parameter's owner-drawn portion to still appear in some cases.

*Source: adobe-forum-sdk · 2011-01-01 · Tags: `parameter`, `hidden`, `update-params-ui`, `dynamic-stream`, `ui-flags` · [View in Q&A](../qa/parameter/)*

---

## What is the minimal code needed for SmartFX PreRender and SmartRender?

In PreRender: checkout the layer with channel_mask set to PF_ChannelMask_ARGB, then union the result rects. In SmartRender: checkout layer pixels and output, check the bit depth via extra->input->bitdepth (8, 16, or 32) and process accordingly. If you don't need to process anything, use PF_COPY(inputP, outputP, NULL, NULL) which is bit-depth independent. Always check in params with ERR2 regardless of error state. Don't forget to set the correct flags in global setup.

```cpp
// PreRender
PF_RenderRequest req = extra->input->output_request;
PF_CheckoutResult in_result;
req.channel_mask = PF_ChannelMask_ARGB;
ERR(extra->cb->checkout_layer(in_data->effect_ref, EFFECT_INPUT, EFFECT_INPUT, &req,
    in_data->current_time, in_data->time_step, in_data->time_scale, &in_result));
UnionLRect(&in_result.result_rect, &extra->output->result_rect);
UnionLRect(&in_result.max_result_rect, &extra->output->max_result_rect);

// SmartRender
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, EFFECT_INPUT, &input_worldP));
ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));
switch (extra->input->bitdepth) {
    case 8: /* process 8 bit */ break;
    case 16: /* process 16 bit */ break;
    case 32: /* process 32 bit */ break;
}
ERR2(PF_CHECKIN_PARAM(in_data, &paramWhatever));
```

*Source: adobe-forum-sdk · 2010-01-01 · Tags: `smartfx`, `smart-render`, `pre-render`, `minimal`, `bit-depth`, `boilerplate` · [View in Q&A](../qa/smartfx/)*

---
