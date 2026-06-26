# Q&A: aegp

**630 entries** tagged with `aegp`.

---

## How do you toggle parameter visibility (show/hide) in an AE plugin?

Use AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to toggle visibility. The approach using PF_PUI_INVISIBLE with ui_flags and PF_UpdateParamUI is unreliable. Instead, use the AEGP stream suites: get the effect ref via AEGP_GetNewEffectForEffect, get the stream via AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag. Note this only works in AE (check in_data->appl_id != 'PrMr'), not Premiere.

```cpp
static PF_Err
changeParamVisibility(PF_InData *in_data, PF_OutData *out_data,
                      PF_ParamDef *paramsDef, PF_ParamIndex paramIndex,
                      PF_Boolean paramVisibleB)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    global_dataP globP = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr') {
        AEGP_EffectRefH meH = NULL;
        AEGP_StreamRefH currStreamH = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex, &currStreamH));
        if (meH && currStreamH) {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
    }
    return err;
}
```

*Contributors: [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2022-12-07 · Tags: `param-visibility`, `aegp`, `dynamic-stream`, `custom-ui`, `pipl`*

---

## Why does Media Encoder not render plugins correctly or fail to load AEGP plugins?

Media Encoder does not load AEGP plugins. If your effect plugin relies on a companion AEGP plugin for rendering, it will fail in Media Encoder. Issues may also be related to static global variables or elements defined in GlobalData/GlobalSetup. The AEGP functionality would need to be incorporated directly into the effect plugin, or you'd need a standalone/command-line tool alternative.

*Contributors: [**gabgren**](../contributors/gabgren/), [**tlafo**](../contributors/tlafo/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2024-02-27 · Tags: `media-encoder`, `aegp`, `render-issues`, `global-data`*

---

## What causes AEGP_StartUndoGroup(null) to crash in AE 2025?

Starting from AE 2025 beta, passing null/NULL to AEGP_StartUndoGroup causes a crash. Use an empty string "" instead. AEGP_StartUndoGroup("") works correctly and behaves as expected (no entry in the undo stack). This regression has occurred before in earlier AE betas but was corrected.

```cpp
// CRASHES in AE 2025:
suites.UtilitySuite5()->AEGP_StartUndoGroup(null);

// WORKS:
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Contributors: [**James Whiffin**](../contributors/james-whiffin/), [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: adobe-plugin-devs · 2024-12-29 · Tags: `undo`, `aegp`, `crash`, `ae-2025`, `regression`*

---

## How do you share state across multiple effect plugins in the same AE session?

Options: (1) Use a companion AEGP plugin that communicates with all your effect plugins. (2) Use dlopen/symbol loading: define a global variable in one plugin and have other plugins dynamically load a function from it to get/set the shared state. (3) Use PlugPlug DLL for IPC communication. For state that should reset on next AE launch (not persist to prefs), the dlopen approach or AEGP companion are preferred.

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2025-03-30 · Tags: `inter-plugin-communication`, `global-state`, `dlopen`, `aegp`, `shared-data`*

---

## How do you get the AE/Premiere version number programmatically?

in_data->version major/minor are not reliably updated between AE versions. Options: (1) Use ExtendScript via AEGP_ExecuteScript with 'app.version' - but may fail with 'cannot run script while modal dialog waiting'. (2) Use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP to get the AE install folder path, then parse the version from the folder name or binary metadata. (3) For plugins with Premiere GPU entry points, PPixSuite's appinfo provides version data (format like '24.3', add 2000 for year).

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2025-03-25 · Tags: `version-detection`, `aegp`, `extendscript`, `premiere`, `compatibility`*

---

## What suites should you avoid calling from Death Hook?

You shouldn't rely on any suite being available in the Death Hook. Attempting to use suites like UtilitySuite's ReportInfo from Death Hook can cause unhandled exceptions. By that point in the shutdown process, many suites may already be torn down.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: adobe-plugin-devs · 2024-03-30 · Tags: `death-hook`, `suites`, `shutdown`, `aegp`*

---

## How do you serialize/capture arb data from arbitrary third-party plugins?

Arb data is serialized via PF_Cmd_Arbitrary_Callback using flatten/unflatten functions, similar to sequence data. The flatten/unflatten versions are not accessible via AEGP functions. For plugins with pure stack-variable arb data (like Curves), you can reinterpret_cast the handle as char* and serialize directly. However, plugins with pointers in their arb structs (like Liquify) cannot be serialized this way because deserialized pointer addresses won't be valid. ExtendScript may offer a path through stream values, but it's at the limit of what's possible.

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**James Whiffin**](../contributors/james-whiffin/) · Source: adobe-plugin-devs · 2025-02-15 · Tags: `arb-data`, `serialization`, `flatten`, `unflatten`, `aegp`*

---

## What does AEGP_GetLayerToWorldXform actually represent?

AEGP_GetLayerToWorldXform translates XY coordinates from the layer origin to XYZ in composition space. 'World' means composition space. You can use the returned matrix for simple layer-to-comp coordinate conversion, or use it to construct a full 'render view' matrix from the camera's point of view.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-11-01 · Tags: `aegp`, `transform`, `matrix`, `coordinate-space`, `layer-to-comp`*

---

## How can I checkout paths or masks from a different layer in the AE SDK?

You can fetch any mask using AEGP_MaskSuite6 and read its vertices using AEGP_MaskOutlineSuite3. Shapes can also be parsed using AEGP_MaskOutlineSuite3 by pushing shape data into the same suite callbacks. To find a shape layer's shape param, you need to traverse the layer's dynamic streams. However, there are limitations: mask vertices are delivered unmodified (not expanded by the 'expand' param), shapes won't arrive 'rounded', and parametric shapes like 'star' have no shape param to read.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-09-01 · Tags: `masks`, `paths`, `shape-layer`, `aegp`, `dynamic-streams`, `vertices`*

---

## How can I invoke AEGP functionality from ExtendScript (JSX) without creating a menu entry?

There is no direct way to invoke an AEGP from JavaScript without aid from a C external object. The simplest approach is to leave a flag in the JavaScript global scope and have the AEGP check for that flag on idle_hook calls (which happen 20-50 times per second). It's not immediate and synchronous, but it works easily. An alternative is using a C external object (ExternalObject in ExtendScript), though it's more complex to set up.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/), [**Rewind_**](../contributors/rewind/) · Source: adobe-forum-sdk · 2025-06-01 · Tags: `aegp`, `extendscript`, `idle-hook`, `external-object`, `interop`*

---

## Why does AEGP_GetLayerNumMasks return 0 even though the layer appears to have masks?

Track mattes and masks are two different things. A track matte is one layer operating with the alpha/luma of the layer above it. A mask is a vector shape drawn on a layer with the pen tool. If you're using a track matte, AEGP_GetLayerNumMasks correctly returns 0 because there are no actual masks on that layer. The mask suite only works with vector masks drawn with the pen tool, not track mattes.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `masks`, `track-matte`, `aegp`, `mask-suite`, `layer`*

---

## How can two different plugin types share arbitrary (ARB) parameter data?

AE blocks expression connections between ARB parameters of different effect types, assuming they can't be guaranteed to have the same structure. On the C side, use AEGP_GetNewStreamValue to read ARB values from other effects. To find the other effect, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH. For consistent identification of a specific effect instance, store an ID in sequence data and access it via AEGP_EffectCallGeneric, or rename an invisible param (though that doesn't survive project reload).

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `arb-param`, `inter-plugin`, `aegp`, `stream-value`, `effect-identification`*

---

## How can I identify a specific effect instance using AEGP_EffectCallGeneric and PF_Cmd_COMPLETELY_GENERAL?

AEGP_EffectCallGeneric has a limitation: you cannot call it from an effect to another instance of the same effect type. Even bouncing through a separate AEGP via a custom suite doesn't work if the call chain originates from the same effect type. For instance identification, alternatives include: (1) Put an ID in sequence data and access via AEGP_EffectCallGeneric from a different effect type. (2) Change a hidden param's name to serve as identifier using AEGP_SetStreamName() (doesn't survive save/load). (3) Use a separate AEGP to handle identification during idle processing.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2018-01-01 · Tags: `effect-instance`, `identification`, `aegp`, `completely-general`, `sequence-data`*

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

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `aegp`, `selection`, `layer-name`, `collection`, `layer-handle`*

---

## Is there a way to set custom layer icons in the AE timeline via AEGP?

There is no API for changing layer icons. The AE interface is an OS window, so theoretically hacky approaches could work, but there's no supported way to do this through the SDK.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `aegp`, `layer-icon`, `ui-customization`, `unsupported`*

---

## Why does my std::map of install keys and matchnames return wrong effect keys?

The map was declared as std::map<A_char, A_long> which stores only a single A_char character, not the full string. The map key type should be a string type (like std::string) or at least an array of A_char to store the full matchname. Using just A_char means all matchnames starting with the same character would overwrite each other.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `aegp`, `matchname`, `install-key`, `string-handling`, `std-map`*

---

## How can I store custom AEGP data persistently in a comp or project?

There's no facility designed for persistent AEGP data storage, but several workarounds exist: (1) Use layer and comp comments - rarely displayed in UI and almost never used by users. (2) Add a locked null layer named descriptively (e.g., 'my stuff, do not delete') and use its comments field or marker comments. (3) Create a folder in the AE project with sub-folders whose names encode comp IDs and data. This creates minimal project clutter.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `aegp`, `persistent-data`, `project-storage`, `comments`, `workaround`*

---

## How can I make an AEGP plugin invisible (not appear in the Window menu)?

Simply don't use AEGP_InsertMenuCommand or AEGP_RegisterCommandHook. These are only needed to create a menu entry. Without them, the AEGP will load and run in the background without any UI presence.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `aegp`, `invisible-plugin`, `menu`, `background-plugin`*

---

## Can the AE SDK provide post-processed shape path vertices (after Offset Path, Zig Zag, Wiggle modifiers)?

AE doesn't provide post-processed paths. You can only read the raw parameter streams and then try to emulate the processed path yourself. There is no way to get the final computed path after shape modifiers like Offset, Zig Zag, or Wiggle have been applied.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `shape-layer`, `path-vertices`, `shape-modifiers`, `aegp`, `limitation`*

---

## Is there an API to get the layer comment field from C++ code?

There is no C API for getting layer comments. Use AEGP_ExecuteScript() to run ExtendScript that reads the layer comment and retrieve the result back to the C side.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2015-01-01 · Tags: `layer-comment`, `aegp`, `execute-script`, `extendscript`*

---

## After signing, notarizing, and stapling a Mac plugin, why does it become untrusted after uploading and re-downloading?

This is a known issue where the notarization appears successful (notarytool says Accepted) but spctl reports 'rejected / source=Unnotarized Developer ID' after download. The downloaded file may get an Apple quarantine xattr. This can happen with AEGP plugins that are .plugin bundles. The issue may be related to how the bundle is zipped/unzipped, or how the stapling interacts with the bundle structure. Check the xattr flags after download with 'xattr -l <path>' to see if quarantine is applied.

```cpp
codesign --options runtime --force --deep --timestamp -strict --sign "Developer ID Application: Name (TEAMID)" <path/to/.plugin/file>
xcrun notarytool submit <path/to/zipped/.plugin/file> --keychain-profile "Profile" --wait
xcrun stapler staple <path/to/.plugin/file>
```

*Source: aescripts discord · 2024-09-12 · Tags: `macos`, `notarization`, `code-signing`, `quarantine`, `spctl`, `aegp`*

---

## What happens if you try to dispose of a world that failed to allocate due to insufficient memory?

If allocation fails, do not attempt to dispose of the world. Attempting to dispose of a world you didn't successfully allocate will trigger an error warning stating you should not dispose of a world you didn't allocate. This error appears on top of the original out of memory warning, creating a confusing user experience.

*Tags: `memory`, `aegp`, `debugging`*

---

## How do you hide a banner in the timeline while showing it in the Effect Control Window?

The banner needs to be associated with a keyframeable parameter to show in both places like the Color Grid plugin does. For a static banner without keyframing, you need to configure it to only display in the ECW and not in the timeline.

*Tags: `ui`, `params`, `aegp`*

---

## Is it illegal to checkout a parameter multiple times?

You cannot checkout parameters during sequence setup, resetup (when AE is loading the project), or sequence setdown. The world checkout can sometimes be empty in these contexts. Additionally, you must handle checkout errors and check in the parameter immediately after using it in loops.

*Tags: `params`, `layer-checkout`, `aegp`, `debugging`*

---

## When should parameter check-in be performed relative to parameter usage?

The check-in must be done immediately after getting the parameter value, especially in loop examples where the parameter is accessed multiple times.

*Tags: `params`, `layer-checkout`, `aegp`*

---

## Should error handling be done when checking out layers, and should check-in occur immediately after use?

Yes, error handling should be implemented for checkout operations. Check-in should be done immediately after getting the value. In a loop structure, you should checkout a frame, check for errors, perform the job if no error occurred, and then check-in the frame before moving to the next iteration. Each checked-out resource must be checked in, whether it was used or not.

```cpp
For X
     Checkout frame x
         If (no err) dojob
     Err2(check-in frame x)
End of loop
```

*Tags: `layer-checkout`, `aegp`, `error-handling`, `render-loop`, `threading`*

---

## Is it possible to set or change the value of a LAYER_PARAM in the plugin UI?

Yes, you can set the layer ID value in a streamVal2 using SetStreamValue.

*Tags: `params`, `ui`, `aegp`*

---

## Can you write to sequence data during PF_update_param_ui with MFR?

No, you cannot write to sequence data during PF_update_param_ui with MFR. You can only do it in the user changed param callback. If you need to modify layer defaults, you may need to use AEGP instead.

*Tags: `mfr`, `sequence-data`, `params`, `aegp`*

---

## How can you set a layer default parameter value?

Layer defaults do not have a direct 'value' property. You can try using PF_LayerDefault_NONE or use the AEGP suite to modify layer defaults.

```cpp
param.u.ld = PF_LayerDefault_NONE
```

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How do you extract scale values from a decomposed transformation matrix?

Use AEGP_MatrixDecompose4() on your result matrix to decompose it into position (posVP), scale (scaleVP), shear (shearVP), and rotation (rotVP) components. The scale values will be in the scaleVP output parameter.

```cpp
A_FloatPoint3 posVP, scaleVP, shearVP, rotVP;
A_Matrix4 resultMat4;
ERR(suites.MathSuite1()->AEGP_MatrixDecompose4(&resultMat4, &posVP, &scaleVP, &shearVP, &rotVP));
```

*Tags: `mfr`, `params`, `aegp`*

---

## How do you toggle a UI element's visibility on and off in an After Effects plugin?

To toggle UI element visibility, use the AEGP ParamUI thread approach instead of directly modifying ui_flags. Get the effect and stream references using AEGP_GetNewEffectForEffect and AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag with the AEGP_DynStreamFlag_HIDDEN flag. The issue with the direct ui_flags approach is that the bitwise NOT operator (!PF_PUI_INVISIBLE) doesn't properly clear the flag—you need to use the AEGP suite methods instead. Note that Premiere has a different method for accomplishing this.

```cpp
static PF_Err
changeParamVisibility(PF_InData            *in_data,
                      PF_OutData            *out_data,
                      PF_ParamDef          *paramsDef,
                      PF_ParamIndex        paramIndex,
                      PF_Boolean           paramVisibleB)
{
    PF_Err                err                    = PF_Err_NONE,
    err2                = PF_Err_NONE;
    global_dataP        globP                = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler        suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr')
    {
        AEGP_EffectRefH            meH                = NULL;
        AEGP_StreamRefH      currStreamH        = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex,     &currStreamH));
        if (meH && currStreamH)
        {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
        if (meH){
            ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
        }
        if ( currStreamH ){
            ERR2(suites.StreamSuite2()->AEGP_DisposeStream( currStreamH ));
        }
    }
    return err;
}
```

*Tags: `ui`, `params`, `aegp`, `premiere`*

---

## Where can I find code to get all active camera properties?

The Resizer example in the SDK contains code for accessing camera properties and interfacing with camera layers.

*Tags: `aegp`, `camera`, `sdk`, `layer-checkout`*

---

## How do you convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL matrix format?

After Effects uses row-major matrix layout while OpenGL uses column-major layout. You need to transpose the matrix during conversion. For a 4x4 matrix, map AE matrix elements (row-major: 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15) to OpenGL format (column-major: 0,4,8,12,1,5,9,13,2,6,10,14,3,7,11,15). This can be done by iterating through the matrix and reassigning: glMatrix[x][0] = matrix.mat[0][x], glMatrix[x][1] = matrix.mat[1][x], glMatrix[x][2] = matrix.mat[2][x], glMatrix[x][3] = matrix.mat[3][x].

```cpp
for (x = 0; x < 4; x++)
{
    glMatrix[x][0] = matrix.mat[0][x];
    glMatrix[x][1] = matrix.mat[1][x];
    glMatrix[x][2] = matrix.mat[2][x];
    glMatrix[x][3] = matrix.mat[3][x];
}
```

*Tags: `aegp`, `opengl`, `matrix-conversion`, `camera`*

---

## What tools should be used for matrix operations and inversion when working with camera matrices in After Effects plugins?

The GLM (OpenGL Mathematics) library is recommended for performing matrix operations and inversions. It provides robust mathematical functions for handling matrix transformations needed when converting and manipulating camera coordinate systems.

*Tags: `aegp`, `opengl`, `matrix-conversion`, `camera`, `build`*

---

## How do you get a Layer Handle from a layer parameter (PF_ADD_LAYER) to access its properties and stream?

A layer parameter only returns a World (pixel data) and not the actual layer definition or handle. It is not possible to retrieve the Layer Handle from a PF_ADD_LAYER parameter to access the layer's properties and stream. You will need to use an alternative approach.

*Tags: `params`, `layer-checkout`, `aegp`*

---

## How do you get a layer parameter value and retrieve the actual layer object from it in AEGP?

Use AEGP_GetNewEffectForEffect to get the effect handle, then AEGP_GetNewEffectStreamByIndex to get the stream for the parameter. Call AEGP_GetNewStreamValue to get the layer ID value, then use AEGP_GetLayerFromLayerID with the parent composition handle to get the target layer handle. Remember to dispose of the effect handle and stream handles when done.

```cpp
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, param_index, &currStreamH));
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &currLayerH));
ERR(suites.LayerSuite5()->AEGP_GetLayerParentComp(currLayerH, &compPH));
if (meH && currStreamH && currLayerH && compPH)
{
    ERR(suites.StreamSuite3()->AEGP_GetNewStreamValue(globP->my_id,currStreamH,AEGP_LTimeMode_LayerTime,&currTime,TRUE, &val));
    if (!err)
    {
        AEGP_LayerIDVal layerId = val.val.layer_id;
        ERR(suites.LayerSuite8()->AEGP_GetLayerFromLayerID(compPH, layerId, &targetLayerH));
    }
}
```

*Tags: `aegp`, `params`, `layer-checkout`*

---

## What is the difference between checking out a layer parameter with PF versus using AEGP streams?

When you checkout a layer parameter using the PF interface, you get a PF_Effect_world. When you use AEGP streams, you get the layer index instead.

*Tags: `aegp`, `params`, `layer-checkout`*

---

## How can you get a drawing reference from within the render loop without relying on user interaction events?

The drawing reference is typically obtained through PF_GetDrawingReference in the PF_Cmd_EVENT handler via the event_extraP context, which is only available when the user interacts with the effect. Getting a drawing reference from within the render loop without user interaction is not straightforward and requires alternative approaches, possibly involving undocumented APIs or workarounds that experienced developers like Shachar Raindel might know about.

```cpp
suites.EffectCustomUISuite1()->PF_GetDrawingReference(event_extraP->contextH, &drawing_ref);
```

*Tags: `ui`, `render-loop`, `aegp`, `debugging`*

---

## Do you need to use AEGP for adding time values in the compute cache API?

The conversation indicates this is a question about the proper API approach for time value handling in the compute cache, but a definitive answer was not provided in the chat.

*Tags: `compute-cache`, `aegp`, `smartfx`*

---

## Can C++ PersistentDataSuite3 and ExtendScript app.preferences share the same settings?

The question was asked but not answered in the conversation. The user clarified they meant app.preferences rather than app.settings, but no definitive answer was provided about whether these two APIs can share the same underlying settings storage.

*Tags: `scripting`, `aegp`, `params`*

---

## What is the difference between in_data->sequence_data and out_data->sequence_data?

in_data->sequence_data is where you read sequence data. out_data->sequence_data is where you write to sequence data. In CC2022 and later, there are special functions for reading/writing. You can only write to sequence data in specific contexts: sequence setup, resetup, user changed param, do dialog, or external_dependencies callbacks. Use a static variable instead if you need to modify data more freely.

*Tags: `sequence-data`, `params`, `aegp`*

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

## Should script text be stored in an arbitrary parameter when implementing a button that triggers script execution?

Yes, the script text should be stored in an arbitrary parameter. A button parameter triggers the AEGP_ExecuteScript() function to execute the script string stored in the arb data.

*Tags: `arb-data`, `params`, `ui`, `aegp`*

---

## How can a plugin render frame N that depends on custom data from previous frames without checking out all 100 previous frames in memory during pre-render?

This is a known challenge with recursive operations in After Effects plugins. The issue is that smart render checks needed frames during pre-render, which forces checking out all intermediate frames. The checkout param solution doesn't account for previous effects. AEGP_CacheAndCheckoutFrame doesn't trigger the pre_render/smart render thread, so it doesn't solve the problem. A potential approach is to use layer checkout parameters during compute cache operations, but this requires careful design to avoid excessive memory usage and to ensure previous effects are properly evaluated.

*Tags: `smart-render`, `compute-cache`, `layer-checkout`, `memory`, `aegp`*

---

## Is PF_Arbitrary_NEW_FUNC ever called in post-CS6 AE architecture?

No, PF_Arbitrary_NEW_FUNC is not called in post-CS6 architecture. Instead, you allocate a default Arb handle during ParamSetup, and for every new instance AE sends a PF_Arbitrary_COPY_FUNC selector with your default arb handle. The arb data won't be zero length.

*Tags: `arb-data`, `params`, `aegp`*

---

## What is the correct way to initialize arbitrary data in ParamSetup?

In ParamSetup, you fill the default arb data by calling a custom function to create the default arb. You allocate the default Arb handle during ParamSetup, and AE will send PF_Arbitrary_COPY_FUNC selectors with your default arb handle for each new instance.

*Tags: `arb-data`, `params`, `aegp`*

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

## Why is there a memory leak when using AEGP_memorysuite on Mac but not Windows during export?

The memory leak appears to be platform-specific (Mac only) and related to how AEGP_memorysuite handles memory operations. The issue is less severe with the mem_QUIET flag. Using standard C++ memory allocation (new/delete) instead of AEGP_memorysuite resolves the leak, and replacing memcpy with strncpy partially solves the problem (from mem pointer to buffer works, but not in the reverse direction). This suggests the issue may be related to how memcpy interacts with AEGP's memory management on macOS, particularly when memory is freed from a different thread than where it was allocated.

*Tags: `memory`, `macos`, `aegp`, `threading`, `debugging`*

---

## Is there a limitation with AEGP_memsuite when allocating and freeing memory during the render thread?

The user observed that memory allocated using aegp_memHandle during the compute cache thread is not being freed when the freemem function is called, even though instruments show no leaks. This suggests there may be specific limitations or requirements for using AEGP_memsuite during the render thread that differ from other threads.

*Tags: `memory`, `aegp`, `render-loop`, `compute-cache`, `threading`, `debugging`*

---

## Why does checkout_layer_pixels return NULL data for a layer parameter after checkout_layer succeeds?

This appears to be a layer checkout issue where the layer parameter has valid layer data after PreRender checkout_layer calls, but checkout_layer_pixels in SmartRender returns a structure with NULL data. The issue may be related to how the layer parameter is defined or how Adobe's GPU template handles layer checkouts. Ensure the layer parameter is properly defined with PF_ADD_LAYER and that the same layer index is used consistently between PreRender and SmartRender callbacks.

```cpp
// ParamsSetup
AEFX_CLR_STRUCT(def);
PF_ADD_LAYER(LAYER_STR, PF_LayerDefault_NONE, GPU_SKELETON_LAYER_Disk_ID);

// SmartRender
ERR(extraP->cb->checkout_layer_pixels(in_data->effect_ref, GPU_SKELETON_LAYER, &env_worldP));
if (env_worldP != NULL && env_worldP->data == NULL) {
    FX_LOG("FATAL!!!! :: Environment layer data is NULL");
}
```

*Tags: `layer-checkout`, `params`, `smart-render`, `gpu`, `aegp`*

---

## Why does changing the match name cause the CUI and parameters to become invisible in the ECW?

This was caused by an underlying C++ syntax error where a PF_Err variable was declared but not initialized, causing AEGP_GetNewEffectStreamByIndex to return 0x0. The real issue was that error checking with if(!err) statements was failing because the uninitialized error variable had a garbage value (usually 1) instead of PF_Err_NONE (0). The solution is to properly initialize all error variables when declaring multiple variables of the same type.

```cpp
// Wrong - err is undefined
PF_Err err, err2 = PF_Err_NONE;

// Correct - both variables initialized
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `pipl`, `aegp`, `debugging`, `params`*

---

## Why doesn't macOS Instruments detect memory leaks from PF_EffectWorlds that are allocated but never disposed?

PF_EffectWorlds are likely allocated deep within the AE engine rather than directly within the plugin code, so memory leak detection tools like Instruments don't recognize them as true leaks since they're allocated at a different level in the AE memory management hierarchy.

*Tags: `memory`, `macos`, `debugging`, `aegp`*

---

## How can you improve memory management for AESDK objects in After Effects plugins?

Create C++ wrappers around AESDK objects to enable automatic memory management. There are already some C++ wrappers of portions of the SDK available on GitHub that can be used for this purpose.

*Tags: `memory`, `build`, `aegp`*

---

## How can I get text layer path vertices in 3D world space when the text layer is parented to another layer?

Use AEGP_GetLayerToWorldXform() to convert from layer space to world space. However, note that PF_PathDataSuite does not return vertices in true layer space as documented. The vertices it returns are affected by 2D transforms (xy-translation, shear from parented transforms, Z Rotation) but not by z-translation, Orientation, or X/Y Rotation properties. To match the behavior of path.points() in expressions (which returns true layer space coordinates), you need to account for this discrepancy by converting PF_PathDataSuite output to match the layer space definition used in expressions before applying the layer-to-world transform.

*Tags: `aegp`, `path-data`, `3d-transforms`, `layer-space`, `text-layer`*

---

## Why might an AEGP not appear in AE and not hit any breakpoints in the entire project?

The questioner did not receive a definitive answer. The conversation moved to other topics without resolution.

*Tags: `aegp`, `debugging`, `macos`*

---

## Can you use AEGP render suite during the render call, or only during PreRender?

You cannot use AEGP render suite during the render call itself. However, you can use it during PreRender to get rendered frames at a given time with the previous effect reference. Alternatively, you can use smart render with frame preloading in pre-render, using the same method as the current frame.

*Tags: `aegp`, `smart-render`, `render-loop`, `smartfx`*

---

## Does Media Encoder load AEGP plugins to render After Effects projects?

No, AEGP plugins are not supported by Media Encoder (AME). If your plugin needs an AEGP plugin to render, you will need to get your AEGP as a standalone/command line tool to work around this limitation.

*Tags: `aegp`, `deployment`, `premiere`*

---

## How should the Compute Cache API be implemented with callback functions?

Use the AEGP_ComputeCacheSuite1 to register a compute class with four callback functions: MyGenerateKeyFunc (to identify the effect ref, layer id, effect position, current time, and time scale id), MyComputeFunc (to perform the computation), MyApproxSizeValueFunc (to return the approximate size of the cached value), and MyDeleteComputeValueFunc (to free the cached value). Register these callbacks using AEGP_ClassRegister with a unique class identifier string.

```cpp
#include "AE_ComputeCacheSuite.h"
A_Err MyGenerateKeyFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeKeyP out_keyP) {
    LOG("Generate Key function");
    return A_Err_NONE;
}
A_Err MyComputeFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeValueRefconP *out_valuePP) {
    LOG("Compute Functions");
    return A_Err_NONE;
}
size_t MyApproxSizeValueFunc(AEGP_CCComputeValueRefconP valueP) {
    LOG("Obtain size function");
    return 0;
}
void MyDeleteComputeValueFunc(AEGP_CCComputeValueRefconP valueP) {
    LOG("Delete function");
}
static PF_Err GlobalSetup(PF_InData* in_data, PF_OutData* out_data, PF_ParamDef* params[], PF_LayerDef* output) {
    AEFX_SuiteScoper<AEGP_ComputeCacheSuite1> compute_suite = AEFX_SuiteScoper<AEGP_ComputeCacheSuite1>(in_data, kAEGPComputeCacheSuite, kAEGPComputeCacheSuiteVersion1);
    AEGP_ComputeCacheCallbacks callbacks = { MyGenerateKeyFunc, MyComputeFunc, MyApproxSizeValueFunc, MyDeleteComputeValueFunc };
    compute_suite->AEGP_ClassRegister("com.mycompany.effect.myComputeCacheClass", &callbacks);
}
```

*Tags: `compute-cache`, `aegp`, `memory`, `caching`*

---

## How can you implement a key generation function that requires AEGP_HashSuite1 when the function signature only provides AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP parameters?

This question was asked but not answered in the conversation.

*Tags: `aegp`, `compute-cache`, `debugging`*

---

## How do you properly access compute cache data and retrieve the basic suite pointer?

Create an access_cache_data pointer from optionsP, verify in_data is not null, then extract the SPBasicSuite pointer from in_data->pica_basicP. Always check that bsuite is valid before using it to avoid allocation errors.

```cpp
access_cache_data* accessCacheData = static_cast<access_cache_data*>(optionsP);
if ( !accessCacheData->in_data ) {return A_Err_ALLOC;}
SPBasicSuite* bsuite;
bsuite = accessCacheData->in_data->pica_basicP;
if (!bsuite){return A_Err_ALLOC; }
```

*Tags: `compute-cache`, `aegp`, `memory`*

---

## What structure should be used to pass in_data and out_data to compute cache functions?

Define a struct access_cache_data that contains PF_InData* and PF_OutData* pointers to pass the required suite information to the cache. Create a new object and copy only the needed parts, then delete it after computing to avoid crashes and maintain independence from the render thread.

```cpp
struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
}
```

*Tags: `compute-cache`, `aegp`, `threading`, `memory`*

---

## How can AEGPSetLayerTransferMode be used to set a layer as a trackmatte in After Effects versions before 23?

AEGPSetLayerTransferMode has limitations in older versions - it only works if the layer is already a trackmatte, and cannot set a layer to become a trackmatte if it isn't already one. The user reported this issue when trying to set a layer's transfer mode to ALPHA trackmatte using LayerSuite7. After Effects 23 and newer have a new method available that provides better backwards compatibility, though the specific method name was not detailed in the conversation.

```cpp
AEGP_LayerTransferMode mode = {};
mode.mode = PF_Xfer_IN_FRONT;
mode.track_matte = AEGP_TrackMatte_ALPHA;
mode.flags = 0;
ERR(suites.LayerSuite7()->AEGP_SetLayerTransferMode(layerH, &mode));
```

*Tags: `aegp`, `layer-checkout`*

---

## Are there certain suites that shouldn't be called from Death Hook?

Yes, there are restrictions. The Utility Suite should not be called from Death Hook as it can cause unhandled exceptions when trying to use functions like ReportInfo.

*Tags: `aegp`, `debugging`, `threading`*

---

## Are there certain AEGP suites that should not be called from Death Hook, and does using Utility Suite to ReportInfo cause exceptions?

The user reported unhandled exceptions when attempting to call Utility Suite's ReportInfo function from Death Hook. This suggests that certain suites have restrictions on being called from Death Hook contexts, likely due to the cleanup phase not supporting all operations. The answer was not explicitly provided in the conversation, but the question indicates a known limitation.

*Tags: `aegp`, `death-hook`, `debugging`*

---

## Is it possible for one effect to render from another effect during smartrender?

Direct effect-to-effect rendering during smartrender is not possible because After Effects will not call smartrender on other layers while a current smartrender is in progress—it waits until the current smartrender is done. Workarounds include: (1) using AEGP outside of render calls but this has poor UI; (2) calling EffectCallGeneric on another plugin if that plugin implements a passthrough code for SmartRender; (3) using AEGP_RenderAndCheckoutLayerFrame_Async which works on any thread but can be buggy; (4) delegating AEGP calls to the UI thread for the next call, though smartrender happens on the render thread and cannot directly call UI thread methods.

*Tags: `smartrender`, `render-loop`, `aegp`, `threading`*

---

## What is AEGP_RenderAndCheckoutLayerFrame_Async and can it be called from smart_render command?

AEGP_RenderAndCheckoutLayerFrame_Async is an asynchronous layer rendering and checkout function that can work on any thread, making it potentially usable from smartrender command, though it is known to be buggy.

*Tags: `smartrender`, `aegp`, `layer-checkout`, `threading`*

---

## What error occurs when trying to adjust effect parameters using AEGP during smartrender?

After Effects gives an error box saying 'error: effect attempting to modify a locked project' when trying to adjust the parameter of an effect using AEGP during a smartrender command.

*Tags: `smartrender`, `aegp`, `params`, `debugging`*

---

## Is it possible to include multiple PiPLs in the same file?

Yes, it is possible to include multiple plug-ins (both AEGPs and effects) in the same file using multiple PiPLs, but it is not recommended. If there are PiPLs for both AEGPs and effects in the same file, the AEGPs must come first. No other hosts, not even Premiere Pro, support multiple PiPLs pointing to multiple effects within the same .dll or code fragment. The recommendation is to use one PiPL and one plug-in per code fragment.

*Tags: `pipl`, `aegp`, `premiere`, `build`*

---

## When is PF_Cmd_GLOBAL_SETUP called in After Effects and Premiere?

In After Effects, GLOBAL_SETUP is called the first time the user applies the effect plugin to a layer. In Premiere, it is called when the application is loading.

*Tags: `aegp`, `pipl`, `premiere`, `ui`*

---

## Is it possible to show a dialog during Global Setup?

It should be possible to show any dialog during Global Setup, though there is some uncertainty about whether dialogs can be displayed during the initial AE loading screen when plugins are being scanned.

*Tags: `ui`, `aegp`, `pipl`*

---

## How do you check out a frame of the current layer after prior effects are applied, rather than before them?

PF_CHECKOUT_PARAM gets the current frame before all prior effects. To get the frame after prior effects (like after Exposure but before the current effect), you need to use a different checkout method. The conversation indicates this is possible but the specific method was not detailed in this exchange.

*Tags: `layer-checkout`, `aegp`, `params`*

---

## What causes slowness when displaying a timeline with many layers in After Effects?

The slowness is primarily in the UI rendering rather than the plugin rendering itself. The bottleneck is likely in After Effects internal calls such as 'get layer info' and 'get layer sprite'. Even with many layers hidden by default, performance issues persist. After Effects also has poor memory management, making it inefficient with large numbers of layers (10k+).

*Tags: `ui`, `memory`, `performance`, `aegp`*

---

## Would using an Artisan API approach help speed up rendering with many layers?

An Artisan API approach likely wouldn't meaningfully speed up the performance, since the bottleneck is in After Effects' internal layer information retrieval calls rather than the rendering process itself.

*Tags: `artisan`, `aegp`, `performance`*

---

## What is the purpose of the changedB parameter in PathOutline?

The changedB parameter appears to be a flag that indicates whether path data has been modified, though the exact public API for modifying path data is not exposed in the SDK. Setting it to TRUE may signal to After Effects that path data needs to be updated in the UI, but the internal APIs used by After Effects to modify path data are not publicly available.

*Tags: `params`, `ui`, `aegp`*

---

## Can mask outline modifications be done as a regular effect that procedurally modifies path vertices each frame rather than as a one-time destructive operation?

You cannot modify any project state in the render thread (like adding layers, changing layer properties, or masks). However, procedurally changing mask vertices might be possible using expressions. Baking the paths is also an effective alternative approach, though not as magical.

*Tags: `aegp`, `scripting`, `render-loop`, `params`*

---

## If mask path APIs are available in the JavaScript API, why couldn't they be used in C++ plug-ins by running the script within the plug-in?

The conversation suggests this is theoretically possible - you can run scripts within a plug-in to access JavaScript API functionality. However, this approach has limitations compared to native C++ implementation, particularly regarding per-character path separation which is still being lobbied for in the C++ plugin API.

*Tags: `scripting`, `aegp`, `params`, `debugging`*

---

## Can mask outlines be modified using AEGP_MaskOutlineSuite?

Yes, AEGP_MaskOutlineSuite can be used to modify mask outlines, but this approach would be destructive and one-time only, similar to running a script that modifies path vertices. For a procedural effect that modifies path vertices each frame before rasterization, a regular effect applied to a layer would be more appropriate.

*Tags: `aegp`, `params`, `render-loop`*

---

## What does 'connected instance to instance' mean in the context of multiple plugin instances?

It refers to having multiple instances of the plugin on the same or different layers communicating with each other.

*Tags: `aegp`, `params`, `threading`*

---

## What is the correct way to call AEGP_StartUndoGroup in After Effects 2025?

In After Effects 2025 beta and later, AEGP_StartUndoGroup must be called with an empty string "" rather than null. Passing null will cause a crash, while passing an empty string works as expected and does not add an entry to the Undo stack.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `undo`, `macos`, `windows`, `debugging`*

---

## How can a plugin with a new mode parameter maintain backward compatibility with older projects that lack this parameter?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on the parameter. This flag ensures that when opening projects created with older versions of the plugin that don't have the new parameter, After Effects will use the default value specified for that parameter rather than causing visual inconsistencies.

*Tags: `params`, `backward-compatibility`, `aegp`*

---

## How does After Effects store arbitrary data of a plugin to disk when that plugin is not installed?

Arbitrary data is serialized via PF_Cmd_Arbitrary_Callback using the flatten function, similar to sequence data. However, the flatten/unflatten versions are not accessible via AEGP functions. For plugins like 'curves' with stack variable handles, the data can be reinterpreted cast as char* and serialized. However, for plugins like 'liquify' that have pointers in their struct, serialization is problematic because deserialized pointer addresses are invalid on reload and will cause crashes.

*Tags: `arb-data`, `aegp`, `sequence-data`, `params`*

---

## Is there a way to serialize all plugin data including arbitrary data for template/capsule creation?

For first-party plugins where you know the handle, you can serialize the data yourself. However, for third-party and Adobe stock plugins, it is very tricky. The main challenge is accessing the flatten/unflatten versions of arbitrary data which are not exposed via AEGP functions. Some plugins have serializable stack variable handles (like 'curves') while others contain pointers (like 'liquify') that cannot be reliably serialized and deserialized. Extended script investigation may be a possible avenue to explore.

*Tags: `arb-data`, `aegp`, `scripting`, `params`*

---

## Is it possible to determine which stream was selected in the effect control window using AEGP API?

Based on the AESDK, there does not seem to be a way to know which stream was selected in the effect control window, unlike the timeline where you can use the selection collection to determine which streams are selected. Additionally, you cannot register a command inside the right-click context menu of the effect control window; the best alternative is to place the command in the Keyframe Assist menu.

*Tags: `aegp`, `ui`, `params`*

---

## Why is the PF_ParamDef* params[] pointer always nullptr when handling PF_Cmd_EVENT?

When handling PF_Cmd_EVENT, you need to perform a switch on extra->e_type. Parameter modification is only possible for specific event types, particularly click and keyboard events. The params pointer may not be available in all event types, and you may need to use AEGP functions instead of directly accessing the params pointer depending on the event type.

```cpp
case PF_Cmd_EVENT:
    err = HandleEvent(in_data, out_data, params);
    // Switch on extra->e_type
    // Check if event is click or keyboard type
```

*Tags: `params`, `aegp`, `ui`, `debugging`*

---

## How can you access the Adobe application version from an After Effects plugin?

You can use ExtendScript with 'app.version' via the aegp execute script. Alternatively, if your plugin supports both After Effects and Premiere Pro with GPU features, you can access the version through the Premiere Pro PICA Suite AppInfo, which is called before global setup. The version data is typically formatted like '24.3', and you need to add 2000 to convert it. For versions with additional components like '24.3.x', some parsing is required.

*Tags: `aegp`, `scripting`, `premiere`, `cross-platform`*

---

## How can you determine the After Effects version at runtime?

You can use a 'dirty solution' by locating the path of the running binary host using AEGP_GetPluginPaths with AEGP_GetPathTypes_APP as the path type. From this you can extract the AE folder name or version from the binary's metadata. Alternatively, you can try mapping AEGP_GetDriverSpecVersion to the AE version number.

```cpp
AEGP_GetPluginPaths(plugin_id, AEGP_GetPathTypes_APP, &path_outH)
```

*Tags: `aegp`, `deployment`, `debugging`*

---

## Does the AEGP version increment in parallel with the host After Effects version?

This was posed as a question about whether AEGP version numbering follows a parallel system to the host version, but no definitive answer was provided in the conversation.

*Tags: `aegp`, `deployment`*

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

## How should sequence_data be accessed in SmartRender when it appears as nullptr?

Since After Effects v22, you must use the PF_EffectSequenceDataSuite to read sequence data in SmartRender. Create an AEFX_SuiteScoper for PF_EffectSequenceDataSuite1, call PF_GetConstSequenceData with the effect_ref to get a const handle, then lock that handle to access the sequence data. This provides read-only access to the sequence data.

```cpp
PF_ConstHandle const_seq;
AEFX_SuiteScoper<PF_EffectSequenceDataSuite1> seqdata_suite =
    AEFX_SuiteScoper<PF_EffectSequenceDataSuite1>(
        in_data,
        kPFEffectSequenceDataSuite,
        kPFEffectSequenceDataSuiteVersion1,
        out_data);
seqdata_suite->PF_GetConstSequenceData(in_data->effect_ref, &const_seq);
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `sequence-data`, `smart-render`, `aegp`, `memory`*

---

## Do you need to lock the handle when calling GetConstSequenceData for sequence data?

Yes, you need to lock the handle even when reading only. Use PF_LOCK_HANDLE(const_seq) to lock the handle before accessing it, and unlock it after use. This is necessary in multi-frame render (MFR) scenarios with multiple plugin applications, and the lack of locking can cause nullptr returns and crashes. While Adobe documentation doesn't clearly document this requirement, it appears to be essential for thread safety.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `mfr`, `sequence-data`, `threading`, `memory`, `aegp`, `debugging`*

---

## How should dynamic dropdown lists be updated in After Effects plugins as of CC2025.2 to avoid crashes and empty display?

Use the AEGP StreamSuite to update dropdown parameters via AEGP_SetStreamValue instead of directly modifying the param union during param_ui thread. Call AEGP_GetNewEffectStreamByIndex to get the stream reference, create an AEGP_StreamValue with the new value, use AEGP_SetStreamValue to apply it, and dispose the stream with AEGP_DisposeStream. This approach is more reliable than strncpy_s manipulation of namesptr, though it may have undo history implications that should be tested.

```cpp
PF_Err setFloatOrBoolOrDropdownParamViaAEGP(uint16_t i_param_to_change, float val,
                                     AEGP_PluginID my_id, AEGP_EffectRefH *effect_refHP, AEGP_SuiteHandler *suitesP, GlobalData* global_data)
{
    PF_Err err = PF_Err_NONE;
    AEGP_StreamValue streamvalue;
    streamvalue.val.one_d = val;
    AEGP_StreamRefH param_refH = NULL;
    ERR(suitesP->StreamSuite2()->AEGP_GetNewEffectStreamByIndex(my_id, *effect_refHP, i_param_to_change, &param_refH));
    streamvalue.streamH = param_refH;
    ERR(suitesP->StreamSuite2()->AEGP_SetStreamValue(my_id, param_refH, &streamvalue));
    ERR(suitesP->StreamSuite5()->AEGP_DisposeStream(param_refH));
    return err;
}
```

*Tags: `popup`, `aegp`, `params`, `ui`, `windows`*

---

## Does using AEGP_SetStreamValue to update dropdown parameters break the undo history in After Effects?

This is uncertain. The implementation includes a debug warning stating that using AEGP_SetStreamValue causes weird bugs in the undo history, but the actual extent and nature of these bugs is not definitively documented. Users should test this functionality in their specific implementation to understand the undo behavior.

*Tags: `aegp`, `params`, `debugging`*

---

## Do PF_LOCK_HANDLE and host_lock_handle functions actually do anything in After Effects?

According to Adobe engineers, PF_LOCK_HANDLE and host_lock_handle functions are dummy functions that do nothing at all. As stated by Jason Bartell from Adobe in a 2022 discussion: 'Executive summary is handle locking and unlocking has not done anything in After Effects since roughly CS6.'

*Tags: `aegp`, `memory`, `debugging`*

---

## Does PF_Interrupt return a value that triggers the catch block when it's not PF_Err_NONE?

No, PF_Interrupt returns an error code, not an exception. A non-PF_Err_NONE error return does not trigger a catch block in C++.

*Tags: `debugging`, `aegp`*

---

## What are the new entry points in After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. Currently, PIPL is still used by After Effects but not by Premiere Pro anymore, so you only need to write PIPL for AE. The new entry points potentially allow for removing PIPL/rsrc resources entirely, though the exact mechanism needs further investigation in the samples.

*Tags: `pipl`, `aegp`, `ae`, `plugin-architecture`*

---

## How do you specify which function is EffectMain when using the new PF_REGISTER_EFFECT entry points?

The exact mechanism for specifying the EffectMain function with the new entry points is not clearly documented. It should be checked in the official After Effects plugin samples, though the investigation into this approach is ongoing.

*Tags: `pipl`, `aegp`, `plugin-architecture`*

---

## What causes the generic exception handler in Adobe suites manager to appear, and why is it more frequent in newer AE versions?

The generic exception handler/catch-all in the Adobe suites manager appears when double freeing or releasing a suite pointer. It has existed since CS2 or CS3 days. It appears more often in newer AE versions likely due to MFR (Multi-Frame Rendering) with global suite handles or suite pointers being shared between threads.

*Tags: `mfr`, `threading`, `memory`, `aegp`*

---

## How should error handling be structured when developing After Effects plugins?

Use std::expected for function returns instead of traditional error codes, allowing for proper .then chaining. Wrap all AE handles in classes with proper construction and destruction via smart pointers. Implement an allocation class manager that clears allocations when exiting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug mode, use a cache validator to detect extra memory still allocated at endpoints.

*Tags: `aegp`, `memory`, `debugging`, `error-handling`*

---

## How can you check if the After Effects renderer is running in headless mode?

Use the AppSuite4 API with the PF_IsRenderEngine function. Create a PF_Boolean variable and call suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine) to set it to true if running in render engine mode.

```cpp
PF_Boolean bIsRenderEngine = true;
suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine);
```

*Tags: `aegp`, `render-loop`, `debugging`, `deployment`*

---

## How can you set text on a text layer from an effect on a frame-by-frame basis?

You need to communicate with an AEGP plugin that can set the text of a layer for each frame, since invoking scripts via the render loop does not work for this purpose.

*Tags: `aegp`, `scripting`, `render-loop`, `text`*

---

## How can a C++ plugin update a text layer every frame after rendering completes?

This is not cleanly possible due to AE's project locking during render operations. Possible workarounds include: (1) writing data to a compute cache or sequence data and reading it from the UI thread via the aegp_idle hook, (2) using AEGP_SetText from the UI thread instead of the render thread, (3) rendering text yourself in the render thread using an external library, or (4) saving to disk and having a ScriptUI panel with a timer read the data. The aegp_idle hook approach may not work without a GUI or in MediaEncoder. Setting text from the render thread causes a locked project error.

*Tags: `aegp`, `render-loop`, `threading`, `ui`, `sequence-data`, `compute-cache`*

---

## Would the aegp_idle hook approach work when rendering through MediaEncoder or without a GUI?

No, the aegp_idle hook approach would not work in MediaEncoder or without a GUI, since the idle hook requires an active UI thread to execute.

*Tags: `aegp`, `threading`, `ui`, `deployment`*

---

## What is the best way to pass data from an AEGP plugin to an effects plugin?

Use an AEGP plugin with an idle hook to process a queue. Every frame, push your string result into a queue, and the idle hook processes the queue occasionally. The AEGP plugin and FX plugin are separate binaries, so you'll need a C interface to push strings into the queue.

*Tags: `aegp`, `threading`, `render-loop`, `memory`*

---

## How can you communicate between an AEGP plugin and an FX plugin to push strings into a queue?

You will need a C interface to push the string into the queue between the two binaries. The AEGP will be one plugin and the FX plugin will be another separate binary.

*Tags: `aegp`, `pipl`, `cross-platform`*

---

## Is it possible to create a multi-layer 2D global illumination plugin where a renderer effect discovers and uses material properties from dummy effects on other layers?

This approach is theoretically possible with the After Effects SDK. You can use the AEGP suite to enumerate layers in a composition and checkout their properties. However, the key challenge is properly communicating dependencies to AE's internal dependency tracking system. You need to explicitly declare all layer and effect dependencies your renderer effect relies on, so that changes to materials or other layers trigger re-renders. This ensures the disk cache is used correctly—cached results are only used when all dependencies remain unchanged. The AEGP layer enumeration and property checkout APIs support this pattern, though it requires careful dependency management to avoid either missing updates or unnecessary re-renders.

*Tags: `aegp`, `layer-checkout`, `caching`, `params`, `compute-cache`*

---

## How can a renderer effect communicate its dependencies on other layers and effects to AE's caching system?

Dependencies must be explicitly registered through the AEGP API when your renderer effect enumerates and checks out layers. When you call layer checkout functions to read pixel data, masks, shapes, and transforms from dependent layers, and query properties of effects on those layers, AE tracks these accesses as dependencies. To ensure proper cache invalidation, you should checkout all relevant layer properties at render time—if any of those checked-out properties change, AE will invalidate the cached result and trigger a re-render. This allows the disk cache to function correctly by only using cached data when all dependencies are identical.

*Tags: `aegp`, `caching`, `layer-checkout`, `compute-cache`, `dependency`*

---

## Is it possible to make a multi-layer 2D global illumination effect where a renderer effect discovers material metadata effects on other layers and communicates these dependencies to AE's caching system?

It is theoretically possible using AEGP_RenderAndCheckoutLayerFrame to enumerate layers and pass their GUIDs via GuidMixInPtr() to register dependencies. However, a more practical approach is to use hidden layer parameters (up to 999) with a script button that assigns layers from the composition to these parameters, allowing checkout_layer() to work efficiently with SmartPreRender. You can also mix in layer order information via GuidMixInPtr() to handle dynamic layer changes without manual updates.

*Tags: `aegp`, `smart-render`, `layer-checkout`, `caching`, `compute-cache`*

---

## Can checkout_layer() be used on arbitrary composition layers during PF_Cmd_SMART_PRE_RENDER that aren't declared as effect parameters?

According to the SDK documentation, you can only grab layer parameters that are explicitly declared as layer parameters on the effect. To work with arbitrary layers, use hidden layer parameters that a script can populate, or use AEGP_RenderAndCheckoutLayerFrame which works with any layer IDs.

*Tags: `smart-render`, `layer-checkout`, `aegp`*

---

## Does AEGP_RenderAndCheckoutLayerFrame perform the expensive rendering immediately or defer it until AEGP_GetReceiptWorld is called?

AEGP_RenderAndCheckoutLayerFrame performs the expensive render operation immediately and returns an AEGP_FrameReceiptH. You can then call AEGP_GetReceiptGuid() on the receipt to get a GUID that can be mixed into GuidMixInPtr() for dependency tracking. There is both a synchronous and asynchronous version available; the async version could potentially be called during SmartPreRender and awaited during the render call/thread.

*Tags: `aegp`, `smart-render`, `layer-checkout`, `threading`*

---

## How should you handle caching when scraping data from dummy effects on other layers?

Call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects, and AE will cache everything fine.

*Tags: `aegp`, `caching`, `layer-checkout`, `memory`*

---

## How can you keep layer parameters synced between multiple locations when users apply keyframes and expressions to one?

This is acknowledged as a difficult problem. One approach mentioned is using AEGP or a script in userchangedparam to keep them synced, but keeping them synchronized when keyframes and expressions are applied to one location and mirroring it constantly is noted as being challenging.

*Tags: `aegp`, `scripting`, `params`, `ui`*

---

## How do 3D Camera Tracker and Warp Stabilizer output progress over frames with live updates without cache purging?

According to an After Effects developer, these built-in tools use internal APIs that are not available to regular plugin developers. The Warp Stabilizer effect 'cheats' and does not operate within the constraints of a normal AE Effect API plug-in. Adobe's internal team has access to secret suites that allow this functionality, which is not exposed to third-party developers. A feature request was made in 2023 to expose this capability to plugin developers, but it remains unavailable.

*Tags: `mfr`, `smart-render`, `caching`, `aegp`, `debugging`*

---

## How can you pass parameters between C and scripting languages in After Effects plugins?

Parameters can be passed as text using sprintf formatting. For example, parameters are formatted as text strings like "paramname=value" and then parsed by the scripting layer. Additionally, some bindings libraries (like PyCairo for Python or custom XML-based C binders for Lua) provide C APIs for assigning bitmap data and layer parameters directly between the compiled C code and the scripting layer.

```cpp
sprintf("%s=%s\n", paramname, value)
```

*Tags: `scripting`, `params`, `aegp`*

---

## What resources exist to understand sequence data in After Effects plugin development?

Tobias Fleischer (reduxFX) has created comprehensive documentation about sequence data that has helped developers understand the concept clearly. This resource is considered essential reference material for working with sequence data in After Effects plugins.

*Tags: `sequence-data`, `reference`, `aegp`, `documentation`*

---

## What alignment guarantees does PF_LayerDef::rowbytes have in After Effects?

rowbytes will always be a multiple of sizeof(PF_Pixel8), which is 4 bytes, but 16-byte alignment is not guaranteed. The minimum guarantee is that rowbytes must be a multiple of the alignment requirement of the pixel type being used. Since After Effects effects don't have pixel types smaller than ARGB8, rowbytes will always be at least 4-byte aligned in practice. Misaligned pointers would result in undefined behavior according to the C specification.

*Tags: `memory`, `params`, `aegp`, `reference`*

---

## What causes a plugin to not work properly in Premiere when it works in After Effects?

The issue can stem from the no_params_vary flag. In After Effects, this can be replaced using MIX_GUI during the pre_render thread. However, this flag has no effect in Premiere, so alternative approaches are needed for cross-application compatibility.

*Tags: `premiere`, `params`, `render-loop`, `cross-platform`, `aegp`*

---

## Why does a plugin fail with 'Not able to acquire AEFX Suite' error when playing in Premiere?

The issue may occur during render/playback if the plugin calls AEFX Suite in a shared thread or has incorrect host application detection logic. Check if there is conditional code that calls AEFX Suite only when the host is NOT Premiere (e.g., `if (app_id != 'PrMr') => call AEFX_suite`), but the host detection returns an incorrect value in Premiere. This can cause the plugin to attempt AEFX Suite calls in Premiere where they are not available. Premiere can also have unexpected behavior with certain suites, such as colorspace-related ones.

*Tags: `premiere`, `aegp`, `threading`, `debugging`*

---

## How can you set sequence data values in After Effects plugins?

You can use the out_data property which has sequence_data as a property that can be used to set the values of sequence data. This approach works in After Effects 2021 and 2022 (in 2022, it will work without the MFR flag set, though a warning may be shown).

*Tags: `sequence-data`, `mfr`, `aegp`*

---

## What is the current behavior of the After Effects Memory Suite functions?

According to discussions on the aescripts Slack channel, the Memory Suite functions originally worked as intended in earlier versions of After Effects. However, in current versions, these functions essentially just call new and delete, suggesting they may no longer provide the specialized memory management they once did. This is important context when deciding how to handle memory allocation and deallocation in plugins.

*Tags: `memory`, `aegp`, `reference`*

---

## How do you expand the result rectangle in After Effects plugin rendering?

To expand the in_result.result_rect and in_result.max_result_rect, modify them before calling UnionLRect. The typical pattern is to expand these rectangles first, then use UnionLRect to union in_result.max_result_rect with in_result.result_rect to make the result rect the same as max result rect, and finally union with the extra->output->result_rect.

```cpp
UnionLRect(&in_result.max_result_rect, &in_result.result_rect); // make result rect the same as max result rect
UnionLRect(&in_result.max_result_rect, &extra->output->result_rect);
```

*Tags: `output-rect`, `render-loop`, `aegp`*

---

## Why must X and Y coordinates be cast to FIX when using subpixel_sample_float, and what sampling method should be used for smoother displacement results?

When using the SamplingFloatSuite1()->subpixel_sample_float() function, coordinates are cast to FIX format (fixed-point representation) rather than remaining as floats. This is part of the After Effects API's internal coordinate system. The FIX format provides the necessary precision for the sampling algorithm while maintaining compatibility with AE's rendering pipeline. For smoother displacement results comparable to After Effects' native displacement or professional plugins, ensure you're using subpixel_sample_float with proper coordinate conversion, and consider whether your displacement values are being calculated with sufficient precision before being passed to the sampling function. The quality difference may also stem from how displacement offsets are computed rather than the sampling function itself.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
        INT2FIX(newX),
        INT2FIX(newY),
        &giP->inSamp_pb,
        &samplePixel));
```

*Tags: `sampling`, `displacement`, `mfr`, `output-rect`, `aegp`*

---

## How can you implement an iterative render loop where a frame needs to be re-rendered multiple times until a calculation is complete?

According to gabgren, you can call render() multiple times within a single frame processing cycle. This requires a flag to tell After Effects to re-render the frame, and inside the render function, an if clause should check whether the render is actually done or not. This pattern was used in threads discussing multi-pass rendering approaches.

*Tags: `render-loop`, `smart-render`, `aegp`*

---

## What causes an infinite progress bar at the end of an effect render?

An infinite progress bar can be triggered by an incoherent PF_Progress value. Setting impossible values like 200/100 (where the current progress exceeds the total) will display this infinite progress bar effect.

*Tags: `ui`, `debugging`, `aegp`*

---

## Is it illegal to checkout a parameter multiple times in After Effects plugins?

Checkout of parameters can be problematic in certain lifecycle stages. You cannot perform parameter checkout during sequence setup, resetup (when AE is loading the project), or sequence setdown. It's important to handle checkout errors properly and ensure that checkin is called immediately after using the parameter value, rather than holding the checkout open.

*Tags: `params`, `layer-checkout`, `debugging`, `aegp`*

---

## Should layer checkout and check-in be done immediately after use in a render loop, or can check-in be deferred?

Layer checkout and check-in should be handled carefully in render loops. For each frame checked out, you should check-in immediately after use, even if an error occurs. The recommended pattern is: for each frame, checkout the frame, perform work if no error occurs, then check-in the frame (using Err2 or equivalent error handling). Alternatively, you can defer check-in until the end of the render thread, but every checked-out layer must be checked in whether it was used or not. Error handling during checkout must also be monitored.

```cpp
For X
    Checkout frame x
    If (no err) do job
    Err2(check-in frame x)
End of loop
```

*Tags: `layer-checkout`, `render-loop`, `memory`, `aegp`, `debugging`*

---

## Is it possible to set or change the value of a LAYER_PARAM in a plugin UI?

Yes, you can set/change the value of a LAYER_PARAM in your plugin UI by setting the layer ID value in a streamVal2 using SetStreamValue.

*Tags: `params`, `ui`, `aegp`*

---

## Can you write to sequence data during PF_update_param_ui callback?

No, you cannot write to sequence data during PF_update_param_ui. Writing to sequence data is only possible in the user changed param callback. With MFR, you also cannot write to sequence data during PF_update_param_ui. However, in certain cases you may be able to accomplish similar goals without AEGP by using PF_LayerDefault_NONE or similar approaches, though AEGP is often the recommended solution.

*Tags: `mfr`, `params`, `sequence-data`, `aegp`*

---

## How do you set a layer default parameter value in After Effects plugins?

Layer definitions do not have a direct 'value' property. Instead, you can use PF_LayerDefault_NONE or similar PF_LayerDefaults constants to set layer default parameters. For more complex parameter manipulation, using the AEGP API is recommended as an alternative approach.

```cpp
param.u.ld = PF_LayerDefault_NONE;
```

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How do you transform layer-space masked bounds to world space using matrix operations in After Effects plugins?

Use AEGP_GetLayerMaskedBounds to get the mask rectangle in layer space, then transform the corner points using the layer-to-world matrix. Create a 4x4 matrix from the 2D point coordinates, multiply it by the layer2WorldMat using AEGP_MultiplyMatrix4, then decompose the result with AEGP_MatrixDecompose4 to extract the final position. The matrix multiplication order matters—multiply the point matrix first, then the layer-to-world transformation.

```cpp
suites.LayerSuite8()->AEGP_GetLayerMaskedBounds(layerH, AEGP_LTimeMode_LayerTime, &curTime, &maskRect);
A_FloatPoint3 topLeft = { maskRect.left, maskRect.top, posVP.z };
A_FloatPoint3 botRight = { maskRect.right, maskRect.bottom, posVP.z };
topLeftP = Multiply2DPointBy4x4Matrix(in_data, out_data, topLeft, myDependencies.layer2WorldMat);
botRightP = Multiply2DPointBy4x4Matrix(in_data, out_data, botRight, myDependencies.layer2WorldMat);
```

*Tags: `aegp`, `matrix-math`, `layer-transform`, `coordinate-space`*

---

## How do you toggle the visibility of a UI parameter in After Effects?

To toggle parameter visibility, use the DynamicStreamSuite2 in the ParamUI thread rather than directly manipulating ui_flags. The issue with toggling ui_flags is that the bitwise NOT operator (!) doesn't work as expected for flag manipulation. Instead, get the effect and stream references, then use AEGP_SetDynamicStreamFlag with the AEGP_DynStreamFlag_HIDDEN flag. Set the last parameter to !paramVisibleB where paramVisibleB is true to show and false to hide the parameter. Note that Premiere Pro requires a different approach.

```cpp
static PF_Err
changeParamVisibility(PF_InData            *in_data,
                      PF_OutData            *out_data,
                      PF_ParamDef          *paramsDef,
                      PF_ParamIndex        paramIndex,
                      PF_Boolean           paramVisibleB)
{
    PF_Err                err                    = PF_Err_NONE,
    err2                = PF_Err_NONE;
    global_dataP        globP                = reinterpret_cast<global_dataP>(DH(out_data->global_data));
    AEGP_SuiteHandler        suites(in_data->pica_basicP);
    if (!err && globP && in_data->appl_id != 'PrMr')
    {
        AEGP_EffectRefH            meH                = NULL;
        AEGP_StreamRefH      currStreamH        = NULL;
        ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
        ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, paramIndex,     &currStreamH));
        if (meH && currStreamH)
        {
            ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(currStreamH, AEGP_DynStreamFlag_HIDDEN, FALSE, !paramVisibleB));
        }
        if (meH){
            ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
        }
        if ( currStreamH ){
            ERR2(suites.StreamSuite2()->AEGP_DisposeStream( currStreamH ));
        }
    }
    return err;
}
```

*Tags: `ui`, `params`, `aegp`, `premiere`*

---

## How can I get all of the active camera's properties in an After Effects plugin?

The Resizer example in the After Effects SDK demonstrates how to work with camera layers and access their properties. This example is a good reference for interfacing with camera layers and retrieving their active properties.

*Tags: `aegp`, `reference`, `sdk`*

---

## What AEGP method should be used to render and checkout a layer frame?

AEGP_RenderAndCheckoutLayerFrame is the AEGP method specifically implemented for rendering and checking out layer frames.

*Tags: `aegp`, `layer-checkout`, `render-loop`*

---

## How do I convert an After Effects matrix from AEGP_GetLayerToWorldXform to OpenGL coordinate system?

After Effects uses a row-major matrix layout (0 1 2 3 / 4 5 6 7 / 8 9 10 11 / 12 13 14 15) while OpenGL uses column-major (0 4 8 12 / 1 5 9 13 / 2 6 10 14 / 3 7 11 15). To convert, transpose the matrix by iterating through rows and columns: for each x from 0 to 3, set glMatrix[x][0] = matrix.mat[0][x], glMatrix[x][1] = matrix.mat[1][x], glMatrix[x][2] = matrix.mat[2][x], glMatrix[x][3] = matrix.mat[3][x]. You may also need matrix operations like inversion, for which the GLM library is recommended.

```cpp
for (x = 0; x < 4; x++)
{
    glMatrix[x][0] = matrix.mat[0][x];
    glMatrix[x][1] = matrix.mat[1][x];
    glMatrix[x][2] = matrix.mat[2][x];
    glMatrix[x][3] = matrix.mat[3][x];
}
```

*Tags: `aegp`, `opengl`, `matrix`, `coordinate-conversion`, `camera`*

---

## What resources explain how to convert After Effects camera matrices to OpenGL format?

Two archived Adobe Community threads provide detailed guidance on matrix coordinate system conversion for AE camera data. The first thread at https://community.adobe.com/t5/after-effects-discussions/glator-for-dummies-and-from-dummy/m-p/6930311 discusses camera matrix conversion generally. The second thread (archived at https://web.archive.org/web/20111223234233/http://forums.adobe.com/thread/570135) provides specific code examples for transposing AE matrices to OpenGL format, explaining the row-major vs column-major difference and matrix operations needed.

*Tags: `aegp`, `opengl`, `camera`, `reference`, `matrix`*

---

## How do you get a Layer Handle from a layer parameter (PF_ADD_LAYER) in After Effects plugins?

Based on investigation, layer parameters (PF_ADD_LAYER) only provide access to the rendered World (pixels) of the selected layer, not the actual layer definition or handle. This means you cannot directly retrieve layer properties or streams through a layer parameter. The available suite functions do not support this capability, so alternative approaches must be used instead of relying on the layer parameter to access layer definitions.

*Tags: `params`, `aegp`, `layer-checkout`*

---

## How do you retrieve a layer parameter value and get the actual layer handle from an effect in After Effects?

Use AEGP_GetNewEffectForEffect to get the effect handle, then AEGP_GetNewEffectStreamByIndex to get the stream for the parameter. Call AEGP_GetNewStreamValue to get the layer ID value, then use AEGP_GetLayerFromLayerID with the parent composition handle to get the actual layer handle. Remember to dispose of the effect handle and stream handles after use.

```cpp
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, param_index, &currStreamH));
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &currLayerH));
ERR(suites.LayerSuite5()->AEGP_GetLayerParentComp(currLayerH, &compPH));
if (meH && currStreamH && currLayerH && compPH) {
    ERR(suites.StreamSuite3()->AEGP_GetNewStreamValue(globP->my_id, currStreamH, AEGP_LTimeMode_LayerTime, &currTime, TRUE, &val));
    if (!err) {
        AEGP_LayerIDVal layerId = val.val.layer_id;
        ERR(suites.LayerSuite8()->AEGP_GetLayerFromLayerID(compPH, layerId, &targetLayerH));
    }
}
```

*Tags: `aegp`, `params`, `layer-checkout`*

---

## What is the difference between checking out a layer parameter with PF_Effect_world versus using AEGP streams?

When you checkout a layer parameter using the PF interface, you get a PF_Effect_world directly. When using AEGP streams, you get an index value instead, which you then need to resolve to an actual layer handle using the AEGP layer suite functions.

*Tags: `aegp`, `params`, `layer-checkout`*

---

## Can After Effects plugins currently access the Param source type for effects, masks, or sources through the dropdown list?

As of February 2023, there is no direct access to the Param source type effect/mask/source dropdown list in the After Effects plugin API. This limitation prevents plugins from programmatically controlling or reading which layer or effect is selected as a source parameter.

*Tags: `params`, `aegp`, `limitation`*

---

## How do you reproduce warp stabilizer banner messages in the viewport—using the Canvas suite or drawing directly on the rendered frame?

The user is asking about the proper approach to display banner messages similar to the warp stabilizer effect. This involves choosing between using the Canvas suite for viewport drawing versus drawing directly on the rendered frame output.

*Tags: `ui`, `render-loop`, `aegp`, `output-rect`*

---

## How can you obtain a drawing reference in After Effects plugin render loops without requiring user interaction?

The standard approach using EffectCustomUISuite1()->PF_GetDrawingReference() only provides a drawing context during PF_Cmd_EVENT callbacks triggered by user interaction. To draw during render without user interaction (similar to VFX Stabilizer behavior), you need an alternative method to access the drawing reference that doesn't depend on the extraPV parameter passed through PF_EventExtra. This requires investigating lower-level drawing APIs or background rendering approaches that don't rely on event-driven context delivery.

*Tags: `ui`, `render-loop`, `aegp`, `debugging`*

---

## How can you draw to the composition window during rendering without relying on user interaction events?

The challenge is that PF_GetDrawingReference() is only available through PF_EventExtra during PF_Cmd_EVENT, which requires user interaction with the effect parameters or composition preview. To draw during a render loop without user interaction (similar to VFX Stabilizer behavior), you need an alternative approach to obtain the drawing reference that isn't tied to user interaction events. This is a difficult problem with limited documented solutions; consulting experienced plugin developers like Shachar Gran is recommended.

*Tags: `ui`, `render-loop`, `aegp`, `debugging`*

---

## Why is the paramUI suite slower when changing a parameter name in After Effects 2023?

The paramUI suite has been observed to be slower when changing a parameter name in After Effects 2023 (version 23.2.1). An alternative approach using the streamSuite has been found to work well as a workaround for this performance issue.

*Tags: `params`, `ui`, `performance`, `aegp`*

---

## How do you load machine learning models in an After Effects plugin without shipping them with the tool?

Models can be pulled from Hugging Face on first use rather than being included with the plugin distribution. This approach reduces initial download size and keeps models up to date.

*Tags: `deployment`, `scripting`, `aegp`*

---

## What approach can be used to create Python-based After Effects plugins?

A Python/AE/C++ bridge can be built to enable Python plugin development for After Effects. This allows developers to write plugins in Python while maintaining compatibility with AE's native plugin system, though the implementation can be complex and require several months of development to achieve stability and polish.

*Tags: `scripting`, `build`, `cross-platform`, `aegp`*

---

## How do you print to the console when debugging an After Effects plugin?

On Mac, use printf. On Windows, use OutputDebugStringA macro to output debug messages that will appear in Visual Studio's console. Note that AEGP_WriteToOSConsole from the utility suite exists but may not work reliably. Starting AE with the -debug flag can help, but OutputDebugStringA is the more reliable approach for Windows debugging.

```cpp
OutputDebugStringA("Debug message here");
```

*Tags: `debugging`, `windows`, `macos`, `aegp`*

---

## How do you add time values when using the compute cache API in After Effects?

When adding time values to the compute cache, you need to use AEGP (After Effects General Plugin) APIs to properly interact with the time-based caching system.

*Tags: `compute-cache`, `aegp`, `caching`*

---

## Is there a way to communicate between a CEP panel and a C++ plugin in After Effects?

Direct communication between CEP and C++ plugins is not possible. However, indirect communication can be achieved: a C++ plugin can open a CEP panel by using the ExecuteScript to find the Command ID (menu) by the extension name, then executing that command through the command suite. CEP cannot directly change plugin parameter values, and the plugin cannot directly check if a CEP panel is open or bring it into focus.

*Tags: `cep`, `c++`, `scripting`, `ui`, `aegp`*

---

## How do you invalidate cached frames in After Effects so they will be re-rendered instead of using the cache?

This question was asked about invalidating rendered frames to force After Effects to re-render them, similar to how built-in effects like Warp Stabilizer and 3D Camera Tracker work. However, no answer was provided in the conversation.

*Tags: `caching`, `render-loop`, `aegp`, `smart-render`*

---

## How can you trigger frame invalidation from external background threads in After Effects plugins?

GuidMixInPtr can be problematic for triggering invalidation from background threads when using global or sequence data variables. A reliable workaround is to change a composition property (like background color) via AEGP calls or scripting, which forces After Effects to check if frames need invalidation. Another approach is to use a hidden parameter that gets modified through AEGP or scripting calls, though this has limitations when modal dialogs are open. The method must ultimately signal to After Effects through a mechanism it actively monitors.

*Tags: `smart-render`, `threading`, `sequence-data`, `params`, `aegp`*

---

## How should DrawImage be called in drawbotSuites to ensure correct opacity and alpha rendering?

When using drawbotSuites.surface_suiteP->DrawImage, you must use an opacity of 1.0. If you don't use 1.0 opacity, the drawing will appear darker than intended and the alpha will still show as 1.0 since After Effects' UI will be behind it with solid alpha. This is important for correct compositing behavior.

```cpp
drawbotSuites.surface_suiteP->DrawImage(/* ... */, 1.0 /* opacity */, /* ... */)
```

*Tags: `ui`, `drawing`, `aegp`, `debugging`*

---

## Why does text appear faint when using FillPath and StrokePath in After Effects plugin UI?

When writing on UI using FillPath and StrokePath with opacity set to 1.0, faint text may result from After Effects UI brightness preferences or project color space compensation. The After Effects suite may automatically compensate for these settings, affecting how drawing operations are rendered on the UI.

*Tags: `ui`, `debugging`, `aegp`*

---

## How can you implement a custom text editor in the Effect Control Window that captures key events like Tab?

When implementing a custom text editor in the ECW using Custom UI events, PF_Event_KEYDOWN events are available but don't include an index for the particular param. Additionally, some keys like Tab don't reach the KEYDOWN handler. To solve this, you need to claim key-focus similar to how built-in slider numeric value editing works, which allows the plugin to intercept all keyboard input including Tab and other special keys that would normally be consumed by the host.

*Tags: `ui`, `custom-ui`, `aegp`, `debugging`*

---

## How can you implement a button in an After Effects plugin that triggers a script and passes data to it?

Use an arbitrary data parameter to store text/script content, then trigger AEGP_ExecuteScript() via a button event to execute the script string. You can manipulate the script string (e.g., via find-and-replace) before execution to pass data into it, and the script can return data back to After Effects.

*Tags: `aegp`, `arb-data`, `ui`, `scripting`*

---

## How can a plugin implement recursive operations where frame N depends on computed data from previous frames without checking out all intermediate frames during pre-render?

This is a known challenge when combining smart render with compute caches that require sequential frame dependencies. The user attempted several approaches: (1) using a compute cache with smart render, but this requires checking out all previous frames during pre-render when rendering frame 100, which is memory-inefficient; (2) using checkout params during compute cache, which doesn't account for previous effects; (3) using AEGP_CacheAndCheckoutFrame, which doesn't trigger pre_render/smart render callbacks on checked-out frames. A solution may involve manually managing frame dependencies outside the smart render system or using layer checkout strategically to load only necessary intermediate results rather than relying on automatic frame dependency detection.

*Tags: `smart-render`, `compute-cache`, `layer-checkout`, `memory`, `aegp`, `sequence-data`*

---

## How can you debug plugin initialization issues by tracking function entry and exit points?

Add logging statements at the entry and exit of key plugin functions, particularly in global setup and parameter setup stages. This helps identify at which stage crashes or entrypoint errors are occurring, making it easier to distinguish between different error types.

*Tags: `debugging`, `aegp`, `plugin-development`*

---

## Why is PF_Arbitrary_NEW_FUNC selector never received, causing arbitrary data to always be zero length initially?

This is a known behavior where the PF_Arbitrary_NEW_FUNC selector is not consistently triggered during arbitrary data initialization. The user reported that their arbitrary data always starts with zero length despite expecting the NEW_FUNC selector to be called. This appears to be an obscure aspect of the arbitrary data lifecycle that requires alternative initialization strategies.

*Tags: `arb-data`, `aegp`, `debugging`, `params`*

---

## Is the PF_Arbitrary_NEW_FUNC selector called in After Effects post-CS6 architecture?

No, PF_Arbitrary_NEW_FUNC is not called in the post-CS6 architecture. Instead, arbitrary data is initialized by allocating a default Arb handle during ParamSetup by calling a custom function to create the default arb data. For every new instance, After Effects sends a PF_Arbitrary_COPY_FUNC selector with your default arb handle. While the NEW_FUNC code can be left in place just in case, it will not be triggered in practice.

*Tags: `arb-data`, `params`, `aegp`, `reference`*

---

## How can you collapse a slider parameter in After Effects plugins?

To collapse a slider parameter, you need to set the global flag PF_OutFlag2_PARAM_GROUP_START_COLLAPSED_FLAG and then set the param flag with 'def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;'. However, this approach may not work for individual sliders in the same way it works for topic groups.

```cpp
def.flags &= PF_ParamFlag_COLLAPSE_TWIRLY;
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## Why is there a memory leak when using AEGP_memorysuite on macOS but not Windows during export?

A user reported a memory leak when using AEGP_memorysuite to allocate, lock, memcpy to/from, and unlock memory handles during export (not preview render). The leak was macOS-specific and did not occur on Windows. Testing with different flags (mem_NONE, mem_CLEAR, mem_QUIET) showed the leak was less severe with mem_QUIET. Replacing the AEGP memory handle with a temporary pointer using new/delete eliminated the leak. Additionally, replacing memcpy with strncpy solved the issue when copying from the mem pointer to a buffer, but not in the opposite direction. This suggests a potential platform-specific issue with how AEGP_memorysuite handles memory during the export pipeline, possibly related to threading or memory alignment differences between macOS and Windows.

*Tags: `memory`, `aegp`, `macos`, `windows`, `debugging`, `threading`*

---

## Is there a memory leak when using aegp_memHandle during the compute cache thread that persists even after calling the free memory function?

A developer reported that instruments showed no leaks, but memory allocated using aegp_memHandle during the compute cache thread was not being freed in reality when the freemem function was called. This issue occurred specifically during render operations, suggesting there may be a limitation with AEGP_memsuite during the render thread that prevents proper memory deallocation.

*Tags: `memory`, `aegp`, `compute-cache`, `render-loop`, `debugging`*

---

## Is it safe to allocate memory in one thread and free it in another thread in After Effects plugins?

It is generally not recommended to allocate memory in one thread and free it in another thread in After Effects plugins. AE makes few threading guarantees beyond pairing SETUP and SETDOWN calls. Even using mutex locks does not reliably solve cross-thread allocation/deallocation issues, as the plugin framework does not provide clear guarantees about thread lifecycle and frequency.

*Tags: `threading`, `memory`, `aegp`, `debugging`*

---

## What is the correct C++ syntax for initializing multiple PF_Err variables in After Effects plugins?

When declaring multiple PF_Err variables, you must initialize each one explicitly. Incorrect: `PF_Err err, err2 = PF_Err_NONE;` (only err2 is initialized, err is undefined). Correct: `PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;` (both variables initialized). This matters because uninitialized error codes will not evaluate to 0 in conditional statements like `if(!err)`, causing logic failures.

```cpp
// Incorrect - err is undefined
PF_Err err, err2 = PF_Err_NONE;

// Correct - both initialized
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `debugging`, `aegp`, `pipl`*

---

## Why doesn't Instruments detect memory leaks from PF_EffectWorlds allocated in After Effects plugins?

Memory leaks from PF_EffectWorlds may not be detected by Instruments because these objects are allocated deep within the After Effects engine rather than directly within the plugin code. The leak detection tools only see allocations at the plugin level, not at the internal AE engine memory allocation level where EffectWorlds are actually created. This means that even significant memory leaks from unmanaged EffectWorlds won't show up in Instruments despite causing the application to run out of memory.

*Tags: `memory`, `debugging`, `macos`, `aegp`*

---

## What is a good approach to manage memory safety for After Effects SDK objects?

Create C++ wrapper classes around AESDK objects to enable automatic memory management. This approach leverages C++ features like constructors and destructors to ensure proper allocation and deallocation of SDK objects. Additionally, there are already existing C++ wrappers for portions of the After Effects SDK available on GitHub that can be referenced or reused for this purpose.

*Tags: `memory`, `open-source`, `reference`, `aegp`, `build`*

---

## How can you manage memory for AESDK objects in C++ plugins?

Create C++ wrappers around all AESDK objects to get automatic memory management. This approach helps prevent memory leaks and ensures proper cleanup of After Effects SDK resources.

*Tags: `memory`, `aegp`, `cpp`, `plugin-development`*

---

## Why does macOS Instruments fail to detect memory leaks from undisposed PF_EffectWorlds in After Effects plugins?

James Whiffin reported that while Instruments successfully detected memory leaks from data allocated in pre-render functions that weren't disposed, it failed to detect leaks from PF_EffectWorlds that were allocated but never disposed. Despite these allocations consuming gigabytes and causing After Effects to run out of memory, they didn't appear in Instruments' leak detection output even when sorted by size. This suggests that PF_EffectWorlds may be allocated through memory management mechanisms that Instruments cannot track, or that After Effects' memory management for effect worlds operates outside standard system memory tracking.

*Tags: `memory`, `macos`, `debugging`, `aegp`*

---

## How can I get text layer path vertices in true layer space that match expression path.points() behavior, especially when the layer is parented?

PF_PathDataSuite does not return vertices in true layer space as documented in the SDK. Instead, it returns vertices in a hybrid space that is affected by 2D transforms (xy-translation, shear from parented transforms, Z Rotation) but not by z-translation, Orientation, or X/Y Rotation properties. To convert PF_PathDataSuite output to true layer space (matching what path.points() returns in expressions), you need to reverse the 2D transforms while preserving the unaffected 3D properties. Use AEGP_GetLayerToWorldXform() to convert to world space after obtaining layer-space vertices. The discrepancy exists because PF_PathDataSuite appears to return vertices in a space between composition and layer space, unlike expressions which correctly return layer-space coordinates independent of layer transform properties.

*Tags: `aegp`, `layer-checkout`, `params`, `reference`, `cross-platform`*

---

## What is the official Adobe documentation for working with path data in After Effects plugins?

The Adobe After Effects Plugin SDK documentation on working with paths is available at https://ae-plugins.docsforadobe.dev/effect-details/working-with-paths.html. This guide describes PF_PathDataSuite and claims that returned vertices are in the layer's coordinate space, though developers have found discrepancies between this documentation and actual behavior when dealing with transformed and parented layers.

*Tags: `reference`, `aegp`, `documentation`*

---

## Why might an AEGP not appear in After Effects and fail to hit any breakpoints during debugging?

This is a Mac-specific AEGP question about plugin visibility and debugging. The issue could be related to security settings or allocation changes on macOS, particularly around code signing, entitlements, or plugin sandbox restrictions. Possible causes include: incorrect plugin bundle structure, code signing issues, missing entitlements, or the plugin not being properly registered in After Effects' plugin directory.

*Tags: `aegp`, `macos`, `debugging`, `deployment`*

---

## How can I check out the current layer with PF_CHECKOUT at different times while preserving all previously applied effects?

When using PF_CHECKOUT to check out the INPUT layer at different times, it returns the effectworld before all effects are applied, including effects applied before the current effect. However, this is possible to achieve with all effects applied, as demonstrated by the Echo effect. The solution involves understanding the layer checkout mechanism and potentially using different checkout strategies or timing parameters to access the fully composited layer rather than the raw input.

*Tags: `layer-checkout`, `aegp`, `mfr`, `render-loop`*

---

## Why does checking out a layer parameter in PreRender still return the layer before all effects are applied?

When using PF_CHECKOUT_PARAM to check out a layer (INPUT) in the PreRender callback, the layer is returned in its pre-effect state rather than with effects applied. This is expected behavior in the AE plugin architecture—the checkout at PreRender stage occurs before the effect chain is fully processed.

*Tags: `aegp`, `layer-checkout`, `render-loop`, `debugging`*

---

## What is the difference between checkout_layer_pixels and PF_Checkout?

checkout_layer_pixels should be used instead of PF_Checkout for layer pixel operations in After Effects plugins.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How can you create a custom banner in the After Effects effects menu like FxFactory plugins do?

This appears to be a FxFactory-specific feature. The discussion suggests several investigation approaches: comparing AE installation folders before and after FxFactory installation to identify changes, examining the .aex plugin file and its PiPL resources for unknown SDK functions, or contacting FxFactory developers directly. It's unclear whether the banner is drawn through standard AEGP suite functions or through custom modifications to the plugin manager.

*Tags: `pipl`, `ui`, `reverse-engineering`, `aegp`, `debugging`*

---

## Is there a way for an effect to programmatically reset itself using a command ID instead of manually setting all parameters to defaults via streams?

James Whiffin asked about programmatically resetting an effect using a command ID rather than manually resetting each parameter through streams. This question was not answered in the conversation.

*Tags: `params`, `aegp`, `pipl`*

---

## Is there a way for an effect to programmatically reset itself instead of manually setting all parameters to their defaults?

James Whiffin asked whether effects can use a command ID or similar mechanism to reset themselves programmatically rather than manually setting each parameter to default values via streams. This question was asked but no answer was provided in the conversation.

*Tags: `params`, `aegp`, `pipl`*

---

## Does Media Encoder load AEGP plugins?

No, Media Encoder does not load AEGP plugins. If your workflow relies on AEGP plugins for rendering, you cannot use Media Encoder as an alternative.

*Tags: `aegp`, `deployment`, `premiere`*

---

## How do you use the Compute Cache API in an After Effects plugin?

The Compute Cache API requires implementing four callback functions: a key generation function (MyGenerateKeyFunc) that identifies the effect ref, layer id, effect position, current time and time scale; a compute function (MyComputeFunc) that performs the actual computation; an approximate size function (MyApproxSizeValueFunc) that returns the memory size of cached values; and a delete function (MyDeleteComputeValueFunc) that frees memory. Register these callbacks in GlobalSetup using AEGP_ComputeCacheSuite1 with AEGP_ClassRegister. The key function should include effect reference, layer id, effect position, current time and time scale to enable per-frame effect caching.

```cpp
#include "AE_ComputeCacheSuite.h"
A_Err MyGenerateKeyFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeKeyP out_keyP) {
    LOG("Generate Key function");
    return A_Err_NONE;
}
A_Err MyComputeFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeValueRefconP *out_valuePP) {
    LOG("Compute Functions");
    return A_Err_NONE;
}
size_t MyApproxSizeValueFunc(AEGP_CCComputeValueRefconP valueP) {
    LOG("Obtain size function");
    return 0;
}
void MyDeleteComputeValueFunc(AEGP_CCComputeValueRefconP valueP) {
    LOG("Delete function");
}
static PF_Err GlobalSetup(PF_InData* in_data, PF_OutData* out_data, PF_ParamDef* params[], PF_LayerDef* output) {
    AEFX_SuiteScoper<AEGP_ComputeCacheSuite1> compute_suite = AEFX_SuiteScoper<AEGP_ComputeCacheSuite1>(in_data, kAEGPComputeCacheSuite, kAEGPComputeCacheSuiteVersion1);
    AEGP_ComputeCacheCallbacks callbacks = { MyGenerateKeyFunc, MyComputeFunc, MyApproxSizeValueFunc, MyDeleteComputeValueFunc };
    compute_suite->AEGP_ClassRegister("com.mycompany.effect.myComputeCacheClass", &callbacks);
}
```

*Tags: `compute-cache`, `aegp`, `memory`, `caching`*

---

## Is Media Encoder compatible with AEGP plugins for rendering After Effects projects?

AEGP plugins are not supported by Adobe Media Encoder (AME). If your plugin requires AEGP functionality to render, you will need to convert the AEGP plugin into a standalone or command-line tool to use with Media Encoder.

*Tags: `aegp`, `deployment`, `premiere`, `tool`*

---

## How can you generate a compute cache key when the key generation function only accepts AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP but requires AEGP_HashSuite1 which needs in_data?

Tim Constantinov asked about the apparent conflict between the parameters accepted by the key generation function and the requirements for using AEGP_HashSuite1. The question highlights that to generate a key, you need access to in_data or in_data->pica_basicP, but the function signature seems to only provide AEGP_CCComputeOptionsRefconP and AEGP_CCComputeKeyP. This appears to be an unresolved technical question about the proper API usage pattern.

*Tags: `compute-cache`, `aegp`, `debugging`*

---

## How do you generate a key and create a pointer for computeIfNeeded or reading a function in After Effects plugins?

When you call computeIfNeeded or want to read a function, you create a pointer and must set in_data->pica_basicP in this pointer along with the other necessary values. The compute cache operates as something independent where you must send everything needed for the computation.

*Tags: `aegp`, `compute-cache`, `smart-render`, `params`*

---

## How do you properly access the compute cache in After Effects plugins and avoid crashes?

When accessing compute cache data, you must create a pointer to access_cache_data and extract the SPBasicSuite from in_data->pica_basicP. To avoid crashes and maintain independence from the render thread, create a new object and copy only the parts you need, then delete it after computing. This prevents thread safety issues and ensures the cache operates independently.

```cpp
access_cache_data* accessCacheData = static_cast<access_cache_data*>(optionsP);
if ( !accessCacheData->in_data ) {return A_Err_ALLOC;}
SPBasicSuite* bsuite;
bsuite = accessCacheData->in_data->pica_basicP;
if (!bsuite){return A_Err_ALLOC; }

struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
};
```

*Tags: `compute-cache`, `memory`, `threading`, `aegp`, `debugging`*

---

## How can you set a layer to be a trackmatte using AEGPSetLayerTransferMode with backwards compatibility before AE 23?

James Whiffin reported that AEGPSetLayerTransferMode only works to set trackmatte mode if the layer is already a trackmatte, but won't convert a non-trackmatte layer to one. He notes that AE 23+ has a new method for this purpose, but for backwards compatibility with earlier versions, the standard AEGP_SetLayerTransferMode approach has this limitation. The code structure involves creating an AEGP_LayerTransferMode struct with the desired transfer mode and track matte type, then calling the LayerSuite7 function.

```cpp
AEGP_LayerTransferMode mode = {};
mode.mode = PF_Xfer_IN_FRONT;
mode.track_matte = AEGP_TrackMatte_ALPHA;
mode.flags = 0;
ERR(suites.LayerSuite7()->AEGP_SetLayerTransferMode(layerH, &mode));
```

*Tags: `aegp`, `layer-checkout`, `backwards-compatible`*

---

## How can I force After Effects to invalidate cached frames when I modify global data in my plugin?

Simply setting PF_OutFlag_FORCE_RERENDER in outflags while changing a GuidMixInPtr value in extra->cb may not always work reliably. One workaround is to "kick" AE by triggering an internal frame cache check, such as changing the composition background color and changing it back. However, if you properly use GuidMixInPtr for forcing rerenders, it should work without this workaround—if you're experiencing issues, it may be worth reviewing source code examples from developers who have successfully implemented this.

*Tags: `compute-cache`, `smartfx`, `aegp`, `params`*

---

## What suites are unsafe to call from the Death Hook in After Effects plugins?

There are restrictions on which suites can be safely called from the Death Hook. The Utility Suite, specifically ReportInfo, has been reported to cause unhandled exceptions when called from Death Hook. This suggests that certain suites with external dependencies or state management are not safe to invoke during plugin shutdown, and developers should be cautious about suite usage in Death Hook contexts.

*Tags: `aegp`, `debugging`, `memory`, `threading`*

---

## Can one effect render from another effect during smartrender in After Effects?

During smartrender, it is not straightforward to have one effect call another effect's render. Smartrender happens on the render thread, and AE will not call smartrender on other layers if a current smartrender is already in progress—it waits until the current smartrender completes. One possible approach mentioned is to use EffectCallGeneric with a custom passthrough code if the other plugin implements it, but this has significant limitations. AEGP calls cannot be made from the render thread directly; they must be delegated to the UI thread. The async render and checkout functions (AEGP_RenderAndCheckoutLayerFrame_Async) might work from any thread but are noted as buggy. Ultimately, the questioner settled on manually implementing effects internally rather than calling other plugins dynamically.

*Tags: `smart-render`, `aegp`, `threading`, `render-loop`*

---

## Why does After Effects throw an 'effect attempting to modify a locked project' error when adjusting effect parameters during smartrender?

During a smartrender command, the project becomes locked to prevent modifications. Attempting to use AEGP calls to adjust effect parameters while smartrender is in progress results in this error because AEGP operations that modify the project cannot execute on the render thread. Such modifications must be deferred to the UI thread after the smartrender completes.

*Tags: `smart-render`, `aegp`, `threading`, `params`*

---

## What is plugplug.DLL and how can it be used for inter-plugin communication?

plugplug.DLL is a DLL that allows After Effects plugins to communicate with each other. You can call the extern "C" functions exposed by plugplug.DLL from your C++ plugin to enable inter-plugin communication. This concept is related to loading plugins from within other plugins, as discussed in the Adobe community post: https://community.adobe.com/t5/after-effects-discussions/how-to-load-a-plugin-from-another-plugin/td-p/6738182

*Tags: `aegp`, `cross-platform`, `windows`, `deployment`, `reference`*

---

## Is it possible to include multiple PiPLs in a single plugin file, and what are the recommendations?

Yes, it is technically possible to include multiple PiPLs (both AEGPs and effects) in the same file, but it is not recommended. If you do use multiple PiPLs in the same file, AEGPs must come first. However, no other hosts (not even Premiere Pro) support multiple PiPLs pointing to multiple effects within the same .dll or code fragment. The recommendation from the SDK is to use one PiPL and one plugin per code fragment, especially to avoid shipping new builds of all plugins when updating just one.

*Tags: `pipl`, `aegp`, `deployment`, `premiere`*

---

## When is PF_Cmd_GLOBAL_SETUP called in After Effects versus Premiere Pro?

In After Effects, PF_Cmd_GLOBAL_SETUP is called the first time the user applies the effect plugin to a layer. In Premiere Pro, it is called when the application is loading. The timing differs between the two host applications.

*Tags: `aegp`, `pipl`, `premiere`, `reference`*

---

## Can you display dialogs during the Global Setup phase of a plugin?

According to discussions in the community, it should be possible to show dialogs during Global Setup. However, there is some uncertainty about whether Global Setup is called during the AE loading screen when scanning plugins, or only when the effect is first applied. The capability exists, but the exact execution context may affect dialog behavior.

*Tags: `ui`, `pipl`, `aegp`, `debugging`*

---

## How can you access the previous frame's data when processing frame n in an After Effects effect plugin?

You can check out frame n-1 at render time to access the input frame from the previous frame. For more complex cases where you need the output from your plugin's previous frame, you should use sequence data, which is specifically designed for simulation plugins that require frame-by-frame dependencies.

*Tags: `sequence-data`, `render-loop`, `layer-checkout`, `aegp`*

---

## What is the documentation for using sequence data in After Effects effect plugins?

Adobe's official documentation on Global Sequence Frame Data is available at https://ae-plugins.docsforadobe.dev/effect-details/global-sequence-frame-data.html#validating-sequence-data. This documentation provides guidance on sequence data validation and includes simulation plugins as a primary example use case.

*Tags: `sequence-data`, `reference`, `documentation`, `aegp`*

---

## Is it possible to force After Effects to render frames sequentially in a plugin?

No, there is no way to force After Effects to give you frames sequentially. The plugin must be designed to handle out-of-order frame rendering or implement its own logic to manage sequential processing if required.

*Tags: `render-loop`, `threading`, `aegp`*

---

## How should you properly copy pixel data from an OpenCV cv::Mat to an After Effects PF_LayerDef structure?

When copying from cv::Mat to PF_LayerDef, you should not manually allocate layerDef->data yourself—AE manages PF_LayerDef creation and destruction through PF_NewWorld()/PF_DisposeWorld(). The layerDef->data pointer should already be allocated by AE. The pixel data copy itself needs to respect both memory layout and channel ordering: OpenCV uses BGRA order (blue at index 0, green at 1, red at 2, alpha at 3) while AE's PF_Pixel8 struct has ARGB fields. Additionally, ensure you're using the correct row byte calculation and pixel indexing that accounts for the actual rowbytes value, not just width * sizeof(PF_Pixel8), as AE may add padding.

```cpp
PF_Pixel8* pixelData = reinterpret_cast<PF_Pixel8*>(layerDef->data);
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
        PF_Pixel8& aePixel = pixelData[y * (layerDef->rowbytes / sizeof(PF_Pixel8)) + x];
        aePixel.alpha = pixel[3];
        aePixel.red = pixel[2];
        aePixel.green = pixel[1];
        aePixel.blue = pixel[0];
    }
}
```

*Tags: `memory`, `params`, `render-loop`, `aegp`*

---

## Do I need to use PF_NewWorld to modify the output PF_LayerDef in the Render function, or can I directly adjust its data?

You should not modify the layer settings (width, height, rowbytes) directly as they are defined by the host app. Instead of directly assigning mat.data to layerDef->data, you should copy the data from your matrix into the layerDef->data buffer using a loop while respecting layerDef->rowbytes. The proper approach is demonstrated in the skeleton plugin, which uses the Iterate Suite to write into output->data rather than allocating output itself.

```cpp
static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        for (int x = 0; x < layer->width; ++x) {
            cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
            PF_Pixel& aePixel = *sampleIntegral32(*layer, x, y);
            aePixel.alpha = pixel[3];
            aePixel.red = pixel[2];
            aePixel.green = pixel[1];
            aePixel.blue = pixel[0];
        }
    }
}
```

*Tags: `aegp`, `memory`, `layer-checkout`, `output-rect`*

---

## How do I properly sample and write pixels to a PF_LayerDef respecting rowbytes alignment?

Use pointer arithmetic to calculate pixel addresses based on rowbytes. The formula is: (char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)). This ensures proper memory alignment. Reference: https://ae-plugins.docsforadobe.dev/effect-details/tips-tricks.html?highlight=center%20of%20a%20pixel#sampling-pixels-at-x-y

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data +
    (y * def.rowbytes) +
    (x * sizeof(PF_Pixel)));
}
```

*Tags: `aegp`, `memory`, `output-rect`, `reference`*

---

## When do I need to use PF_Cmd_FRAME_SETUP to customize output size in AE plugins?

You only need to use PF_Cmd_FRAME_SETUP (https://ae-plugins.docsforadobe.dev/effect-basics/command-selectors.html) if your effect expands the output buffer size, such as glow or drop shadow effects. For effects that maintain the same output dimensions as input, this step is not necessary.

*Tags: `aegp`, `output-rect`, `reference`*

---

## What is the official Adobe documentation for pixel sampling and layer manipulation in After Effects plugins?

Adobe provides comprehensive documentation at https://ae-plugins.docsforadobe.dev/effect-details/tips-tricks.html covering sampling pixels at X,Y coordinates and related plugin development techniques.

*Tags: `reference`, `aegp`, `pipl`, `debugging`*

---

## Should you use cv::mixChannels with memcpy or the IterateSuite for channel swizzling performance?

You should benchmark both approaches. Using cv::mixChannels followed by memcpy may be slower than using After Effects' IterateSuite (which provides one thread per row) and performing channel swizzling manually during the copy operation from matrix to layer. The IterateSuite approach can provide better performance by combining the channel conversion and copy in a single operation per row.

*Tags: `threading`, `render-loop`, `optimization`, `aegp`*

---

## How do you check out a frame of the current layer after prior effects are applied in After Effects?

Use checkout_layer_pixels from PF_SmartRenderCallbacks to get the frame after prior effects have been applied. This allows you to access the layer state after upstream effects like Exposure have been rendered, but before the current effect processes it. However, this approach may not work in Premiere Pro, requiring alternative solutions for cross-application compatibility.

*Tags: `layer-checkout`, `smartfx`, `render-loop`, `premiere`, `aegp`*

---

## How can you access layer pixels in Premiere Pro during parameter changes if checkout_layer_pixels is not available in Cmd_RENDER?

checkout_layer_pixels is available in PF_SmartRenderCallbacks for After Effects, but for Premiere Pro compatibility, you need to access layer pixels during the user changed parameter callback instead of the render command. The approach involves handling pixel checkout in the parameter change event rather than the standard render path.

*Tags: `premiere`, `aegp`, `params`, `layer-checkout`, `cross-platform`*

---

## How can you implement live drawing and UI interactions like rotoscope in After Effects plugins?

Live drawing and UI interactions such as rotoscope functionality in After Effects plugins are implemented through the PF_CMD_EVENT command, which allows plugins to handle real-time user input and interactive drawing on the canvas.

*Tags: `ui`, `aegp`, `interactive`, `drawing`, `events`*

---

## How do you check out a frame of the current layer after prior effects have been applied in After Effects plugins?

To check out a frame after prior effects are applied (rather than before), you need to use a different approach than PF_CHECKOUT_PARAM, which gets the frame before all prior effects. The conversation indicates this is possible but the specific method was not detailed in the exchange. The questioner was trying to retrieve a frame after the Exposure effect but before the AI Color Match effect in the effect stack.

*Tags: `aegp`, `layer-checkout`, `params`, `render-loop`*

---

## What are the common performance bottlenecks when working with large numbers of layers in After Effects plugins?

Performance issues with large layer counts (800+ layers) typically stem from UI rendering rather than the render engine itself. The bottleneck often comes from repeated After Effects API calls like 'get layer info' and 'get layer sprite', not from Artisan optimization. Additionally, After Effects has poor memory management with large layer counts, making it inefficient to handle thousands of layers.

*Tags: `aegp`, `ui`, `memory`, `performance`, `debugging`*

---

## How do you ensure an After Effects effect uses unaltered input pixels rather than output from a previous render when parameters change?

After Effects automatically provides the correct input layer on each render call. In PF_Cmd_SMART_RENDER, use extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world) to get the right input. In PF_Cmd_RENDER, use params[0]->u.ld to access the correct input layer. You don't need to do anything special—AE handles providing the unaltered input, not the output from your plugin's previous render.

```cpp
// PF_Cmd_SMART_RENDER
extra->cb->checkout_layer_pixels(in_data->effect_ref, 0, &input_world);

// PF_Cmd_RENDER
params[0]->u.ld  // correct input layer
```

*Tags: `smart-render`, `layer-checkout`, `render-loop`, `aegp`*

---

## How do you store a struct in global_data in an After Effects plugin?

You can store a struct in global_data by creating a pointer to it in the global_data structure, then in PF_GlobalSetup allocate it with `globaldata->structPointer = new StructName();`. While you can optionally delete it in PF_GlobalSetdown, this is not strictly necessary since the OS will clear the memory when After Effects closes.

```cpp
// In globaldata definition:
StructName* structPointer;

// In PF_GlobalSetup:
globaldata->structPointer = new StructName();

// Optional in PF_GlobalSetdown:
delete globaldata->structPointer;
```

*Tags: `memory`, `aegp`, `params`*

---

## Can you use smart pointers in After Effects global data objects?

No, you should not use smart pointers in global data objects. Instead, use new/malloc directly since After Effects does funky things with that pointer. You can do delete/free on the pointer during global setup/teardown, but it's optional.

*Tags: `memory`, `aegp`, `plugin-architecture`*

---

## How can you reliably assign a persistent ID to an effect instance that survives effect reordering?

One approach is to hash the layer ID, composition ID, and index of the effect together. However, this has the limitation that if the effect's index changes (e.g., when effects are reordered), the ID must be recalculated. An alternative suggestion from Adobe forums is to store a copy of the ID in a hidden arbitrary parameter and use the UI/ARB thread to update the value from sequence data, though this approach has not been thoroughly tested and may not be intuitive to implement.

*Tags: `arb-data`, `params`, `ui`, `aegp`*

---

## What is the purpose of the changedB parameter in PathOutline, and can paths be modified through the SDK?

The changedB parameter appears to signal whether path data has been modified, though the exact mechanism is unclear. The public SDK does not expose functions for modifying paths—they are read-only. It's theorized that After Effects internally uses an undocumented API for path modification that mirrors the PathOutline disposal API, but this functionality is not available to plugin developers.

*Tags: `aegp`, `params`, `sdk`, `reference`*

---

## Can you procedurally modify mask vertices each frame in an After Effects plugin effect?

You cannot modify project state (including masks) in the render thread of a plugin. However, procedurally changing mask vertices might be possible using expressions. Alternatively, 'baking' the paths is an effective workaround, though not as dynamic. If the functionality is available in the JavaScript API, you could potentially run scripts from within your plugin to achieve this.

*Tags: `aegp`, `render-loop`, `params`, `scripting`*

---

## What is the AEGP_MaskOutlineSuite and how does it differ from procedural mask modification?

The AEGP_MaskOutlineSuite allows you to work with mask outlines, but modifications made through it would be destructive and one-time only (similar to running a script that modifies path vertices). This differs from a regular effect plugin that could procedurally modify path vertices each frame before rasterization.

*Tags: `aegp`, `params`, `scripting`*

---

## Can AEGP_MaskOutlineSuite be used to procedurally modify mask path vertices each frame as a regular effect?

AEGP_MaskOutlineSuite can modify mask outlines, but the distinction is between destructive/one-time modifications (like running a script that permanently changes path vertices) versus procedural, non-destructive modifications applied each frame before rasterization. A regular effect that procedurally modifies path vertices each frame would be the latter approach, allowing the modifications to be recalculated and applied dynamically during rendering rather than permanently altering the mask data.

*Tags: `aegp`, `mask`, `render-loop`, `params`*

---

## What does 'connected instance to instance' mean in the context of After Effects plugins?

It refers to having multiple instances of the plugin on the same or different layers communicating with each other.

*Tags: `aegp`, `params`, `architecture`*

---

## Is it expected to receive thousands of arb dispose calls when shutting down After Effects?

Yes, this is expected behavior. After Effects' allocators don't dispose of resources until available memory is saturated or on thread destruction/exit. You can test this behavior by artificially setting the available memory to a low value like 2GB, which will trigger more destruction calls during runtime.

*Tags: `memory`, `arb-data`, `aegp`, `debugging`*

---

## Is there an open-source reference for After Effects plugin development and layer handling?

The virtualritz/after-effects GitHub repository (https://github.com/virtualritz/after-effects) contains useful reference code for plugin development, including examples of working with PF_LayerDef structures, bit depth detection, and other AEGP-related functionality.

*Tags: `open-source`, `reference`, `aegp`, `layer-checkout`*

---

## What change in After Effects 2025 beta affects AEGP_StartUndoGroup with null parameter?

Starting from After Effects 2025 beta, passing null to AEGP_StartUndoGroup will cause a crash. In previous versions, this did not cause adverse effects, but it is no longer safe to do so.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup(null);
```

*Tags: `aegp`, `debugging`, `crash`, `api-change`, `macos`, `windows`*

---

## What parameter should be passed to AEGP_StartUndoGroup to avoid crashes in After Effects 2025?

As of After Effects 2025 beta, passing null to AEGP_StartUndoGroup() causes a crash, whereas passing an empty string "" works correctly and functions as expected without adding an entry to the Undo stack. Previous versions did not crash with null, so this is a breaking change in AE 2025.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `debugging`, `macos`, `windows`*

---

## How do you handle backward compatibility when adding new parameters with default values to an After Effects plugin?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag. This mechanism ensures that when opening older projects that don't have a new parameter, the plugin will use the specified default value without breaking the appearance of those projects. This is the standard way to handle parameter additions across plugin versions.

*Tags: `params`, `backward-compatibility`, `aegp`*

---

## How does After Effects serialize arbitrary data from plugins to disk?

Arbitrary data is serialized using PF_Cmd_Arbitrary_Callback with the flatten function, similar to sequence data. However, the flatten/unflatten versions are not accessible via AEGP functions. For plugins with stack-variable handles (like "curves"), the handle can be reinterpreted cast as char* and serialized. Plugins with pointers in their structures (like "liquify") cannot be easily serialized this way because deserialized pointer addresses are invalid on reload, causing crashes.

*Tags: `arb-data`, `serialization`, `aegp`, `params`*

---

## What is Maxon Studio and how does it relate to project templates?

Maxon Studio is a template tool that transforms compositions into capsules which can be reused as templates to regenerate projects. It is similar to the AEGP ProjDumper sample but with significantly more features. The tool has been internal since inception, with initial release delayed due to lacking features and UX. It is being prepared for wider studio release with plans to capture arbitrary plugin data into capsules for reuse across 3rd party and Adobe stock plugins.

*Tags: `aegp`, `arb-data`, `tool`, `reference`*

---

## How can an AEGP detect which parameter stream is selected in the effect control window?

There is currently no documented way in the After Effects SDK to determine which stream was selected in the effect control window (ECW). Unlike the timeline where you can use the selection collection to identify selected streams, the ECW does not provide equivalent selection APIs. As a workaround, developers typically place custom commands in the Keyframe Assist menu rather than registering them in the right-click context menu, which does not support command registration.

*Tags: `aegp`, `params`, `ui`, `selection`*

---

## Why is the params pointer always nullptr when handling PF_Cmd_EVENT?

When handling PF_Cmd_EVENT, you need to check the event type via extra->e_type first. You can only modify param values for certain event types, specifically click and keyboard events. The params pointer may not be available in all event contexts, and you may need to use AEGP functions instead to modify parameters depending on the event type.

```cpp
case PF_Cmd_EVENT:
    err = HandleEvent(in_data, out_data, params);
    // Check extra->e_type for event type (click, keyboard, etc.)
    switch(extra->e_type) {
        case PF_Event_CLICK:
        case PF_Event_KEYDOWN:
            // Can modify params here
            break;
    }
```

*Tags: `params`, `aegp`, `ui`, `debugging`*

---

## How can you reliably get the After Effects version in a plugin when in_data->version major and minor don't update between releases?

James Whiffin noted that in_data->version major and minor fields are not being updated reliably between AE 2024 and 2025, returning the same values. This suggests developers should investigate alternative methods for version detection, such as querying the application directly or using other fields in the plugin data structure, though no specific solution was provided in this conversation.

*Tags: `aegp`, `debugging`, `macos`, `windows`*

---

## How can you detect the After Effects version from within a plugin?

You can use ExtendScript with 'app.version' via the AEGP execute script. Alternatively, if your plugin supports both After Effects and Premiere Pro with GPU acceleration, you have access to a Premiere Pro PICA Suite AppInfo that provides the version. The version data is in format like 24.3, and you need to add 2000 to get the actual version number (e.g., 24.3 becomes 2024.3). Some versions may include additional components like 24.3.x which require parsing.

*Tags: `aegp`, `scripting`, `version-detection`, `premiere`, `gpu`*

---

## What causes the 'Cannot run a script while a modal dialog is waiting for response' error when calling scripting from a plugin?

This error occurs when attempting to run scripting from a plugin while a modal dialog is active and waiting for user response. It is a limitation of using scripting from plugins - the scripting engine cannot execute while After Effects has a modal dialog blocking the main thread. This can happen even after checking if scripting is available first, and may occur during parameter setup.

*Tags: `scripting`, `aegp`, `debugging`, `ui`*

---

## How can you determine the After Effects version at runtime in a plugin?

You can use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP as the path type to locate the running binary host, then extract the version from the AE folder name or binary metadata. Alternatively, you can try mapping AEGP_GetDriverSpecVersion to the AE version number, though it's unclear if AEGP version increments in parallel with the host version.

*Tags: `aegp`, `debugging`, `deployment`, `cross-platform`*

---

## How can you set a flag for an entire After Effects session across all plugins without using preferences or a companion AEGP plugin?

You can use dynamic symbol loading with dlopen to access a global variable from one plugin in other plugins. Define a global variable in one of your plugins and dynamically load a function from that plugin using dlopen. Other plugins can then call this function to read the state of the variable. This approach avoids the need for preferences (which persist across sessions) or a separate companion AEGP plugin.

*Tags: `aegp`, `cross-platform`, `ipc`, `plugins`, `session-state`*

---

## What are the recommended AE plugin communication methods for sharing state across multiple plugins?

Alex Bizeau from Maxon recommended two approaches: (1) PlugPlug for easy IPC communication, and (2) AEGP suite with stream params for managing state. For simpler session-specific flags that don't need to persist, dynamic symbol loading with dlopen to access global variables is an even lighter solution than creating a companion AEGP plugin.

*Tags: `aegp`, `ipc`, `plugins`, `communication`, `stream-params`*

---

## How can I access sequence_data in SmartRender when it appears as nullptr?

In After Effects v22 and later, you must use the PF_EffectSequenceDataSuite to read sequence data in SmartRender instead of directly accessing in_data->sequence_data. Use AEFX_SuiteScoper to obtain the suite, then call PF_GetConstSequenceData to retrieve a const handle. This handle can then be locked with PF_LOCK_HANDLE to access the sequence data structure. Note that this method provides read-only access.

```cpp
PF_ConstHandle const_seq;
AEFX_SuiteScoper<PF_EffectSequenceDataSuite1> seqdata_suite =
    AEFX_SuiteScoper<PF_EffectSequenceDataSuite1>(
        in_data,
        kPFEffectSequenceDataSuite,
        kPFEffectSequenceDataSuiteVersion1,
        out_data);
seqdata_suite->PF_GetConstSequenceData(in_data->effect_ref, &const_seq);
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `smartrender`, `sequence-data`, `aegp`, `mfr`*

---

## When should you lock a handle when reading sequence data in After Effects plugins?

When calling GetConstSequenceData(), you should lock the handle before casting and unlock after use, even for read-only access. The lock syntax is: seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq); followed by unlock. This is particularly important in multi-frame rendering (MFR) scenarios with multiple plugin applications, as failure to lock can cause nullptr returns and threading-related crashes. The Adobe documentation does not clearly document this requirement, but it is necessary to prevent data corruption across threads.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
// use seqP
PF_UNLOCK_HANDLE(const_seq);
```

*Tags: `sequence-data`, `mfr`, `memory`, `threading`, `aegp`, `debugging`*

---

## How should dynamic dropdown lists be updated in After Effects plugins to avoid crashes in CC 2025.2?

Use the AEGP StreamSuite to update dropdown parameters instead of directly modifying the param_union. Create a function like setFloatOrBoolOrDropdownParamViaAEGP that uses AEGP_GetNewEffectStreamByIndex and AEGP_SetStreamValue to properly update the parameter value. Call this during UpdateParamsUI. Note: There is a known warning that this approach may cause issues with undo history, though the exact impact is unclear.

```cpp
PF_Err setFloatOrBoolOrDropdownParamViaAEGP(uint16_t i_param_to_change, float val,
                                     AEGP_PluginID my_id, AEGP_EffectRefH *effect_refHP, AEGP_SuiteHandler *suitesP, GlobalData* global_data)
{
    PF_Err err = PF_Err_NONE;
    AEGP_StreamValue streamvalue;
    streamvalue.val.one_d = val;
    AEGP_StreamRefH param_refH = NULL;
    ERR(suitesP->StreamSuite2()->AEGP_GetNewEffectStreamByIndex(my_id, *effect_refHP, i_param_to_change, &param_refH));
    streamvalue.streamH = param_refH;
    ERR(suitesP->StreamSuite2()->AEGP_SetStreamValue(my_id, param_refH, &streamvalue));
    ERR(suitesP->StreamSuite5()->AEGP_DisposeStream(param_refH));
    return err;
}
```

*Tags: `params`, `ui`, `aegp`, `popup`, `windows`*

---

## Is there a reference implementation of dynamic dropdown list handling in After Effects plugins?

Diffusae (an AE plugin project) has a working implementation for updating dropdown parameters like a models list. It uses the AEGP StreamSuite approach with AEGP_GetNewEffectStreamByIndex and AEGP_SetStreamValue called from UpdateParamsUI. This approach is more reliable than directly manipulating param_union.pd.u.namesptr.

*Tags: `params`, `ui`, `aegp`, `open-source`, `reference`*

---

## Why do lock/unlock functions still exist in the After Effects SDK if they weren't ported to 64-bit?

According to Tobias Fleischer, the lock/unlock functions were not ported when the AE codebase moved to 64-bit in CS6 (2011), yet they still exist in the current SDK and sample code calls them. While rowbyte notes these functions are somewhat redundant in a 64-bit address space, they still provide a safe way to dereference handles (which are effectively double pointers in 64-bit) without confusion. The SDK provides a DH macro as an alternative for direct dereferencing.

*Tags: `aegp`, `memory`, `sdk`, `64-bit`, `reference`*

---

## What does the GuidMixInPtr callback in PreRender do?

The extra->cb->GuidMixInPtr callback in PreRender can indicate whether a new render is needed. If this callback suggests no new render is needed, SmartRender may not be called, which is expected behavior.

```cpp
extra->cb->GuidMixInPtr
```

*Tags: `smart-render`, `params`, `aegp`*

---

## Does PF_Interrupt return a value that triggers catch blocks or just an error code?

PF_Interrupt returns an error code (PF_err), not an exception. A non-NULL PF_err does not trigger a catch block in the traditional sense—the error handling depends on how the calling code checks the error return value rather than exception-based control flow.

*Tags: `aegp`, `debugging`, `error-handling`*

---

## What is a command ID tool for After Effects plugin development?

Justin shared a command ID tool that helps developers work with After Effects commands. The tool was highlighted in a recent LinkedIn post by Justin and is useful for plugin development workflows.

*Tags: `aegp`, `tool`, `debugging`, `reference`*

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

## Are there duplicate command IDs in After Effects command ID mappings that developers should be aware of?

Yes, there are duplicate command IDs in After Effects command ID mappings. For example, 'undo' appears as both ID 16 and ID 2371 in JSON files, but they represent different commands—ID 16 is the actual 'undo' command while ID 2371 is 'clear undo'. Developers should avoid treating keys as unique and should verify which ID corresponds to the intended command functionality.

*Tags: `aegp`, `reference`, `debugging`*

---

## Can you set global_outflags in a pipl file to be host-application specific?

The user asked whether pipl files support setting global_outflags with host-app-specific conditions. They wanted to disable the no_params_vary flag for Premiere due to a bug while keeping it enabled in After Effects. No definitive answer was provided in the conversation, so this question remains unresolved.

*Tags: `pipl`, `premiere`, `aegp`, `params`*

---

## How can you set different global_outflags for different host applications in a PIPL plugin?

You can set flags per host in global setup by checking the appl_id in in_data. For example, you can conditionally set out_flags based on whether the host is Premiere or After Effects: if (in_data->appl_id != 'PrMr') { out_data->out_flags = PR_OUT_FLAGS; }. This allows you to remove flags like no_params_vary for Premiere while using them in After Effects.

```cpp
if (in_data->appl_id != 'PrMr') {
  out_data->out_flags = PR_OUT_FLAGS;
}
```

*Tags: `pipl`, `premiere`, `aegp`, `cross-platform`*

---

## Is it possible in PiPL to set global_outflags specific to a host application like Premiere?

Yes, you can set global_outflags with host-specific conditions in PiPL files. One approach is to use GUID-mix with time values to conditionally apply flags like no_params_vary for different host applications, allowing you to remove a flag for Premiere while keeping it enabled in After Effects.

*Tags: `pipl`, `premiere`, `params`, `aegp`*

---

## Are there alternatives to PiPL for describing plugin entrypoints in After Effects plugins?

Yes, PiPL is considered somewhat deprecated. There are alternative ways to describe plugin entrypoints, though the conversation does not specify the exact modern replacement method. Developers should investigate newer plugin descriptor formats beyond the traditional PiPL approach.

*Tags: `pipl`, `aegp`, `deprecated`, `reference`*

---

## What are the new entry points for After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. PIPL is no longer used by Premiere Pro, so developers only need to write PIPL for After Effects. Some investigation suggests it may be possible to remove PIPL entirely by using these new entry points, though the exact mechanism is not yet fully documented.

*Tags: `pipl`, `aegp`, `EffectMain`, `entry-point`, `reference`*

---

## Can multiple effects be registered within a single DLL for After Effects plugins?

Yes, according to Alex Bizeau from maxon, it is possible to call multiple register effect functions in one DLL. This approach could be used to create a bootstrapper similar to what was done for OFX, allowing a single DLL to load multiple effects and reduce code duplication across OFX, After Effects, and AVX plugins.

*Tags: `pipl`, `aegp`, `plugin-architecture`, `deployment`, `build`*

---

## When was the URL-based plugin registration method added to After Effects?

The URL-based plugin registration version was added two versions ago in After Effects 23, according to tlafo. This allows plugins to be registered without needing traditional PIPL resources.

*Tags: `pipl`, `aegp`, `deployment`, `reference`*

---

## Is PiPL still required for After Effects plugins, or can plugins be registered without it?

PiPL is still required for After Effects plugins. While the function PF_Register_effect_ext2 was added two versions ago (in AE 23) to allow registration with URL info, this does not eliminate the need for PiPL/rsrc files. Plugins without PiPL cannot be found by After Effects. The new entry point functionality has advantages, but PiPL is still mandatory, though some of its values can be overwritten in code. However, OFX (which is partly based on the AE SDK) successfully eliminated PiPL entirely, making it more flexible for multi-plugin deployment.

*Tags: `pipl`, `aegp`, `registration`, `plugin-architecture`*

---

## What are the advantages of writing host and plugin format-independent code for multiple platforms?

Writing host and plugin format-independent code frees up significant resources by allowing the processing code to be written once and only occasionally requiring wrapper project updates. This approach has proven effective across multiple platforms including After Effects, Premiere Pro, OFX, Nuke, and Frei0r. For example, when porting the GMIC plugin suite (containing ~2000 plugins) to OFX, all plugins fit in a single file, whereas for AE/Premiere Pro, 2000 tiny individual loader plugins had to be created. The OFX SDK has also benefited from this approach by eliminating detours and inconveniences present in the AE SDK, such as PiPL requirements.

*Tags: `cross-platform`, `ofx`, `plugin-architecture`, `aegp`, `premiere`*

---

## How can you reliably get the current After Effects or Premiere version number (e.g., 2025)?

James Whiffin noted that in_data->version major and minor fields are not being reliably updated between versions like 2024 and 2025, returning the same values. Jonah reported that accessing the version appears to work on macOS After Effects, though behavior on Premiere and Windows still needs verification.

*Tags: `aegp`, `cross-platform`, `macos`, `windows`, `premiere`, `debugging`*

---

## What causes error code 1397908844 when rendering with multiple effects in After Effects?

According to an Adobe Community post (https://community.adobe.com/t5/after-effects-discussions/error-while-rendering-with-multiple-effects-with-code-1397908844-using-pf-newworld-pf-disposeworld/td-p/14285656), this error occurs when a plugin is accidentally double-releasing a suite. The error has been documented since at least 2007 and can also occur in other Adobe applications like Illustrator.

*Tags: `debugging`, `render-loop`, `memory`, `aegp`*

---

## What causes the generic exception handler/catch-all error in Adobe After Effects, and why does it appear more frequently in newer versions?

The generic exception handler/catch-all in the Adobe suites manager is triggered by issues like double freeing or releasing a suite pointer. This error has existed since CS2 or CS3 days. It appears more often in newer AE versions due to Multi-Frame Rendering (MFR) with global suite handles or suite pointers being shared between threads, which increases the likelihood of memory management conflicts.

*Tags: `mfr`, `threading`, `aegp`, `memory`*

---

## What is a modern approach to error handling in After Effects SDK development?

Use std::expected for function returns instead of traditional error codes. This allows for easier error handling and proper chaining with .then() methods. Create type aliases like ExpectedAEGP and ExpectedPF to wrap the expected types for different AE APIs. Additionally, wrap all AE handles in classes with proper construction and destruction in smart pointers, and implement an allocation class manager that clears allocations when exiting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug builds, implement a cache validator to detect extra memory still allocated at endpoints.

```cpp
fn get_thing() -> Result<Thing, Error>;
```

*Tags: `aegp`, `memory`, `debugging`, `error handling`, `smart pointers`, `resource management`*

---

## How can error handling be improved when developing After Effects plugins?

Use std::expected for all function returns instead of traditional error codes. This allows for proper error handling chains using .then() chaining, making error handling much easier and more maintainable. Wrap this in a result struct containing both error codes and results.

*Tags: `aegp`, `error-handling`, `cpp`, `best-practices`*

---

## What best practices should be followed for managing After Effects handles in C++ plugins?

Wrap all AE handles into classes with proper construction and destruction in destructors, and use smart pointers to manage them. Additionally, implement a greater allocation class manager that clears allocations when quitting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug mode, use a cache validator to verify no extra memory remains allocated at endpoints.

*Tags: `aegp`, `memory`, `smart-pointers`, `debugging`, `best-practices`*

---

## How can you check if After Effects is running in headless render engine mode?

Use the AppSuite4 API to check the render engine flag. Call PF_IsRenderEngine() to determine if the renderer is running in headless mode, which is useful for plugins that need to behave differently during background rendering versus interactive use.

```cpp
PF_Boolean bIsRenderEngine = true;
suitesP->AppSuite4()->PF_IsRenderEngine(&bIsRenderEngine);
```

*Tags: `aegp`, `render-loop`, `debugging`, `api`*

---

## Can a plugin expose generated data back to the user for use in expressions?

The question was asked but not answered in the conversation. No response was provided about whether plugins can expose data to expressions or other user-accessible formats.

*Tags: `aegp`, `params`, `scripting`*

---

## Can an After Effects effect plugin compute heavy calculations and update parameter values from the render function?

This appears to be an unsolved use case. The developer wants to perform computationally expensive operations in an effect plugin's render function and have those results update a parameter slider, but current After Effects plugin architecture doesn't provide a straightforward mechanism for plugins to push parameter updates back to the host during rendering. Parameters are typically read during render, not written to.

*Tags: `params`, `render-loop`, `aegp`, `threading`*

---

## How can you apply masks to GPU-rendered output in After Effects when the GPU API only provides masked/cropped buffers?

Eric encountered this limitation while developing the Wrap It 3D camera projection plugin. After Effects' GPU rendering API (checkout_layer_pixels) only returns post-mask-applied, cropped buffers with no GPU equivalent of PF_CHECKOUT_PARAM to access the raw unmasked source. Attempted workarounds included using PF_OutFlag2_REVEALS_ZERO_ALPHA with expanded PreRender requests, but this didn't provide the full unmasked layer to GPU. A manual CPU→GPU upload of the unmasked source defeats GPU acceleration benefits. The core limitation is that AE's native GPU world has no mechanism to provide raw unmasked source data on GPU. As a workaround, users may need to use track mattes in GPU mode, or consider building a custom GPU context (OpenGL, Vulkan, or WebGPU) outside of After Effects' native GPU pipeline to maintain full control over the source data.

*Tags: `gpu`, `layer-checkout`, `masks`, `aegp`, `render-loop`*

---

## Should you use After Effects' native GPU API or create a custom GPU context for complex effects that need full source access?

According to Gabgren and Tim Constantinov's experience, After Effects' native GPU world is less developed with not all features available. For effects requiring full control over source data and GPU acceleration, it's better to create your own GPU context using OpenGL, Vulkan, or WebGPU rather than relying on AE's built-in GPU pipeline. This approach allows you to keep CPU-side operations intact and selectively send only necessary data to your custom GPU context. Tim Constantinov's team experimented with native AE GPU and OpenGL before settling on WebGPU, and Wunk has had positive results with Vulkan.

*Tags: `gpu`, `opengl`, `vulkan`, `webgpu`, `aegp`, `cross-platform`*

---

## How can internally computed positions generated during plugin rendering be exposed to users for use in expressions or parameters?

When a plugin generates its own set of positions as part of the render process (rather than reading from an existing AE path), those values can potentially be exposed to users in two ways: (1) by indexing them in expressions, allowing users to reference computed values dynamically, or (2) by outputting them through a point control parameter, which would make the values available as a controllable/readable parameter in the After Effects UI.

*Tags: `params`, `ui`, `scripting`, `aegp`*

---

## How can a plugin render content that extends beyond the adjustment layer bounds?

When rendering content that exceeds the layer boundaries (such as text on an adjustment layer smaller than the composition), you need to use the output_rect parameter to specify the actual rendering region. By setting output_rect to encompass the full area where your content will be drawn, rather than constraining it to the layer bounds, the plugin can render beyond the adjustment layer's dimensions and display the full content within the composition.

*Tags: `output-rect`, `render-loop`, `aegp`, `ui`*

---

## How can a C++ plugin update a text layer every frame during rendering without locking the project?

This is a fundamental limitation in After Effects: calling AEGP_SetText from the render thread causes the project to lock. Several workarounds exist: (1) Write data to sequence data or compute cache from the render thread, then read and apply it from the UI thread using the aegp_idle hook, which is called multiple times per second; (2) Use your own text rendering engine in the render thread with an external library; (3) Write to disk and have a ScriptUI panel with a timer read it back (though this is unreliable). However, none of these work reliably in MediaEncoder or headless rendering without a GUI. The fundamental issue is that modifying a text layer from render could create infinite loops, which is why AE locks the project.

*Tags: `aegp`, `render-loop`, `threading`, `ui`, `sequence-data`, `compute-cache`*

---

## What are alternative approaches to accessing text layer properties when the standard AEGP text API is broken?

Alex Bizeau shared that the Kerning API in ExtendScript is broken, but works well in Expression Script. A workaround is to use ExtendScript to write an expression to a text layer that writes values to a separate text layer, then read that second layer's value back in ExtendScript. This bypasses the broken AEGP text kerning API. While hacky, this demonstrates using expressions as an intermediary to work around AE API limitations.

*Tags: `aegp`, `scripting`, `ui`, `reference`*

---

## What is a reliable method to pass data from an effect plugin to a text layer?

An AEGP plugin using an idle hook is the best approach. Each frame, push string results into a queue, and periodically process the queue in the idle hook. This requires two binaries (AEGP plugin and FX plugin) communicating through a C interface. Alternatively, you can encode data into dead pixels in your render and use sampleImage expressions to retrieve those pixels and set source text, though this has caveats with render format reliability.

*Tags: `aegp`, `text-layer`, `plugin-communication`, `render-loop`, `idle-hook`*

---

## How can you pass data from an AEGP plugin to an effects plugin in After Effects?

You need to create two separate binaries: an AEGP plugin and an FX plugin. Communicate between them using a C interface that pushes the string into a queue. This allows data to be shared across the two plugin types.

*Tags: `aegp`, `plugin-architecture`, `plugin-communication`, `c-interface`*

---

## Can you pass text output from an effect to a text layer instead of rendering it yourself?

Yes, you can encode text data into pixels and pass it to a text layer. One approach is to use dead pixels (out-of-frame pixels or pixels with alpha of 0) to encode the string data in RGB color values. This allows an effect that produces both pixels and text to delegate text rendering to a native text layer rather than rendering it directly.

*Tags: `ui`, `params`, `aegp`, `output-rect`*

---

## How can a plugin effect communicate implicit dependencies on other layers and effects to After Effects' caching system?

When building multi-layer effects that discover and depend on other layers' properties at runtime, you need to explicitly communicate these dependencies to AE's render dependency tracking system. This involves checking out the layers you depend on and notifying AE of those checkouts so the cache knows to invalidate when those layers change. The renderer effect should enumerate dependent layers, check out their pixels/masks/shapes/transforms, and use the appropriate AEGP calls to register these dependencies so AE's smart render and disk cache systems understand the effect needs to re-render when those layers are modified.

*Tags: `aegp`, `layer-checkout`, `caching`, `smart-render`, `compute-cache`, `memory`*

---

## Is it feasible to build a 2D global illumination plugin using material property metadata effects on individual layers with a separate renderer effect?

Yes, this architecture is theoretically viable with the AE SDK. The pattern involves creating dummy effects on individual layers that only expose material parameters (emissive, diffuse, glass properties, etc.) without rendering output, while a separate renderer effect on an adjustment layer or solid discovers and interprets these metadata effects at render time. The key challenge is properly communicating cross-layer dependencies to AE's caching system so changes to materials trigger re-renders of the expensive global illumination computation while still benefiting from disk caching when unrelated elements change.

*Tags: `aegp`, `params`, `layer-checkout`, `caching`, `smart-render`, `rendering`*

---

## Can I use checkout_layer() during PF_Cmd_SMART_PRE_RENDER to access arbitrary composition layers not exposed as effect parameters?

According to maxon developer Alex Bizeau, you can only grab layer params directly during SMART_PRE_RENDER. However, you can use AEGP_RenderAndCheckoutLayerFrame with any layer IDs, and both sync and async versions exist. For multiple layers, a workaround is to use hidden layer parameters (up to 999) assigned via a button script that updates the comp, so checkout_layer() can access them. You may also handle dynamic layer order/removal by looping through the comp in SmartPreRender and mixing layer order into GuidMixInPtr() for dependency tracking.

*Tags: `smart-render`, `layer-checkout`, `aegp`, `params`, `caching`*

---

## Can I pass a layer receipt GUID to GuidMixInPtr() to register layer dependencies without rendering pixels during SmartPreRender?

You can retrieve a GUID from an AEGP_FrameReceiptH using AEGP_GetReceiptGuid and pass it to extra->cb->GuidMixInPtr(). However, AEGP_RenderAndCheckoutLayerFrame is the expensive call that renders the entire layer, so calling it during SmartPreRender defeats the performance benefit. The suggested approach is to use layer parameters instead, which allow AE to track dependencies without rendering during the SmartPreRender phase.

*Tags: `smart-render`, `aegp`, `caching`, `compute-cache`*

---

## How can you efficiently cache data when scraping parameters from dummy effects on multiple layers?

When implementing a control scheme using layer parameters on the main effect with dummy effects on other layers, call GuidMixInPtr() on the data you scrape with the AEGP suites from the other layers' dummy effects. This ensures After Effects caches everything properly and maintains good performance even when reading all layers one by one.

*Tags: `aegp`, `params`, `caching`, `memory`, `layer-checkout`*

---

## What is a good approach for implementing multi-layer effect controls in After Effects plugins?

A viable user-friendly approach is to use layer parameters on the main effect combined with dummy effects on other layers, rather than requiring users to switch to artisan rendering mode. This integrates well with other effects and provides better usability while maintaining acceptable performance. Testing at quarter resolution can help evaluate performance characteristics.

*Tags: `aegp`, `params`, `ui`, `layer-checkout`, `render-loop`*

---

## How can you keep effect parameters synced between multiple locations when users apply keyframes and expressions?

This is identified as a challenging problem in plugin development. When trying to mirror parameters across multiple UI locations (such as keeping layer effect controls synced with a master adjustment layer control), synchronization becomes difficult if the user applies keyframes or expressions to either location. The conversation suggests using AEGP (After Effects General Plugin) calls or userchangedparam script callbacks as potential approaches, but the questioner notes that constantly mirroring these complex parameter states is not straightforward.

*Tags: `aegp`, `params`, `ui`, `scripting`*

---

## What is an effective architecture for creating interactive material systems with child plugins on layers controlled by a master adjustment layer?

One approach is to use an adjustment layer with a master plugin that loops over child layers and inspects their effects for specifically-named control instances. Rather than creating custom effects, you can use built-in Slider Control and Checkbox Control effects renamed in the UI, with the master plugin scanning for these named controls. The master effect reads their values to drive the material interactions. This keeps material-specific controls directly on the affected layers rather than centralizing all parameters in the master effect, improving UX. A more polished approach would involve creating dummy effects that don't render but provide a cleaner interface for users to specify layer properties without manually adding and renaming slider controls.

*Tags: `params`, `ui`, `aegp`, `mfr`*

---

## How can you synchronize layer data changes in an After Effects plugin using AEGP?

According to Alex Bizeau from maxon, you need to periodically check the project timestamp for changes, then gather your layers and check children for potential updates. This approach is computationally expensive but is noted as the proper synchronization alternative available with AEGP.

*Tags: `aegp`, `layer-checkout`, `caching`, `memory`*

---

## What is the most logical way to prevent an After Effects plugin from being detected in Premiere Pro?

One approach is to return an error in the global setup function, since Premiere Pro runs global setup for all plugins during application startup. This may cause the plugin to be hidden from the effects list, though this behavior was noted as not always working reliably.

*Tags: `premiere`, `deployment`, `pipl`, `aegp`*

---

## How do 3D Camera Tracker and Warp Stabilizer output progress and live update without purging cache?

According to an After Effects developer, these built-in effects use internal APIs that are not available to plugin developers. The effects "cheat" and do not operate within the constraints of a normal AE Effect API plug-in. Developers have attempted workarounds involving refresh kicks, but these have proven unreliable across macOS and Windows. A feature request was made in 2023 to expose this capability to third-party developers, but it remains unavailable through the public API.

*Tags: `aegp`, `smart-render`, `caching`, `render-loop`, `debugging`, `cross-platform`*

---

## Should obsolete CPU architecture fields have been removed from in_data when After Effects went 64-bit only?

Yes, the fields in in_data related to old CPU architecture detection (such as Gestalt values for 68x and PowerPC) should have been removed from the plugin API when Adobe transitioned to 64-bit only support around CS5, rather than keeping legacy fields that are no longer used or meaningful.

*Tags: `aegp`, `deprecated`, `backwards-compatibility`, `macos`, `windows`*

---

## How can you bind C code to scripting languages for After Effects plugins?

There are multiple approaches to create C bindings for scripting languages in AE plugins. For Python, you can use existing bindings (like cairo/pycairo) and pass parameters as text using sprintf formatting (e.g., sprintf("%s=%s\n", paramname, value)), then use the language's C API for bitmap and layer parameter assignment. For Lua, you can use XML-based C-binder code generation tools. Both approaches are similarly tedious without automation or code-generation tools to facilitate the binding process.

```cpp
sprintf("%s=%s\n", paramname, value)
```

*Tags: `scripting`, `plugin`, `build`, `aegp`*

---

## What is a good reference for understanding sequence data in After Effects plugin development?

Tobias Fleischer (reduxFX) has created comprehensive documentation about sequence data that has been very helpful for developers learning this concept. His guide clarifies how sequence data works in After Effects plugins.

*Tags: `sequence-data`, `reference`, `aegp`, `documentation`*

---

## Is PF_LayerDef::rowbytes always guaranteed to be a multiple of 4?

This question was asked but not answered in the conversation. No response or clarification was provided.

*Tags: `memory`, `aegp`, `reference`*

---

## What alignment guarantees does PF_LayerDef::rowbytes provide in After Effects?

PF_LayerDef::rowbytes is guaranteed to always be a multiple of sizeof(PF_Pixel8), which is 4 bytes. 16-byte alignment is not always guaranteed. The rowbytes value is arbitrary and your code should be prepared to deal with any stride. Additionally, a buffer might be a sub-reference to another buffer, so you should not write into the bytes outside your image reference frame into the rowbytes gutter.

*Tags: `memory`, `params`, `aegp`, `output-rect`*

---

## Why is rowbytes alignment important for pointer dereferencing in After Effects plugins?

Dereferencing a pointer that is not correctly aligned for the referenced type results in undefined behavior according to the C specification (6.3.2.3 7 in the C spec https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3047.pdf). If rowbytes was not a multiple of alignof(Pixel Type), you would be dereferencing misaligned pointers everywhere, which is UB. Apple documents this issue at https://developer.apple.com/documentation/xcode/misaligned-pointer.

*Tags: `memory`, `aegp`, `macos`, `debugging`*

---

## How should popup menu items be formatted in After Effects plugin parameter definitions?

When using PF_ADD_POPUPX or add_popup to define popup menu parameters, the separator (pipe character |) must be placed at the end of each line for all choices except the last one. For example: "choice1|" "choice2|" "Choice3". Ensure the parameter definition is properly structured with AEFX_CLR_STRUCT before the popup definition and verify there are no missing parameter definitions between the popup and any subsequent function calls.

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_POPUPX("Color", 3, 2,
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## Can you use the Iterate8Suite in Premiere Pro plugins, and what are the alternatives for multithreading?

The Iterate8Suite has issues in Premiere Pro and doesn't work reliably with multithreading. The suite should work in 8-bit ARGB colorspace (SuiteV2 since 2022), but for BGRA you must use the special BGRA suite. If you need multithreading, implement your own using standard libraries like std::parallel for cross-platform compatibility.

*Tags: `premiere`, `threading`, `aegp`, `multithreading`*

---

## Does PF_ABORT() work the same way in Premiere as it does in After Effects?

Unlike in After Effects where PF_ABORT(in_data) can interrupt rendering and set err to PF_Interrupt_CANCEL, this mechanism does not work reliably in Premiere. Even when frames take significant time to render (over half a second), calling PF_ABORT() does not trigger an abort. In Premiere, you may need to finish rendering every frame that is requested rather than relying on abort functionality to interrupt the render process.

*Tags: `premiere`, `render-loop`, `aegp`, `cross-platform`*

---

## Why does my plugin report 'Not able to acquire AEFX Suite' when playing in Premiere?

This error typically occurs during playback/render in Premiere and may be caused by calling AEFX Suite functions in a shared thread or in code paths that execute in Premiere context. Check if you have conditional logic like `if (app_id != 'PrMr') => call AEFX_suite` where the app_id detection may be returning an incorrect value in Premiere, causing AEFX Suite calls to execute when they shouldn't. Additionally, Premiere can have quirks with certain suites (similar to known colorspace suite issues), so verify your host application detection logic and ensure AEFX Suite calls are not made from Premiere contexts.

*Tags: `premiere`, `aegp`, `threading`, `debugging`*

---

## How can I avoid noise artifacts when After Effects automatically converts 16/32-bit project input to 8-bit for my plugin?

Rather than relying on After Effects' automatic conversion, you should modify your plugin to accept 16 and 32 bits per channel (bpc) inputs and perform the conversion to 8-bit yourself. This approach provides several benefits: users will benefit from having the output remain in 16/32 bpc, you will avoid the warning sign next to your plugin in the UI, and you'll have full control over the conversion process to avoid dithering artifacts introduced by AE's automatic conversion.

*Tags: `params`, `output-rect`, `aegp`, `debugging`*

---

## What does AEGP_GetLayerToWorldXform matrix represent and how should it be used?

AEGP_GetLayerToWorldXform translates XY coordinates from the layer origin to XYZ in composition space. It can be used for simple coordinate conversion between layer-space and composition-space, or to construct a full render view matrix from the camera's point of view.

*Tags: `aegp`, `coordinate-transform`, `layer-space`, `composition-space`, `matrix`, `camera`*

---

## How can I hide an effect parameter from both the effect window and timeline window?

Set the AEGP_DynStreamFlag_HIDDEN flag during the PF_Cmd_UPDATE_PARAMS_UI command. This command is called when a project loads and the effect is shown for the first time, ensuring hidden parameters remain hidden across project loads and new effect applications. Note: There is a known bug in After Effects where owner-drawn portions of hidden parameters may still display space in certain circumstances (e.g., when applying a new effect instance), though the stream itself will be correctly hidden.

*Tags: `params`, `ui`, `aegp`, `debugging`*

---

## What is the correct command to call when hiding deprecated effect parameters?

Use PF_Cmd_UPDATE_PARAMS_UI to set the AEGP_DynStreamFlag_HIDDEN flag on parameters you want to hide. This ensures parameters are hidden when projects load and when effects are first displayed, making it the recommended approach over setting PF_PUI_NO_ECW_UI which only removes parameters from the effect window but not the timeline window.

*Tags: `params`, `ui`, `aegp`*

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

## How should you detect parameter changes that occur from keyframe timeline scrubbing rather than direct user interaction?

Parameter synchronization across user interaction, timeline scrubbing, and rendering presents challenges. For detecting changes during timeline scrubbing, you can use PF_Cmd_UPDATE_PARAMS_UI. However, a more robust approach is to use expressions instead of keyframes on dependent parameters. You can set expressions programmatically during UPDATE_PARAMS_UI using AEGP_SetExpression, which handles all three scenarios (user interaction, timeline scrubbing, and rendering) consistently.

*Tags: `params`, `aegp`, `render-loop`*

---

## How can you retrieve the original parameter value with full precision before downsampling is applied by in_data->downsample_x and downsample_y?

Use AEGP_GetNewStreamValue instead of the plain checkout method to retrieve parameter values. This approach preserves the original precision of the parameter value before any downsampling by in_data->downsample_x and in_data->downsample_y is applied, which is critical for maintaining double precision in 3D point parameters.

*Tags: `params`, `aegp`, `memory`*

---

## Why does AEGP_GetLayerNumMasks return 0 when a mask is clearly set on the layer in After Effects?

The issue may stem from confusion between track mattes and masks. Track mattes are different from masks—a track matte uses the alpha or luma of one layer to affect another layer, while a mask is a vector shape drawn on a layer with the pen tool. If you're trying to use the Mask Suite on a layer that only has a track matte applied, AEGP_GetLayerNumMasks will correctly return 0 because there are no actual masks on that layer. Additionally, verify that the layer handle is valid and check the error value returned by AEGP_GetLayerNumMasks to diagnose the root cause.

```cpp
A_long numMasks = 0;
RECORD_ERROR(suites.MaskSuite6()->AEGP_GetLayerNumMasks(layerHandle, &numMasks));
for (int i = 0; i < numMasks; i++) {
  AEGP_MaskRefH maskHandle;
  RECORD_ERROR(suites.MaskSuite6()->AEGP_GetLayerMaskByIndex(layerHandle, i, &maskHandle));
  auto mask = ExportMask(context, maskHandle);
  masks.push_back(mask);
  RECORD_ERROR(suites.MaskSuite6()->AEGP_DisposeMask(maskHandle));
}
```

*Tags: `aegp`, `mask`, `debugging`, `macos`*

---

## Can you use transform_world multiple times in a loop with the same output buffer as input for subsequent transformations?

No, you cannot use the same buffer as both input and output for transform_world, as it reads and overwrites simultaneously, resulting in corrupted output. Instead, create a temporary buffer and alternate between buffers: transform input to temp, temp to output, output back to temp, repeating as needed. Never overwrite the original input buffer that After Effects caches, as this causes bizarre bugs.

```cpp
// Correct approach for repeated transformations
// 1. create a new buffer "temp"
// 2. transform the input to temp
ERR(in_data->utils->transform_world(in_data->effect_ref, in_data->quality, PF_MF_Alpha_STRAIGHT, in_data->field, input, &composite_mode, NULL, &matrix1, 1L, TRUE, &output->extent_hint, temp));
// 3. transform temp to output
ERR(in_data->utils->transform_world(in_data->effect_ref, in_data->quality, PF_MF_Alpha_STRAIGHT, in_data->field, temp, &composite_mode, NULL, &matrix2, 1L, TRUE, &output->extent_hint, output));
// 4. repeat by transforming output back to temp for next iteration
```

*Tags: `transform_world`, `memory`, `render-loop`, `aegp`, `debugging`*

---

## How can two different plugin types share arbitrary data through their arb parameters?

AE blocks direct expression connections between arb parameters of different plugin types, as it assumes different effects cannot be guaranteed to have the same structure. However, you can read arb values from other effects using AEGP_GetNewStreamValue on the C/C++ side. To reference another plugin instance, use AEGP_GetLayerEffectByIndex with an AEGP_LayerH and effect index. For consistent identification across sessions, store an ID in sequence data and access it via AEGP_EffectCallGeneric, though this has limitations with duplicated effects. Alternatively, rename an invisible parameter, though this won't survive project reloads.

*Tags: `arb-data`, `aegp`, `params`, `cross-plugin`*

---

## What AEGP functions can be used to reference and read arbitrary data from other effect instances?

Use AEGP_GetLayerEffectByIndex to get an effect reference from a layer using AEGP_LayerH and the effect index. Then use AEGP_GetNewStreamValue to read arb values from that effect. For consistent effect identification, store an ID in sequence data and retrieve it using AEGP_EffectCallGeneric.

*Tags: `aegp`, `arb-data`, `sequence-data`*

---

## What is the minimal code structure needed to implement smart render in an After Effects plugin?

The minimal smart render implementation requires a PreRender function that checks out the input layer with the correct channel mask and updates result rectangles, and a SmartRender function that checks out input and output buffers, handles parameter checkout, and processes the image according to bit depth (8, 16, or 32 bit). The code must properly union the result rectangles and always check in parameters even on error conditions.

```cpp
static PF_Err PreRender(PF_InData *in_data, PF_OutData *out_data, PF_PreRenderExtra *extra) {
  PF_Err err = PF_Err_NONE;
  PF_RenderRequest req = extra->input->output_request;
  PF_CheckoutResult in_result;
  req.channel_mask = PF_ChannelMask_ARGB;
  ERR(extra->cb->checkout_layer(in_data->effect_ref, EFFECT_INPUT, EFFECT_INPUT, &req, in_data->current_time, in_data->time_step, in_data->time_scale, &in_result));
  UnionLRect(&in_result.result_rect, &extra->output->result_rect);
  UnionLRect(&in_result.max_result_rect, &extra->output->max_result_rect);
  return err;
}

static PF_Err SmartRender(PF_InData *in_data, PF_OutData *out_data, PF_SmartRenderExtra *extra) {
  PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
  PF_EffectWorld *input_worldP = NULL, *output_worldP = NULL;
  ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, SHIFT_INPUT, &input_worldP));
  ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));
  if(input_worldP && output_worldP) {
    PF_ParamDef paramWhatever;
    AEFX_CLR_STRUCT(paramWhatever);
    ERR(PF_CHECKOUT_PARAM(in_data, EFFECT_PARAM, in_data->current_time, in_data->time_step, in_data->time_scale, &paramWhatever));
    switch (extra->input->bitdepth) {
      case 8: /* process 8 bit */ break;
      case 16: /* process 16 bit */ break;
      case 32: /* process 32 bit */ break;
    }
    ERR2(PF_CHECKIN_PARAM(in_data, &paramWhatever));
  }
  return err;
}
```

*Tags: `smart-render`, `aegp`, `reference`, `params`*

---

## How do you remove a proxy from a composition or footage in After Effects?

There are several ways to remove a proxy in After Effects. You can click the small red square icon next to your composition or footage in the Project window to toggle the proxy on and off. Alternatively, you can remove the proxy altogether from the Interpret Footage window. Another method is to select the Proxy object in the Project Panel, then go to File > Set Proxy... and choose None.

*Tags: `ui`, `reference`, `aegp`*

---

## Does the number of available threads returned by AEGP_GetNumThreads change during an After Effects session?

To the best of knowledge, the number of available threads does not change throughout an AE session. The decision on using PF_Iterations_ONCE_PER_PROCESSOR is an optimization decision that depends on your algorithm. For image processing, PF_Iterations_ONCE_PER_PROCESSOR is rarely optimal because some threads finish their chunk before others and cannot help with remaining work on other threads. Using n iterations instead assures minimal loss of available CPU power for most image processing algorithms.

*Tags: `threading`, `render-loop`, `aegp`, `optimization`*

---

## Why doesn't AEGP_EffectCallGeneric() trigger PF_Cmd_COMPLETELY_GENERAL on Mac when trying to identify a specific effect instance?

According to community expert shachar carmi, AEGP_EffectCallGeneric() has limitations and cannot reliably call from one effect instance to another instance of the same type. The expert suggests that if you need inter-instance communication, you should create a separate AEGP plugin to handle these calls during idle processing, rather than calling directly between effect instances of the same type. As an alternative for identification that doesn't need to survive save/load, you can change a hidden parameter's name to be the identifier and read it via the stream suite.

```cpp
// Instead of calling between same-type effects, use a separate AEGP
// Or use hidden parameter name as identifier via stream suite
AEGP_SetStreamName()
```

*Tags: `aegp`, `params`, `debugging`, `cross-platform`, `macos`, `windows`*

---

## How can an effect instance identify itself given an AEGP_EffectRefH handle?

Only an effect instance has access to its own sequence data handle when AE passes it during calls, and AE may move that memory between calls. For identifying specific instances, experts recommend using a mechanism where each effect instance stores a unique identifier (like a 64-bit ID generated at instantiation). Since direct inter-instance communication via AEGP_EffectCallGeneric() is problematic for same-type effects, alternative approaches include: (1) using a separate AEGP plugin to mediate calls, or (2) for non-persistent identification, changing a hidden parameter's name to encode the identifier and reading it via the stream suite.

*Tags: `aegp`, `sequence-data`, `params`, `arb-data`*

---

## What is the recommended way to store global data like arrays or vectors in After Effects plugins?

While global variables are technically allowed and commonly used, the best practice for After Effects plugins is to allocate memory and store it in a pointer on your global_data handle. This approach is more RAM-friendly and respects AE's memory allocation and prioritization. Avoid using std::vector directly since it uses system malloc/new rather than AE's memory management. If you must use std::vector, write a custom allocator that uses AE's memory allocation functions. Global variables should only be used for constant data that doesn't change once set, and be aware that such memory persists until AE shuts down and cannot be properly released after global_setdown.

*Tags: `memory`, `aegp`, `global-state`*

---

## Can a custom 3D importer be written to behave like the .obj importer with 3D transforms and depth buffer support?

No, this is not possible with standard importer AEGPS (IO and FBIO plugins). Importers only parse a requested frame and pass it to After Effects as an image. True 3D behavior is handled by Artisan plugins, which perform full composition rendering. To achieve 3D import functionality, you would need to write an Artisan plugin, but this requires handling all rendering yourself, not just the 3D import.

*Tags: `aegp`, `3d`, `importer`, `reference`*

---

## How do you get the name of the selected layer in an After Effects plugin?

To get the name of the selected layer, first call AEGP_GetNewCollectionFromCompSelection() to get the current selection. Then traverse the collection using AEGP_GetCollectionNumItems() and AEGP_GetCollectionItemByIndex() to get individual collection items. Each item returns an AEGP_CollectionItemV2 structure. Check the 'type' field to identify if it's a layer. If it is a layer type, extract the AEGP_LayerH from the union field u.layer. Finally, pass this AEGP_LayerH handle to AEGP_GetLayerName() to retrieve the layer name.

```cpp
// Get selected layers
AEGP_CollectionH collectionH;
AEGP_GetNewCollectionFromCompSelection(NULL, &collectionH);

A_long num_items;
AEGP_GetCollectionNumItems(collectionH, &num_items);

for (A_long i = 0; i < num_items; i++) {
    AEGP_CollectionItemV2 item;
    AEGP_GetCollectionItemByIndex(collectionH, i, &item);
    
    if (item.type == AEGP_CollectionItemType_LAYER) {
        AEGP_LayerH layer_h = item.u.layer.layer_handle;
        A_UTF16Char layer_name[AEGP_MAX_LAYER_NAME_LEN];
        AEGP_GetLayerName(layer_h, layer_name);
    }
}
```

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How can I set a parameter value during the paramSetup function?

ParamSetup is called once per session per effect type, not once per instance, so all instances would get the same value. Instead, use sequence data setup (PF_Cmd_SEQUENCE_RESETUP), which is triggered once per instance when created. However, you cannot directly set parameters during sequence setup because the call doesn't have a specific instance associated with it and the params array contains junk data. The correct approach is to set a default value during the PF_ADD_FLOAT_SLIDER call in paramSetup, or set a flag in sequence data indicating a new instance, then check that flag during idle_hook or UPDATE_PARAMS_UI to make the change there.

*Tags: `params`, `sequence-data`, `aegp`*

---

## Can I modify parameters during PF_Cmd_SEQUENCE_RESETUP?

No, you cannot modify parameters during sequence setup calls. Sequence setup typically occurs without a specific instance context, meaning the passed params array contains invalid data and attempting to acquire an AEGP_EffectRef will crash. The workaround is to set a flag in the new sequence data marking it as a brand new instance, then check this flag during idle_hook or UPDATE_PARAMS_UI to apply parameter changes there, remembering to clear the flag afterward.

*Tags: `sequence-data`, `params`, `aegp`*

---

## Is there an API to set custom layer icons for AEGP plugins?

No official API exists for customizing layer icons in AEGP. While there are no standard SDK methods to change layer icons (like the star icon for shape layers or camera icon for camera layers), some developers have explored workarounds by directly manipulating the After Effects interface as an OS window, though this approach is not recommended or officially supported.

*Tags: `aegp`, `ui`, `sdk`*

---

## Why does my map of effect matchnames to install keys return incorrect install keys when applying effects?

The issue was that the map was declared as `std::map<A_char, A_long>`, storing only a single character instead of the full matchname string. The map should be declared with a string type (e.g., `std::map<std::string, A_long>` or `std::map<A_char*, A_long>`) to properly store and look up the complete effect matchname. When accessing the map with `InstallKeys::keys[*myMatchName]`, only the first character of the matchname was being used as the key, causing incorrect lookups.

```cpp
// Incorrect:
std::map<A_char, A_long> keys;
InstallKeys::keys[*nextMatchName] = nextKey;

// Correct:
std::map<std::string, A_long> keys;
InstallKeys::keys[std::string(nextMatchName)] = nextKey;
```

*Tags: `aegp`, `plugin-development`, `debugging`, `c++`*

---

## What is the correct error code to return from an After Effects plugin when a string conversion or operation fails?

Use PF_Err_OUT_OF_MEMORY as the standard error code for failed operations. Avoid PF_Err_INTERNAL_STRUCT_DAMAGED because it tells After Effects that the effect instance is corrupted and After Effects will stop communicating with that instance. PF_Err_OUT_OF_MEMORY is the appropriate way to signal that an operation failed and its results should be ignored.

*Tags: `sdk`, `error-handling`, `aegp`*

---

## What are the memory safety considerations when using AEGP_RenderAndCheckoutLayerFrame_Async?

According to community discussion with Adobe's AE team, there is a known memory release problem when using the async version of renderAndCheckout. As a result, developers have reverted to using the synchronous version instead, rendering one frame at a time on each idle cycle to avoid UI lag while properly managing memory.

*Tags: `aegp`, `memory`, `async`, `render-loop`, `threading`*

---

## How can you safely implement async layer frame rendering in an AEGP plugin?

Create an AsyncManager class that wraps AEGP_RenderAndCheckoutLayerFrame_Async with proper callback handling. The async call is thread-safe as long as only one thread calls it at a time. Use a static callback function that receives the request via refcon, retrieves the world from the frame receipt, and invokes a user-provided callback function. Store the callback as a heap-allocated std::function pointer passed through the refcon parameter, then delete it after use to prevent memory leaks.

```cpp
class AsyncRenderManager {
public:
  void renderAsync(LayerRenderOptionsPtr optionsH, std::function<void(WorldPtr)> callbackF) {
    auto callbackPtr = new std::function<void(WorldPtr)>(callbackF);
    auto ret = std::async(std::launch::async, [optionsH, callbackPtr]() {
      RenderSuite().renderAndCheckoutLayerFrameAsync(optionsH, callback,
        reinterpret_cast<AEGP_AsyncFrameRequestRefcon>(callbackPtr));
    });
  }
  
  static A_Err callback(AEGP_AsyncRequestId request_id, A_Boolean was_canceled, A_Err error,
    AEGP_FrameReceiptH receiptH, AEGP_AsyncFrameRequestRefcon refconP0) {
    auto callbackPtr = reinterpret_cast<std::function<void(WorldPtr)> *>(refconP0);
    if (callbackPtr && *callbackPtr) {
      FrameReceiptPtr ptr = std::make_shared<AEGP_FrameReceiptH>(receiptH);
      auto world = RenderSuite().getReceiptWorld(ptr);
      (*callbackPtr)(world);
      delete callbackPtr;
    }
    return error;
  }
};
```

*Tags: `aegp`, `threading`, `async`, `memory`, `render-loop`*

---

## Can iterate_generic be used to parallelize multiple transform_world calls across threads?

No, iterate_generic should not be used to call transform_world from utility threads. The iterate_generic suite uses utility threads that are separate from the main render threads, and AE API interaction is only safe from main threads. transform_world is internally multithreaded but is not designed to be called from non-main utility threads, which can result in bugs or crashes. Additionally, acquiring suites repeatedly in the iteration callback causes significant overhead. Instead, consider writing a custom transform implementation optimized for multi-threaded execution.

```cpp
suites.Iterate8Suite1()->iterate_generic(
  numThreads,
  &xform_data,
  MT_Xform
);

PF_Err MT_Xform(void* refcon, A_long threadInd, A_long iterNum, A_long iterTotal) {
  // Avoid calling transform_world or other AE suite functions from here
  // Pre-acquire suites in main thread instead
}
```

*Tags: `threading`, `render-loop`, `aegp`, `smartfx`, `performance`*

---

## How can I store custom AEGP plugin data in a comp or project?

There is no official facility for storing AEGP data in a comp or project, but there are several workarounds: (1) Use layer and comp comments fields, which are rarely used by other plugins and not prominently displayed in the UI; (2) Add a null layer to the comp with a clear name like "my stuff, do not delete", lock it, and store data in its comments field while documenting its purpose to users; (3) Create a folder in the project with a clear name and store subfolders named with comp IDs containing the data, minimizing visible clutter.

*Tags: `aegp`, `arb-data`, `project-structure`*

---

## How can I make an AEGP plugin invisible and prevent it from appearing in the After Effects Window toolbar?

To make an AEGP plugin invisible and remove it from the Window toolbar, do not use AEGP_InsertMenuCommand or AEGP_RegisterCommandHook. These functions are only needed to create menu entries. By omitting these calls, the plugin will work in the background without appearing in the UI.

*Tags: `aegp`, `ui`, `deployment`*

---

## How can sequence data be safely shared and synchronized between PF_Cmd_EVENT and SmartPreRender calls?

Sequence data is intentionally separate between the render and UI threads by design, allowing render threads to execute asynchronously without data changes from other threads. To share data between them, use global data (shared across threads but not instance-specific) or store render instance data with an identifier such as comp item ID + layer ID + effect index on layer, then have the UI thread look up the correct data using a mutex. Note that arb/hidden params can only be written from the UI thread; render threads cannot modify project data.

*Tags: `sequence-data`, `threading`, `render-loop`, `aegp`, `arb-data`*

---

## Which identifiers (comp item ID, layer ID, or effect index) remain stable when layers are reordered in After Effects?

Comp item ID doesn't change during a session but may change between sessions or when projects are imported. Layer ID doesn't change when layers are reordered, but can change when layers are copied, cut/pasted, or duplicated to another comp, and may be re-used after deletion. Layer IDs are only unique within a single comp. Effect index changes when the effect moves in the effect stack but not when the layer is reordered; multiple instances of the same effect on one layer can have the same index.

*Tags: `aegp`, `sequence-data`, `arb-data`*

---

## How can I setup an After Effects plugin to asynchronously receive video frames from an external pipe?

You can setup a separate C++ thread (independent of the AE SDK) to handle reading/writing data to a pre-allocated memory buffer. Use a mutex to safely access this memory from both the async pipe thread and AE's render calls. To trigger the async pipe thread to process data promptly, use AEGP_CauseIdleRoutinesToBeCalled(). However, reconciling AE's random frame access model with sequential async pipe operations is challenging—you may need to either stall the render until required frames are available or cache frames to a file and fetch them on demand. For fetching source frames without blocking the UI, use AEGP_CheckoutOrRender_ItemFrame_AsyncManager or AEGP_CheckoutOrRender_LayerFrame_AsyncManager.

*Tags: `threading`, `async`, `aegp`, `memory`, `caching`*

---

## How can I access post-processed shape path vertices (with effects like offset, zig zag, or wiggle applied) using the After Effects SDK?

Unfortunately, After Effects does not provide direct access to post-processed paths through the SDK. You can only read the parameter streams directly and then manually emulate the processed path behavior by implementing the effects yourself.

*Tags: `aegp`, `sdk`, `shape-layers`, `params`*

---

## How can I access decoded video frames in a specific order for face detection in an After Effects plugin?

You can request any project item's image at any time using AEGP_RenderAndCheckoutFrame and AEGP_RenderSuite4. However, After Effects renders frames on-demand and in random order by design. For sequential processing requirements, consider two approaches: (1) fast sequential pre-processing upfront at the cost of user experience, or (2) slower random-access processing for simplicity. Examples like AE's camera tracker and Lockdown perform sequential background processing while keeping the UI responsive. You may also need to use AEIO_InqNextFrameTime() to determine frame order and build a custom data structure to track the sequence.

*Tags: `aegp`, `smart-render`, `caching`, `render-loop`, `sequence-data`, `ui`*

---

## How can I retrieve layer comments from within an After Effects plugin using the C API?

There is no direct C API available to get layer comments. The recommended workaround is to use AEGP_ExecuteScript() to execute a script that retrieves the layer comment, and then retrieve the result back to the C side.

*Tags: `aegp`, `scripting`, `api`*

---

## Why is PF_Cmd_SEQUENCE_FLATTEN sent when an effect is first applied, and why are SEQUENCE_RESETUP and SEQUENCE_SETDOWN called multiple times on different threads?

PF_Cmd_SEQUENCE_FLATTEN is sent when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set during global setup. Flattening allows plugins to serialize complex data structures (containing pointers or other non-copyable elements) into a single copyable piece of memory. Multiple SEQUENCE_RESETUP calls occur because AE's rendering architecture separates UI and rendering threads, each maintaining their own copy of sequence data. The PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA flag allows returning a flattened copy without losing the original, reducing some resetup calls on the UI thread. The PF_OutFlag2_MUTABLE_RENDER_SEQUENCE_DATA_SLOWER flag accommodates plugins requiring separate sequence data per rendering thread. Multiple SEQUENCE_SETDOWN calls correspond to cleanup on different threads.

*Tags: `sequence-data`, `threading`, `aegp`, `mfr`*

---

## How can I store per-effect instance data and ensure unique instance IDs when effects are copied, duplicated, or loaded from disk?

There is no built-in mechanism for per-instance unique IDs. A practical workaround is to store an instance ID in sequence data and use global data to track instances. On SEQUENCE_RESETUP, flag the sequence data as 'requires checking the id', then scan the project (or layer) for conflicting IDs. If duplicates are found, deduce which is the original instance (by tracking where it was last seen) and assign a new ID to the copy. This approach requires careful state management across copy/paste, duplication, and file loading operations. Note that during SEQUENCE_RESETUP, you may receive either flattened or unflattened data, so tag your data structure to track its state.

*Tags: `sequence-data`, `arb-data`, `aegp`, `threading`*

---

## What is the best practice for spawning a window with text input UI from effect parameters?

There are several approaches: (1) Use AEGP_ExecuteScript to launch a JavaScript prompt for easy cross-OS compatibility—it takes text input from the user and you can check the returned value to see if the user hit 'ok' or 'cancel'. (2) Use OS-level windows for more advanced UI, which works on both macOS and Windows but requires more documentation review. (3) Store the text data using either sequence data or an arbitrary (arb) parameter—the arb route is preferred as it allows undo/redo. You can add a custom UI to the arb parameter that both displays the current string value and is clickable, or use a simple button parameter to trigger the window while keeping the arb parameter hidden.

*Tags: `ui`, `params`, `arb-data`, `cross-platform`, `aegp`, `scripting`*

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

## How do I make the idle hook get called more frequently to support higher frame rate precomp playback?

Use the AEGP_CauseIdleRoutinesToBeCalled function instead of trying to manually set max_sleepPL. This will cause idle routines to be called more frequently. Note that on macOS the call rate appears to be capped at around 30 calls per second, while on Windows it can reach 60 calls per second.

*Tags: `aegp`, `idle-hook`, `macos`, `windows`, `cross-platform`*

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

## How can I handle the save event in an After Effects plugin (AEGP)?

You can use the command_hook to detect save events, but it is not 100% reliable as sometimes saves occur without triggering a call. A more robust approach is to implement an idle_hook on your AEGP that checks if the project is dirty. Track the dirty flag state between hook calls—if the project transitions from dirty to clean between successive idle hook invocations, it indicates the user has saved the project.

*Tags: `aegp`, `scripting`, `reference`*

---

## How can I create an After Effects effect that only provides settings without changing the layer visually?

There are several approaches: (1) Use AE's utils->Copy() function to copy the input buffer to the output buffer, which is fast and straightforward. (2) Set the PF_OutFlag_AUDIO_EFFECT_ONLY flag to prevent AE from calling your effect for image rendering—it will only be called for audio, which you must implement by copying input to output. This is faster than copying the image buffer. (3) Try setting the output rect size to 0 or negative during pre-render to make AE skip rendering, though this approach has inconsistent results. The simplest approach is using utils->Copy() to pass through the input unchanged.

```cpp
static PF_Err
Render(	PF_InData		*in_data,
		PF_OutData		*out_data,
		PF_ParamDef		*params[],
		PF_LayerDef		*output)
{
	PF_Err	err	= PF_Err_NONE;
	// Use utils->Copy() to copy input to output
	return err;
}
```

*Tags: `aegp`, `render-loop`, `memory`, `output-rect`*

---

## How can you force rendering from a separate thread in After Effects plugins?

You cannot directly set out_flags from a separate thread. Instead, use the idle hook, which fires 30-50 times per second on the main UI thread. From the idle hook, you can: (1) change an invisible parameter value to force re-render, (2) call command numbers to purge cache, or (3) use AEGP_EffectCallGeneric to have the effect instance act on itself. The idle hook is the recommended approach for syncing work from separate threads back to the main thread.

```cpp
// In idle hook (main thread)
// Change invisible param to trigger re-render
out_data->out_flags |= PF_OutFlag_REFRESH_UI | PF_OutFlag_FORCE_RERENDER;
```

*Tags: `threading`, `render-loop`, `aegp`, `ui`, `debugging`*

---

## How can I prevent users from adding multiple instances of my effect to the same layer?

You can identify your effect uniquely by storing its AEGP_InstalledEffectKey during global setup using AEGP_GetNumInstalledEffects, AEGP_GetNextInstalledEffect, and AEGP_GetEffectMatchName. Then during UPDATE_PARAMS_UI, scan the effect's layer and check each effect's install key using AEGP_GetInstalledKeyFromLayerEffect to detect multiple instances. Once detected, you can alert the user, delete the redundant instance, disable it, or skip rendering for redundant copies by passing input to output. Note that AEGP_DisableCommand cannot remove effects from the menu, and users may still duplicate or copy/paste the effect.

*Tags: `aegp`, `params`, `ui`, `plugin-development`*

---

## What is the issue with storing effect identification data in sequence data across render calls?

Sequence data stored during effect initialization may not persist consistently across different command calls like PF_Cmd_Render. Using AEGP_InstalledEffectKey instead provides a more reliable way to uniquely identify effect instances across the plugin lifecycle without relying on sequence data storage.

*Tags: `sequence-data`, `params`, `aegp`*

---

## Why does setting PF_OutFlag_NON_PARAM_VARY cause memory usage to increase dramatically?

Setting PF_OutFlag_NON_PARAM_VARY or PF_OutFlag_WIDE_TIME_INPUT causes After Effects to call FrameSetup>Render>FrameSetDown for each frame instead of once, which can appear to increase memory usage significantly. However, this is often normal behavior where After Effects is caching frames. The memory will be released when purging AE's memory cache or when the memory is needed for new frame renders. Ensure you implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup, preventing accumulation across multiple frame renders.

*Tags: `memory`, `caching`, `render-loop`, `aegp`*

---

## How should I handle memory allocation when rendering multiple frames with NON_PARAM_VARY flag?

When using PF_OutFlag_NON_PARAM_VARY to render multiple frames, you must implement PF_Cmd_FRAME_SETDOWN to properly free memory allocated during PF_Cmd_FRAME_SETUP. This prevents memory accumulation as FrameSetup and FrameSetDown are called for each frame. Without implementing FRAME_SETDOWN, allocated memory from each frame setup will accumulate rather than being released between frames.

*Tags: `memory`, `render-loop`, `aegp`, `debugging`*

---

## How do you get the duration of a composition layer in the After Effects SDK for C++?

To get layer duration, use AEGP_GetLayerDuration(). To get composition duration, use AEGP_GetItemFromComp() followed by AEGP_GetItemDuration().

*Tags: `aegp`, `sdk`, `c++`, `reference`*

---

## How do I write the generate_key function for the compute cache API without access to in_data?

The generate_key function doesn't have an in_data pointer, but you only need SPBasicSuite to acquire suites, not the full in_data. You have two options: (1) store SPBasicSuite in a global variable (shown in some SDK samples), or (2) use the cleaner approach of passing SPBasicSuite through the AEGP_CCComputeOptionsRefconP struct that all compute cache callbacks receive. If you want to use suite handling macros, create a local copy of the in_data struct and assign the passed SPBasicSuite value to its pica_basicP member. Do not pass in_data itself as it's only valid for the duration of a single AE call.

```cpp
PF_Err AEFX_AcquireSuite( PF_InData *in_data, /* >> */
PF_OutData *out_data, /* >> */
const char *name, /* >> */
int32_t version, /* >> */
const char *error_stringPC0, /* >> */
void **suite) /* << */
{
  PF_Err err = PF_Err_NONE;
  SPBasicSuite *bsuite;
  bsuite = in_data->pica_basicP;
  if (bsuite) {
    (*bsuite->AcquireSuite)((char*)name, version, (const void**)suite);
    if (!*suite) {
      err = PF_Err_BAD_CALLBACK_PARAM;
    }
  }
  return err;
}
```

*Tags: `compute-cache`, `aegp`, `mfr`, `smartfx`*

---

## Is there an official Adobe documentation reference for the compute cache API?

Yes, Adobe's official documentation for the compute cache API is available at https://ae-plugins.docsforadobe.dev/effect-details/compute-cache-api.html, which includes details about AEGP_ComputeCacheCallbacks, the generate_key function, and the AEGP_HashSuite1 for computing hashes.

*Tags: `compute-cache`, `reference`, `aegp`, `documentation`*

---

## How do you convert an A_Time value to a floating-point seconds value in the After Effects SDK?

A_Time is a rational type with two components: value and scale. To convert to seconds as an A_FpLong, divide the value by the scale: `A_FpLong currentTime = currT.value / currT.scale;`. Note that A_Time values are rationals and may not map exactly to floating point, potentially causing off-by-one frame issues, so for precision it's better to work directly with rational time operations.

```cpp
ERR(suites.ItemSuite6()->AEGP_GetItemCurrentTime(itemH, &currT));
A_FpLong currentTime = currT.value / currT.scale;
```

*Tags: `aegp`, `params`, `reference`*

---

## Where can you find documentation and definitions for After Effects SDK data types like A_Time and A_FpLong?

Most AE SDK types are not documented in the PDF reference. The best approach is to examine the SDK header files directly. In your IDE, right-click on any AE type and select 'Go to Definition' to jump to the header where the type is defined. For example, Pf_fp_long is defined as a double. The main documentation resource is https://ae-plugins.docsforadobe.dev/index.html, though it may have gaps.

*Tags: `reference`, `aegp`, `debugging`*

---

## What is example code for converting seconds to an A_Time value in Objective-C?

Here is sample Objective-C code that converts from seconds to A_Time by calculating a timescale based on the composition's framerate: Create a timescale by multiplying framerate by 100, then construct the A_Time with `A_Time time = {seconds * timescale, timescale};`. You can retrieve the framerate using `AEGP_GetCompFramerate()` from the CompSuite.

```cpp
+ (A_Time) timeFromSeconds:(CFTimeInterval)seconds {
  AEGP_CompH comp = [self activeComposition];
  A_FpLong framerate = [self framerateFromComp:comp];
  A_FpLong timescale = framerate * 100;
  A_Time time = {seconds * timescale, timescale};
  return time;
}
```

*Tags: `aegp`, `params`, `macos`*

---

## How can a C++ effect plugin modify transform parameters of other layers in a project?

Use the StreamSuite to affect any parameter (which are internally streams) in the project. The 'project dumper' sample project demonstrates how to access streams. However, since CC2015, you cannot change the project during a render call. To modify project parameters, use UI events (anything other than render and pre-render calls) when you can modify the project as needed.

*Tags: `aegp`, `params`, `render-loop`, `streaming`, `sdk`*

---

## Is there an official sample project demonstrating how to access and manipulate streams in the After Effects SDK?

Yes, Adobe provides the 'project dumper' sample project as part of the After Effects SDK. This sample demonstrates how to access streams and is useful for understanding how to affect parameters in a project using the StreamSuite.

*Tags: `reference`, `open-source`, `sdk`, `aegp`, `streaming`*

---

## Why does a checked-out layer get resized and positioned incorrectly when the effect layer is moved and re-rendered?

The issue occurs because when a layer is moved partially outside the composition, After Effects optimizes rendering by asking plugins to render only the visible rectangle. If you checkout the full layer and copy it to a smaller output buffer, the copy operation resizes the source buffer to match the destination buffer size, causing the layer to be squeezed. The solution is to calculate the offset between the layer position and composition center, then use composite_rect() instead of copy() to properly position the checked-out layer in the output buffer while respecting the output_worldP->origin_x and output_worldP->origin_y values.

```cpp
// Calculate offset
A_long x = (A_long)(effectLayerData.position.x - in_data->width / 2);
A_long y = (A_long)(effectLayerData.position.y - in_data->height / 2);

// Use composite_rect instead of copy, and check origin values
err = worldTransformSuite->composite_rect(in_data->effect_ref,
  in_data->quality,
  PF_MF_Alpha_STRAIGHT,
  in_data->field,
  &checkout_layer_worldP->extent_hint,
  checkout_layer_worldP,
  &compMode,
  NULL,
  output_worldP);
```

*Tags: `layer-checkout`, `aegp`, `render-loop`, `output-rect`*

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

## How can I prevent the timeline properties from opening when setting AEGP_DynStreamFlag_HIDDEN?

According to shachar carmi, there is no way to expose a parameter in the ECW (Effect Control Window) without exposing it in the timeline. As a workaround, try using PF_ParamFlag_SKIP_REVEAL_WHEN_UNHIDDEN flag, which may prevent the parameter from being exposed in both places. Alternatively, PF_UpdateParamUI differs from AEGP_SetDynamicStreamFlag in its behaviors and may produce more suitable results. The 'Supervisor' SDK sample project demonstrates how PF_UpdateParamUI is used.

*Tags: `aegp`, `params`, `ui`, `sdk`*

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

## What is the most performant way to iterate through pixels when converting 8-bit to 16-bit data in After Effects plugins?

Use IterateGeneric to iterate through each line instead of each pixel, which provides much better performance compared to iterating pixel-by-pixel. Additionally, consider using PF_COPY for the actual pixel conversion operation, and refer to methods from the Transformer example that convert PF_Pixel to PF_Pixel16 if needed.

*Tags: `memory`, `caching`, `performance`, `aegp`, `sdk`*

---

## How should you properly store and reuse a frame snapshot across multiple renders in an AE plugin?

You can create a temporary EffectWorld and store it in sequence_data, then use it as the output for your effect across multiple frames. Alternatively, you can access the pixel data directly via effect_worldP->data (the base address of the world's buffer), save it to a file using a library like tinypng, and then import it back as needed. You should avoid copying data directly onto a layer's source buffer because AE may refresh that buffer without the plugin's knowledge, and the buffer may have already been cached elsewhere in the pipeline.

```cpp
effect_worldP->data  // base address of the world's buffer for direct pixel access
```

*Tags: `sequence-data`, `memory`, `render-loop`, `aegp`, `caching`*

---

## Can invisible effect parameters be accessed from expressions in After Effects?

Invisible parameters marked with PF_PUI_INVISIBLE cannot be accessed by name or match name in expressions, but they can be accessed by their index number. This was confirmed through experimentation where accessing by index worked successfully, while name and match name references returned property missing errors.

*Tags: `expressions`, `params`, `scripting`, `aegp`*

---

## How do I retrieve a layer's position after checking it out in an After Effects plugin?

To get a layer's position, use AEGP_GetNewLayerStream with the AEGP_LayerStream_POSITION parameter. The origin_x and origin_y fields from a checked-out layer only give the 0,0 location within the grabbed buffer, not the layer's actual position in the composition. If you need the layer's source dimensions instead, use AEGP_GetNewStreamValue to get the AEGP_LayerIDVal, then AEGP_GetLayerFromLayerID, AEGP_GetLayerSourceItem, and finally AEGP_GetItemDimensions.

```cpp
AEFX_CLR_STRUCT(checkout);
ERR(PF_CHECKOUT_PARAM(in_data,
  PARENT_LAYER_ID,
  in_data->current_time,
  in_data->time_step,
  in_data->time_scale,
  &checkout));
// Use AEGP_GetNewLayerStream with AEGP_LayerStream_POSITION
// origin_x and origin_y only give buffer-relative coordinates, not composition position
```

*Tags: `layer-checkout`, `aegp`, `params`, `reference`*

---

## How do you get the composition width and height in the PF_Cmd_USER_CHANGED_PARAM function?

The in_data->width and height fields denote your effect's layer source item full resolution size (unmasked, unchanged by effects), not the composition size. If you need the actual composition dimensions, you need to get the effect's layer source item and retrieve the project item's dimensions from it. The output pointer will be NULL in PF_Cmd_USER_CHANGED_PARAM, unlike in SmartRender where you can use checkout_output.

*Tags: `params`, `aegp`, `reference`*

---

## Why isn't sequence_data updated when modified in UpdateParameterUI and passed to SequenceResetup?

The issue is that UpdateParameterUI is not the correct place to modify sequence_data with FORCE_RERENDER. According to Adobe documentation, FORCE_RERENDER works during PF_Cmd_USER_CHANGED_PARAM and also in CLICK and DRAG events (if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented). Modifications to sequence_data should be made in PF_Cmd_USER_CHANGED_PARAM instead of UpdateParameterUI to ensure the data is properly synchronized between UI and render threads.

*Tags: `sequence-data`, `aegp`, `threading`, `params`, `ui`*

---

## Why isn't drawing to a newly created Effect World displaying correctly when using transform_world?

When drawing pixel-by-pixel to a newly created Effect World using manual pixel manipulation, ensure you are drawing directly to the output parameter rather than an intermediate world buffer. The issue in this case was a logic error in the broader code context, not the pixel-drawing approach itself. The technique of using sampleIntegral32() to get a pointer to a pixel and then modifying its RGBA values is correct, but verify the world is being used in the right scope and that PF_Fill works as a control test to confirm the world itself is valid.

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

static PF_Err drawPixel32(int x, int y, int size, PF_EffectWorld *output) {
  PF_Err err = PF_Err_NONE;
  int u, v;
  for (u=0; u < size; u++) {
    for (v=0; v < size; v++) {
      PF_Pixel *myPixel = sampleIntegral32(*output, x+u, y+v);
      myPixel->red = 0;
      myPixel->green = 0;
      myPixel->blue = 255;
      myPixel->alpha = 255;
    }
  }
  return err;
}
```

*Tags: `aegp`, `output-rect`, `memory`, `debugging`*

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

## How do you trigger PF_Cmd_UPDATE_PARAMS_UI to dynamically show/hide parameters when user input changes?

To trigger PF_Cmd_UPDATE_PARAMS_UI, you need to set two flags: (1) PF_OutFlag_SEND_UPDATE_PARAMS_UI in global setup, and (2) PF_ParamFlag_SUPERVISE on the parameter you wish to supervise. Without PF_OutFlag_SEND_UPDATE_PARAMS_UI set during global setup, the UPDATE_PARAMS_UI command will not be triggered even if sequence_data changes. Additionally, use PF_UpdateParamUI and the Stream/DynamicStream suites to modify parameter visibility at runtime, and set PF_OutFlag_REFRESH_UI in out_data->out_flags to refresh the UI.

```cpp
// In global setup:
out_data->out_flags = out_data->out_flags | PF_OutFlag_SEND_UPDATE_PARAMS_UI;

// In param setup:
def.flags = PF_ParamFlag_SUPERVISE;

// In PF_Cmd_USER_CHANGED_PARAM:
out_data->out_flags = out_data->out_flags | PF_OutFlag_FORCE_RERENDER | PF_OutFlag_REFRESH_UI;
```

*Tags: `params`, `ui`, `aegp`, `sequence-data`*

---

## When handling parameter changes with PF_Cmd_USER_CHANGED_PARAM, what flag should be set to force re-rendering?

Set the `PF_OutFlag_FORCE_RERENDER` flag in `out_data->out_flags` at the end of your parameter change handler to force After Effects to re-render the layer when the parameter is modified.

```cpp
case PF_Cmd_USER_CHANGED_PARAM:
  err = HandleEvent(in_data, out_data, params, output, reinterpret_cast<const PF_UserChangedParamExtra*>(extra));
  break;

// In HandleEvent:
out_data->out_flags = out_data->out_flags | PF_OutFlag_FORCE_RERENDER;
```

*Tags: `params`, `render-loop`, `aegp`*

---

## Is it necessary to handle PF_Cmd_UPDATE_PARAMS_UI if PF_Cmd_USER_CHANGED_PARAM is already implemented?

PF_Cmd_USER_CHANGED_PARAM is called only on user interactions, while PF_Cmd_UPDATE_PARAMS_UI is called at other times such as when a new instance of an effect is created or when an instance is loaded with a project. If the invocation times of PF_Cmd_UPDATE_PARAMS_UI are not relevant to your plugin's functionality, then there is no need to implement it.

*Tags: `params`, `ui`, `aegp`*

---

## How can a C++ plugin read the time slider position when its own UI window is displayed and plugin parameters haven't changed?

Use the AEGP suite to query the current time directly instead of relying on in_data->current_time, which only updates when EffectMain commands are triggered. Call AEGP_GetActiveItem() to get the active item, then use AEGP_GetItemCurrentTime() to retrieve the current time value. Alternatively, use an idle hook on the plugin side combined with a timer on the window side to synchronize playback between the AE timeline and the custom win32 window.

```cpp
A_long time_ae(PF_InData * in_data)
{
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  A_Time time;
  AEGP_ItemH myitem;
  suites.ItemSuite8()->AEGP_GetActiveItem(&myitem);
  suites.ItemSuite8()->AEGP_GetItemCurrentTime(myitem, &time);
  return time.value;
}
```

*Tags: `aegp`, `ui`, `params`, `windows`, `debugging`*

---

## What causes an access violation error when calling AEGP_GetActiveItem() in a C++ plugin?

An access violation when calling AEGP_GetActiveItem() is most likely caused by an invalid in_data pointer or an invalid pica_basicP passed to the AEGP_SuiteHandler. Ensure that the function is being called during a valid AE command execution where in_data is guaranteed to be valid, and verify that the passed in_data pointer has not been corrupted or used outside its valid scope.

*Tags: `aegp`, `debugging`, `memory`*

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

## If SmartFX is implemented in a 32-bit After Effects plugin, do I still need to implement the Render() event?

If SmartFX is implemented, After Effects does not send RENDER or FRAME_SETUP calls anymore; instead it sends Smart Render and Pre-render calls. However, if you need Premiere Pro compatibility, you should keep both mechanisms implemented since Premiere Pro does not use SmartFX and still calls Render and Frame Setup events.

*Tags: `smartfx`, `render-loop`, `premiere`, `aegp`*

---

## Why does AEGP_GetLayerToWorldXform return zeros when called from multiple threads using Iterate8Suite1?

Most AEGP functions are not thread-safe. AEGP_GetLayerToWorldXform should only be called from the main thread that After Effects called your plug-in with. Calling it from other threads will return zeros or cause internal verification failures. Only a few AEGP functions are documented as thread-safe.

*Tags: `aegp`, `threading`, `memory`, `sdk`*

---

## Why don't slave parameters update when the master parameter is animated and scrubbed through the timeline?

Since CC2015, the render thread is not allowed to modify the project in any way, which prevents parameter updates during rendering. Additionally, with MFR (Multiple Frame Rendering), multiple frames are rendered simultaneously, making it ambiguous what value a parameter should have. If the slave parameter only affects the display UI and doesn't influence the actual render, you can work around this by: (1) creating a custom UI that redraws on time changes using its draw event, or (2) creating an AEGP with an idle_hook that scans the current composition for your effect and modifies it as needed. However, if the slave parameter needs to affect the render or be read by expressions in other effects/layers, this design pattern is not supported by After Effects' workflow.

*Tags: `params`, `render-loop`, `mfr`, `ui`, `aegp`*

---

## Why does a plugin's blur effect keep increasing when changing iteration parameters?

The plugin was writing into the input layer buffer and corrupting it. After Effects provides the original input buffer, not a copy that can be modified. When iterating the blur in a loop and changing the iteration parameter, the corrupted input from the previous operation causes cumulative blur effects. The issue stems from assigning the input layer directly (PF_EffectWorld tmpWrld = params[0]->u.ld) which creates a reference rather than a true copy.

```cpp
// Incorrect - creates reference:
PF_EffectWorld tmpWrld = params[0]->u.ld;

// Correct approach - allocate new buffer:
// AEGP_WorldSuite3->AEGP_New()
// AEGP_FillOutPFEffectWorld()  // creates a PF_EffectWorld wrapper for the allocated AEGP_WorldH
```

*Tags: `params`, `memory`, `render-loop`, `aegp`*

---

## How do you create a proper copy of an input buffer in an After Effects plugin instead of a reference?

Use AEGP_WorldSuite3->AEGP_New() to allocate a new world buffer, then call AEGP_FillOutPFEffectWorld() to create a PF_EffectWorld wrapper for the allocated AEGP_WorldH. This prevents the common mistake of assigning the input layer directly (which only creates a reference) and corrupting the original input data when iterating operations.

*Tags: `memory`, `aegp`, `buffer-allocation`*

---

## How do I make a 3D plugin display the 3D icon and trigger rendering automatically when the camera moves in After Effects?

To enable 3D camera support in your After Effects plugin, you need to set the flags PF_OutFlag2_I_USE_3D_CAMERA and PF_OutFlag2_I_USE_3D_LIGHTS, and use AEGP_GetEffectCameraMatrix to access the camera matrix. However, you must also change the corresponding flags in the PiPL file. Additionally, on Windows, you need to clean and rebuild your project, as the PiPL only regenerates after a clean build.

*Tags: `3d`, `aegp`, `pipl`, `build`, `camera`*

---

## How can I detect when my effect plugin is first applied to a layer in After Effects?

Detect a new effect application by setting a flag in sequence data during the SEQUENCE_SETUP call, which is the only call unique to a new application. At that time, you cannot get a layer handle because AE hasn't associated the instance yet. Wait for the UPDATE_PARAMS_UI call (after the flag is set) to get the layer handle. Then iterate through effects on the layer and use AEGP_GetInstalledKeyFromLayerEffect() to compare InstallKeys with your effect's InstallKey to identify duplicates.

*Tags: `aegp`, `sequence-data`, `params`, `plugin`*

---

## How do you get the original foreground layer pixels when implementing a custom blend mode effect on an adjustment layer?

To get the foreground layer's original pixels, use the layer parameter checkout mechanism. First, set up a layer parameter with "self" as the default. Then use PF_CHECKOUT_PARAM() to checkout the layer at the desired time, and access the fetched pixels at checkout.u.ld. Alternatively, you can use AEGP_GetLayerSourceItem() to get the layer's source item, AEGP_NewFromItem() to create render options, AEGP_RenderAndCheckoutFrame() to get a receipt, and AEGP_GetReceiptWorld() to fetch the pixels. The layer parameter checkout method fetches pre-masks and pre-effects pixels, which is what you need for blend modes.

```cpp
PF_ParamDef checkout;
ERR(PF_CHECKOUT_PARAM( in_data,
index_of_layer_param,
in_data->current_time,
in_data->time_step,
in_data->time_scale,
&checkout));
// Fetched pixels are at checkout.u.ld
```

*Tags: `layer-checkout`, `aegp`, `blend-mode`, `pipl`, `params`*

---

## How can I create a plugin that adds image layers and regenerates them when properties change?

This is possible by combining an AEGP panel plugin with an effect plugin. The AEGP panel (implemented similar to the SDK's 'Panelator' sample) creates and manages layers with an inspector window for user controls. The effect plugin handles rendering and stores configuration data using either hidden parameters (which can be read/written by the AEGP window) or sequence data (a memory chunk that stores arbitrary data without UI representation, undo stack entries, or reset behavior). The effect plugin can regenerate images whenever properties change, with the decisions made in the panel interface driving the effect's rendering behavior.

*Tags: `aegp`, `params`, `sequence-data`, `ui`, `reference`*

---

## What is the recommended sample project for learning how to create a dockable panel in After Effects?

The SDK sample project 'Panelator' demonstrates how to create a dockable panel in After Effects. This sample shows the fundamentals of building a panel interface that can be populated with custom controls and integrated into the AE workspace.

*Tags: `aegp`, `ui`, `reference`, `sdk`*

---

## How can an effect plugin determine what type of layer it is attached to?

Use AEGP_GetLayerObjectType() to determine the layer type. This function returns one of several values: AEGP_ObjectType_NONE (-1), AEGP_ObjectType_AV (includes all pre-AE 5.0 layer types including audio, video sources, and adjustment layers), AEGP_ObjectType_LIGHT, AEGP_ObjectType_CAMERA, AEGP_ObjectType_TEXT, or AEGP_ObjectType_VECTOR. To access the layer handle from within an effect, use AEGP_GetEffectLayer().

```cpp
AEGP_ObjectType_NONE = -1,
AEGP_ObjectType_AV, /* Includes all pre-AE 5.0 layer types (audio or video source, including adjustment layers) */
AEGP_ObjectType_LIGHT,
AEGP_ObjectType_CAMERA,
AEGP_ObjectType_TEXT,
AEGP_ObjectType_VECTOR,
```

*Tags: `aegp`, `params`, `reference`*

---

## What is the correct way to bypass rendering in an After Effects effect plugin?

To bypass rendering in an AE effect plugin, you have a few options: (1) Try setting the output rectangle during pre-render to a 0-sized rect or a rect where the right value is smaller than the left value to tell AE to skip the render, though this approach is not well-documented and may require experimentation. (2) The more reliable and practical approach is to manually copy the input buffer to the output buffer in your render function. According to community experts, the overhead of copying is negligible even when stacking multiple such effects, making this a pragmatic solution that works reliably on both audio and non-audio layers.

*Tags: `render-loop`, `output-rect`, `aegp`, `reference`*

---

## Is there a built-in After Effects flag to prevent an effect from rendering?

The PF_OutFlag_AUDIO_EFFECT_ONLY flag can be used to avoid rendering on non-audio layers, but it does not work for audio layers themselves. For a universal solution that works on all layer types, manually copying the input to the output during the render pass is the recommended approach rather than relying on flags or output rectangle tricks.

*Tags: `params`, `aegp`, `reference`*

---

## How can I set keyframe velocity and influence values using the AEGP API?

The AEGP_SetKeyframeTemporalEase function is broken and does not properly set keyframe velocity and influence values—these settings simply do not stick. You can set keyframe interpolation types (linear, hold, etc.) but not velocity. A workaround is to use AEGP_ExecuteScript() to execute JavaScript that sets the interpolations instead, though this approach is slower when dealing with many properties and layers.

*Tags: `aegp`, `keyframe`, `velocity`, `api`, `workaround`*

---

## Where should AEGP_ExecuteScript be called in a C++ plugin to display a dialog each time the effect is applied?

AEGP_ExecuteScript should not be placed in GlobalSetup or ParamsSetup, as both occur only once per session. Instead, use sequence_setup, which is called once when a new instance is applied to a layer (not when duplicated or copy/pasted), or update_params_ui, which requires tracking to avoid launching the dialog multiple times for the same instance.

*Tags: `aegp`, `scripting`, `ui`, `params`*

---

## How can I create a custom text input field in the Effect Control Window instead of using a JavaScript popup?

Custom UI elements in the ECW or comp window must be handled exclusively via DrawBot. You cannot add OS-level text fields directly to the UI. Instead, you need to either: (1) translate user input through AE's event system onto an offscreen text controller and copy its image buffer, or (2) create a text editor from scratch using the event system and DrawBot. Alternatively, use a JavaScript window for input and display the result as non-interactive text in the ECW that launches the JavaScript window when clicked.

*Tags: `ui`, `drawbot`, `aegp`, `scripting`*

---

## How can I read layer transform data like position, anchor, and scale in an After Effects plugin?

To get a layer's numeric transform values (position, anchor, scale), use AEGP_GetNewLayerStream to get the parameter's stream and AEGP_GetNewStreamValue to retrieve its numeric value. However, if you need all of the layer's transformations in the composition including parenting or camera movement effects that affect the layer's position on screen, use AEGP_GetLayerToWorldXform mixed with the camera matrix. Note that you'll need to get AEGP_PluginID during GlobalSetup(), which is required for many AEGP methods.

*Tags: `aegp`, `params`, `layer-checkout`, `transform`*

---

## How should you handle frame interruption in After Effects plugins to avoid crashes when advancing frames quickly?

When a user interrupts frame processing (by advancing to another frame, clicking buttons, etc.), After Effects does not forcefully kill the rendering thread. Instead, the plugin must call PF_ABORT() during the render call (not pre-render) to check if AE wants processing to stop. If PF_ABORT returns true, you should return a PF_Interrupt_CANCEL error message to tell AE the output buffer should not be used. Crashes during interruption are often caused by re-entrancy issues—if your crash occurs when accessing shared data structures like matrices, ensure thread-safety by using mutexes or allocating structures on the stack rather than globally. Call PF_ABORT periodically during lengthy processing to maintain responsive interactivity while scrubbing or changing parameters.

```cpp
if (err = PF_ABORT(in_dataP)) {
  err = PF_Interrupt_CANCEL;
  return err;  // Must return after setting the error
}
```

*Tags: `threading`, `render-loop`, `memory`, `debugging`, `aegp`*

---

## How can I load a file into a C++ plugin just once instead of every frame in the render function?

Use global_data or sequence_data to store the loaded dictionary outside the render function. global_data stores information common to all instances of an effect throughout an AE session and is not saved with the project, while sequence_data stores information specific to one applied instance of the effect and can be saved with the project. You can put any data you want in these memory handles. Refer to the "supervisor" sample project in the SDK for implementation details on how to use global_data.

*Tags: `memory`, `caching`, `aegp`, `reference`*

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

## How do you copy pixel data from a loaded BMP file into a PF_EffectWorld?

Once you have loaded the BMP pixel data and allocated an effect world, you can access the effect world's pixel data using pointer arithmetic: PF_Pixel *p = world->data + x + y * (world->rowbytes / sizeof(PF_Pixel)). Then copy individual color channels from the BMP data to the effect world pixel, such as p->red = bmpPixel[0], p->green = bmpPixel[1], p->blue = bmpPixel[2], and p->alpha = 255. Be aware that the channel order and bit depth may differ between the BMP format and the effect world format, so value conversion may be required.

```cpp
PF_Pixel *p = world->data + x + y * (world->rowbytes / sizeof(PF_Pixel));
p->red = bmpPixel[0];
p->green = bmpPixel[1];
p->blue = bmpPixel[2];
p->alpha = 255;
```

*Tags: `memory`, `mfr`, `output-rect`, `aegp`*

---

## How do you animate shapes in an After Effects plugin as the user moves the timeline slider?

Use in_data->current_time to get the current frame's time step and in_data->time_step to get the number of steps per second. AE calls the plugin to render a frame with a given time stamp. When rendering a sequence, AE calls the plugin multiple times, each with a corresponding timestamp. You can then compute your shape function shape(t) for each requested frame time, allowing dynamic shapes with varying numbers of paths over time.

```cpp
double x = 100;
double y = 100;
double R = 50;
double phi1 = 0.;
double phi2 = 2*MF_PI;
cairo_t *cr = cairo_create(surface);
cairo_arc(cr, x, y, R, phi1, phi2);
// Use in_data->current_time to get frame time and call shape(t) for animation
```

*Tags: `mfr`, `params`, `render-loop`, `aegp`*

---

## Why doesn't an After Effects plugin receive PF_Event_KEYDOWN when selected in the Effects Controls window?

The plugin needs to have at least one parameter with an ECW (Effects Controls Window) custom UI exposed and untwirled (not collapsed) to receive keyboard events. As a workaround, adding an invisible ARB parameter with custom UI can also fix the issue.

*Tags: `ui`, `debugging`, `params`, `aegp`*

---

## How can you get the index of an effect on a layer in After Effects?

To find the effect index on a layer, get the stream reference of the first effect parameter using EffectSuite4, then use AEGP_GetNewParentStreamRef to get the effect's stream representation, and finally use AEGP_GetStreamIndexInParent to retrieve the effect index on the layer. This is the reverse operation of AEGP_GetLayerEffectByIndex.

*Tags: `aegp`, `params`, `reference`*

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

## Why does changing a button parameter name cause a heap corruption crash when the project closes?

The issue occurs because modifying button parameter names using strcpy on param_union.button_d.u.namesptr outside of PF_Cmd_UPDATE_PARAMS_UI causes heap corruption that manifests during sequence data setdown. The solution is to only modify button text within the PF_Cmd_UPDATE_PARAMS_UI command selector. If modification is needed elsewhere, use strncpy with the string length (without the null terminator +1) to avoid corrupting the buffer, though this may cause garbled text display. Alternatively, consider using AEGP_ExecuteScript() with JavaScript to change button text, which may avoid the issue entirely.

```cpp
// CORRECT METHOD - only call within PF_Cmd_UPDATE_PARAMS_UI
PF_ParamDef param = *params[ID];
strcpy((char*)param.u.button_d.u.namesptr, newLabel);
AEGP_SuiteHandler handler(in_data->pica_basicP);
ERR(handler.ParamUtilsSuite3()->PF_UpdateParamUI(in_data->effect_ref, ID, &param));

// WORKAROUND - outside UPDATE_PARAMS_UI, use strncpy without null terminator
strncpy((char*)param_union.button_d.u.namesptr, label.c_str(), label.length());
```

*Tags: `params`, `ui`, `memory`, `debugging`, `sequence-data`, `aegp`*

---

## How can you refresh the UI to show parameter changes made in PF_Cmd_USER_CHANGED_PARAM?

The PF_OutFlag_REFRESH_UI flag can only be set from specific command selectors (PF_Cmd_EVENT, PF_Cmd_RENDER, PF_Cmd_DO_DIALOG), not from PF_Cmd_USER_CHANGED_PARAM where parameter changes typically need to be reflected. One workaround is to use AEGP_ExecuteScript() to execute JavaScript that triggers a UI update, or restructure the plugin logic to defer UI updates until one of the supported command selectors is called.

*Tags: `params`, `ui`, `render-loop`, `aegp`, `scripting`*

---

## How do I get the layer handle from a selected layer in a layer parameter?

The layer parameter's u.ld field contains a layer ID. Use the AEGP_GetLayerFromLayerID function to convert that layer ID into a layer handle, which you can then use to access the layer's transforms and other properties.

*Tags: `params`, `aegp`, `layer-checkout`*

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

## How can I force an overlay to refresh when ARB parameter data changes without requiring mouse movement?

Use the PF_Event_IDLE event which is automatically sent to composition UIs. In the idle event handler, check system time or other factors to determine if a refresh is needed. If so, set PF_OutFlag_REFRESH_UI or call PF_InvalidateRect(), which will trigger a PF_Event_DRAW call to redraw the overlay and reflect the updated ARB data.

*Tags: `ui`, `arb-data`, `aegp`, `render-loop`*

---

## How can I detect if a layer is a shape layer using the AEGP?

Use the function AEGP_GetLayerObjectType() and check if the result equals AEGP_ObjectType_VECTOR to determine if a layer is a shape layer.

```cpp
AEGP_GetLayerObjectType()
// Returns AEGP_ObjectType_VECTOR for shape layers
```

*Tags: `aegp`, `shape`, `layer-checkout`, `sdk`*

---

## How can you collapse a parameter in the After Effects timeline using the SDK?

An alternative way to collapse a parameter is to use AEGP_SetStreamFlags() to hide and then unhide the parameter, which may collapse the parameter in the timeline as well. This approach can be used in addition to PF_ParamFlag_START_COLLAPSED and PF_ParamFlag_COLLAPSE_TWIRLY, which work for collapsing twirlies in the Effect Control Window (ECW).

*Tags: `sdk`, `params`, `ui`, `aegp`*

---

## How can I group multiple plugin operations into a single undo step in After Effects?

Use StartUndoGroup and EndUndoGroup to bunch multiple operations together into one undo step. This prevents the undo stack from appearing limited when multiple actions are performed.

*Tags: `aegp`, `undo`, `sdk`*

---

## When should I set expressions on parameters in a plugin to ensure proper undo behavior?

While UpdateParameterUI is a common location, consider using USER_CHANGED_PARAM instead if UpdateParameterUI causes undo stack issues. USER_CHANGED_PARAM may provide better control over when expressions are applied and how they integrate with After Effects' undo system.

```cpp
ERR(suites.StreamSuite4()->AEGP_SetExpression(globP->my_id, aegp_get_position_streamH, ("transform.position")));
```

*Tags: `aegp`, `stream`, `expression`, `params`*

---

## How can I move a layer from one composition to another or create a precomposition from a layer using AEGP calls?

You can trigger the precompose command just like any other After Effects command through AEGP. However, if you attempt to execute this during a PF_Cmd_USER_CHANGED_PARAM callback, it will crash because precomposing invalidates the effect from which you're currently operating. To avoid this crash, you need to call the precompose command from an AEGP idle hook instead, which executes the operation outside of the current effect's execution context.

*Tags: `aegp`, `precomp`, `plugin-crash`, `callback`, `idle-hook`*

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

## Can I call a command selector like PF_Cmd_DO_DIALOG in the render function?

No, since CC2015 the render thread cannot perform operations that change the project. Instead, you should use alternative approaches: (1) Use an idle hook to send a message from the render thread to an AEGP, which then creates a window during the idle hook call when it's safe. (2) Use SEQUENCE_SETUP, which is called whenever a new instance of your effect is applied. (3) Use GLOBAL_SETUP, which is called once per session when the effect is first applied. These approaches allow you to open dialogs for user input (like username/password) without trying to trigger commands from the render thread.

*Tags: `render-loop`, `aegp`, `threading`, `ui`*

---

## How do you force re-rendering when masks are deleted or added in an AEGP plugin?

Set the PF_OutFlag2_DEPENDS_ON_UNREFERENCED_MASKS flag in your plugin's global setup. This tells After Effects that your effect depends on mask information even when those masks aren't directly referenced as parameters, ensuring the render refreshes when masks change.

*Tags: `aegp`, `mfr`, `params`, `render-loop`*

---

## What other techniques can help trigger re-renders when external layer properties change in AEGP plugins?

Use GuidMixInPtr to force in external factors, set 'non param vary' in global setup, or consider implementing SmartFX for more sophisticated dependency tracking. Note that GuidMixInPtr may require SmartFX implementation and additional tweaking.

*Tags: `aegp`, `smartfx`, `mfr`, `params`*

---

## Can I disable an After Effects plugin when it loads in Premiere Pro by checking the application ID?

Previously, it was possible to disable a plugin in Premiere Pro by checking in_data->appl_id for 'PrMr' and returning an error code (-1 or similar) at the beginning of the entrypoint function. However, this method no longer works with Premiere Pro CC2018 and later versions. The recommended approach is to install the plugin in version-specific folders rather than relying on application ID checking.

*Tags: `plugin`, `after effects`, `premiere`, `aegp`, `deployment`*

---

## Is it possible to import footage in the background using AEGP functions?

No, it is not possible to import footage in the background. AEGP functions must execute on the main thread only, so background import operations are not supported by the After Effects SDK.

*Tags: `aegp`, `threading`, `import`, `main-thread`*

---

## How can I make PF_Topic rename changes non-undoable without cluttering the undo history?

Start and end an undo group with an empty string as the name (two quotes, not NULL). Wrap your AEGP_SetStreamName() call within this undo group. The operation won't appear in the undo queue, but it remains technically undoable—just not listed in the user-visible undo history.

```cpp
// Start undo group with empty name
A_Err err = AEGP_StartUndoGroup("");
// Perform your name change
AEGP_SetStreamName(cur_streamH, new_name);
// End undo group
AEGP_EndUndoGroup();
```

*Tags: `aegp`, `params`, `undo`, `plugin`, `c++`*

---

## Why does AEGP_ExecuteScript fail with a modal dialog error when called in PF_Cmd_SEQUENCE_RESETUP?

You cannot run AEGP_ExecuteScript while any After Effects dialog is open, including the project loading progress window that appears during sequence setup. The error 'Unable to execute script at line 0. After Effects error: Can not run a script while a modal dialog is waiting for response' indicates a modal dialog is blocking script execution. You need to engineer your process so the script runs at a different time, such as in PF_Cmd_UPDATE_PARAMS_UI instead, where no blocking dialogs are present.

*Tags: `aegp`, `sequence-data`, `scripting`, `debugging`*

---

## Is there a way to access app.settings.saveSetting from C++ plugin code instead of ExtendScript?

The conversation does not provide a direct C++ API equivalent for app.settings.saveSetting. However, the suggested workaround is to defer script execution to a different command handler like PF_Cmd_UPDATE_PARAMS_UI where no modal dialogs are blocking execution, allowing AEGP_ExecuteScript to run successfully.

*Tags: `aegp`, `scripting`, `params`*

---

## How can I copy a layer parameter choice from one PF_Param_LAYER to another programmatically?

To copy a layer parameter from one PF_Param_LAYER to another, get the layer id from the first parameter using AEGP_GetNewStreamValue(), then set that layer_id value to the second parameter using AEGP_SetStreamValue(). The layer_id is accessed via value2P.val.layer_id.

```cpp
AEGP_GetNewStreamValue();//get layer id from first param
value2P.val.layer_id;//this is the gotten layer id.
AEGP_SetStreamValue();//set it to the second param.
```

*Tags: `params`, `aegp`, `layer-checkout`*

---

## Are effect_ref and AEGP_EffectRefH the same thing and interchangeable?

No, effect_ref and AEGP_EffectRefH are not interchangeable. While they may have similar values because both denote RAM addresses, they hold different kinds of data. The effect_ref is used for very few things like GetNewEffectForEffect(), while AEGP_EffectRefH is used for a wide range of AEGP callbacks. Additionally, you cannot rely on the validity of either beyond the scope of a single AE call—after a call ends, AE may change its dataset and invalidate the references. To safely use AEGP functions from an idle hook, store your effect's index on the layer, layer ID, and comp item ID instead, then look up a fresh AEGP_EffectRefH during each idle hook call.

*Tags: `aegp`, `threading`, `params`, `memory`*

---

## How can I safely modify effect parameters from a custom thread in After Effects?

You cannot safely call most AEGP functions from custom threads—doing so will cause catastrophic crashes. The only exception is AEGP_CauseIdleRoutinesToBeCalled(), which is safe to call from any thread. To modify effect parameters, use an idle_hook instead. Store your effect's index on the layer, layer ID, and comp item ID (not AEGP_EffectRefH), then during the idle hook call, look up the comp, layer, and effect to obtain a fresh AEGP_EffectRefH for use in that single call. This ensures references remain valid and all operations occur on AE's main thread.

*Tags: `aegp`, `threading`, `idle-hook`, `params`*

---

## Why doesn't PF_Cmd_RENDER execute when an audio plugin uses PF_OutFlag_I_USE_AUDIO on a precomposed audio-only layer?

When a plugin sets PF_OutFlag_I_USE_AUDIO and the source layer is precomposed with no visual layers, After Effects may not trigger PF_Cmd_RENDER. The solution is to add PF_OutFlag_NON_PARAM_VARY to the global output flags in the plugin's global setup. This flag tells AE that the plugin's output varies based on non-parameter data (in this case, audio data), ensuring render commands are properly triggered even when the precomp contains only audio. Note: This flag is not required when the source layer is not precomposed, as AE automatically handles render triggering in that case.

*Tags: `audio`, `params`, `aegp`, `render-loop`, `layer-checkout`*

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

## How do you capture when a popup parameter selection changes in an After Effects plugin?

To capture popup menu selection changes, the parameter must be created with the PF_ParamFlag_SUPERVISE flag. When such a parameter is changed, the plugin receives a USER_CHANGED_PARAM call. All plugin calls go through the main entry point function (EffectMain), which should implement the selector to handle this notification.

*Tags: `params`, `ui`, `aegp`, `plugin-development`*

---

## Can AEGP_RegisterIdleHook be used in an effect plugin to sync sequence data and UI?

AEGP_RegisterIdleHook is a general call that is not instance-specific, even when placed in an effect plugin. It executes once globally rather than once per effect instance. To access sequence data from individual effect instances, you must use EffectCallGeneric() to find and call each instance separately, allowing each instance to perform its own sequence data comparison. This is the only way to access sequence data from an idle hook in an effect plugin.

*Tags: `aegp`, `sequence-data`, `params`, `effect-plugin`*

---

## What does the max_sleepPL argument in an idle hook function represent?

The A_long *max_sleepPL argument in an idle hook function represents the maximum sleep time in units of 1/60 seconds. It functions as an upper bound ('wait no longer than X * 1/60 seconds') rather than a fixed duration, providing a way to control idle hook execution frequency without requiring longer waits.

*Tags: `aegp`, `render-loop`, `threading`*

---

## How can I find the command ID for Numpad+0 (play preview from start) in After Effects?

Setup a command hook with the 'all commands' flag. Once you've made some preliminary calls, you can press 0 and catch the command ID that is triggered. This method works because the Numpad+0 command doesn't have a corresponding menu item, so app.findMenuCommandId() won't work directly.

*Tags: `scripting`, `aegp`, `ui`, `debugging`*

---

## Is there an API to retrieve the rotation value of individual characters when a text animator is applied?

No, there is no such API available in After Effects. However, you can get the text in its current transformation as shape vertices and attempt to deduce the rotation from the vertex data, though this approach is highly problematic and not reliable.

*Tags: `text`, `api`, `aegp`, `scripting`*

---

## What does the PF_ prefix stand for in the After Effects SDK?

PF stands for 'Plug-in Filter', as opposed to AEGP which stands for 'After Effects General Plug-in'. The PF_ prefix is used throughout the SDK for parameters and classes related to filter plugins.

*Tags: `pipl`, `aegp`, `sdk`, `reference`*

---

## What does the dimensionL parameter represent in AEGP_GetKeyframeTemporalEase, and how should it be used?

The dimensionL parameter refers to the separate X, Y, and Z axes for properties where each axis can be controlled individually with different ease settings. For example, position and scale properties can have different ease settings per axis. The dimensionL ranges from 0 to (temporal_dimensionality - 1), where 0 typically represents a single dimension for properties like sliders, colors, or points that animate as one uniform property.

*Tags: `aegp`, `params`, `keyframe`*

---

## What is an alternative approach to copying and pasting keyframes between parameters in After Effects plugins?

Instead of manually copying keyframe properties through AEGP_GetKeyframeTemporalEase, you can emulate copy and paste functionality using a collection to set selected keyframes and then execute the native After Effects copy/paste commands via AEGP_DoCommand. Alternatively, you can implement the entire process in JavaScript and invoke it from the C API using AEGP_ExecuteScript().

*Tags: `aegp`, `scripting`, `params`*

---

## Does using AEGP_SetSelection force a render, and how can I avoid it?

AEGP_SetSelection should not force a render by itself. If you're experiencing unwanted renders when calling this function, it's likely due to another plugin in the composition that relies on collection state as part of its pre-render GUID, or a bug in your own code (such as missing break statements in case statements). The unexpected render behavior is probably not caused by AEGP_SetSelection directly, but by dependent plugins or logic that responds to the selection change.

*Tags: `aegp`, `render-loop`, `debugging`*

---

## What functionality should be implemented in the Render function versus the 8-bit and 16-bit functions?

Your plugin's entry point function gets invoked with a RENDER or SMART_RENDER call. As long as your plugin responds to that call, you can implement whatever functionality you need. There is no strict separation required—you have flexibility in how you organize your plugin's main functionality across these functions.

*Tags: `render-loop`, `aegp`, `pipl`, `reference`*

---

## Why does AEGP_RenderAndCheckoutLayerFrame cause a hard crash when trying to get layer pixels?

The issue was caused by incorrect pointer declaration. Instead of declaring AEGP_FrameReceiptH *receiptPH (pointer to handle), declare it as AEGP_FrameReceiptH receiptPH (handle itself) and pass it as &receiptPH to the function. Additionally, pass null values for the cancel function parameters instead of uninitialized variables.

```cpp
AEGP_FrameReceiptH receiptPH;
ERR(suites.RenderSuite5()->AEGP_RenderAndCheckoutLayerFrame(optionsH, NULL, NULL, &receiptPH));
ERR(suites.RenderSuite4()->AEGP_GetReceiptWorld(receiptPH, &worldH));
```

*Tags: `aegp`, `layer-checkout`, `debugging`, `memory`*

---

## What is the correct way to change a parameter value during PF_Event_IDLE?

According to Adobe SDK documentation, parameter values can only be modified during PF_Cmd_USER_CHANGED_PARAM and specific PF_Cmd_EVENT types (PF_Event_DO_CLICK, PF_Event_DRAG, PF_Event_KEYDOWN). To change parameters outside these events, use AEGP_SetStreamValue() from the AEGP suite. If the parameter has keyframes, use the keyframe suite instead, though AEGP_SetStreamValue() will still work despite what the documentation states. AEGP_SetStreamValue() is available in all After Effects versions since 2000.

```cpp
params[someindex]->u.fs_d.value = somevalue;
params[someindex]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `sdk`, `idle-event`*

---

## How should an importer plugin interact with an effect plugin to share custom file data?

An importer plugin allows custom file types to be imported into After Effects and placed in compositions. There are two main scenarios for importer-to-effect interaction: (1) The importer parses the file on demand and populates an image buffer, which can then be read by the effect like any other layer source. After Effects can cache the result for faster access on subsequent frame renders. This approach expands usability for custom file types. (2) The effect uses the layer selector only to get the selected layer's ID, retrieves the source project item and file path from it, and then parses the file directly in the effect. This approach allows for data files without graphic interpretation and enables selective storage of data rather than loading all file data into RAM. Both scenarios are valid, and you can combine both approaches—parsing in the importer while also allowing the effect to fetch and parse the file path independently.

*Tags: `importer`, `aegp`, `layer-checkout`, `caching`, `arb-data`*

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

## Why does my After Effects plugin cause memory to balloon to 4-5 GB when changing parameters?

This can happen for two main reasons: (1) After Effects caching images as a user preference, which generally shouldn't be a concern, or (2) your plugin uses PF_NewWorld, which in some AE versions doesn't free memory even when PF_DisposeWorld is called. The memory is only freed when the user manually purges all caches through the menu. To fix this, consider allocating memory through the memory suite instead of PF_NewWorld, and release it after processing. Alternatively, you can point a PF_EffectWorld's 'data' parameter to arbitrarily allocated memory by filling in the other relevant data fields.

*Tags: `memory`, `caching`, `pf_newworld`, `aegp`, `debugging`*

---

## How can a plugin force an undo or redo action in After Effects?

Use AEGP_DoCommand() with command IDs: 16 for undo and 2035 for redo. Note that directly manipulating undo state is generally discouraged as a best practice.

```cpp
AEGP_DoCommand();
// undo = 16
// redo = 2035
```

*Tags: `aegp`, `ui`, `reference`*

---

## How can I duplicate layers along a 3D path with auto-orientation in After Effects?

The After Effects API does not offer built-in tools for rendering 3D transformations. While transform_world() is available for 2D transformations, you need to implement your own 3D transform algorithm to render layer instances along a path. One approach is to calculate the transformation yourself and use transform_world() to fake a 3D look with scale and rotation. For a proof of concept, examine the "Artie" sample project which contains macros for getting XY coordinates of a texture using a 4x4 matrix. If antialiasing is not a concern, this code can provide a quick setup for a 3D transform function.

*Tags: `3d`, `transform`, `path`, `aegp`, `reference`*

---

## How can I efficiently access neighboring pixel values during CCU iteration without using sampleIntegral32 on every pixel?

You can use pointer math to access adjacent pixels directly from the input buffer pointer. If inP is a pointer to the current pixel, you can get the next pixel on the same line with `PF_Pixel *next = inP + 1;`. However, for better performance when needing multiple integer samples of other pixels, use iterate_generic instead of the regular iteration suites, as it allows much more optimization potential than acquiring and releasing the sample suite on every pixel.

```cpp
PF_Pixel *next = inP + 1; // gives you the next pixel on the same line
```

*Tags: `ccu`, `mfr`, `memory`, `performance`, `iteration`, `aegp`*

---

## How can I detect when the user performs undo or redo actions in my After Effects effect plugin so I can update my custom UI?

One effective approach is to save a random number or state identifier with your data, then poll the effect data (stored in a data parameter) once or twice per second to check if the state index differs from what your UI window expects. If there's a mismatch, the data has changed via undo/redo or project load. To optimize performance, you can save the random number to a separate parameter like a slider. When saving data, use StartUndoGroup() and EndUndoGroup() to bundle operations into a single undo entry. This technique works well for effects with custom external UIs that manage internal parameters in a structure, which is then serialized to an AEGP data parameter using AEGP_SetStreamValue().

*Tags: `undo`, `redo`, `ui`, `aegp`, `params`, `debugging`*

---

## How can I pass data from a PopupDialog function to the Render function in an After Effects plugin?

There are two recommended approaches: (1) Store the message in global_data along with an effect identifier (comp itemID, layerID, and effect index), so the render thread can check if its matching identifier has a message waiting. (2) Write the data to an arbitrary parameter using SetStreamValue(), which gets synchronized immediately and reliably between threads. The arb param approach also allows undo/redo functionality for the user.

*Tags: `sequence-data`, `threading`, `aegp`, `arb-data`*

---

## How do I fix the 'could not convert Unicode characters' error when setting an output file path in After Effects plugins?

Use A_UTF16Char instead of A_char for the output path parameter. The correct approach is to cast a wide string literal to A_UTF16Char and pass it to AEGP_SetOutputFilePath. Example: const A_UTF16Char *outPath = reinterpret_cast<const A_UTF16Char *>(L"C:\\whee.mov"); ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(0, 0, (A_char*)outPath));

```cpp
const A_UTF16Char *outPath = reinterpret_cast<const A_UTF16Char *>(L"C:\\whee.mov");
ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(0, 0, (A_char*)outPath));
```

*Tags: `aegp`, `scripting`, `unicode`, `output-rect`*

---

## What happens when a user interrupts rendering by scrubbing the playhead in After Effects?

When the user interrupts rendering by moving the playhead, After Effects does not wait for the render function to complete, and frame setdown is not called in-between. The plugin should call PF_ABORT to check if After Effects wants to quit the current frame's render. If the plugin decides to honor the abort request, it should return PF_Interrupt_CANCEL as the error code from the main entry point.

*Tags: `render-loop`, `aegp`, `interruption`, `mfr`*

---

## What happens if a plugin doesn't check for interrupt with PF_ABORT during rendering?

After Effects does not forcibly kill plugins in mid-call. Instead, AE tells the plugin it wants to abort and waits for the render call to return. The call should return either PF_ErrNONE or PF_Interrupt_CANCEL. If PF_ErrNONE is returned, AE might cache that result. If PF_Interrupt_CANCEL is returned, AE will discard the output buffer content as it has been reported as invalid.

*Tags: `render-loop`, `aegp`, `caching`, `mfr`*

---

## How can you track which version of a plugin was used to save an After Effects project?

There is no dedicated API in the After Effects SDK to retrieve the plugin version that saved a project. The recommended approach is to store the version number in the sequence data of your effect. This allows you to detect version changes when reopening projects and handle updates accordingly.

*Tags: `sequence-data`, `params`, `aegp`, `versioning`*

---

## How do you prevent sequence data from being lost during the flatten call when saving projects?

In After Effects versions before CC2017, you need to purge unwanted sequence data during the flatten call and reconstruct it during resetup. Starting with CC2017 and later, a new API call allows you to keep the original sequence data while delivering a flattened copy, preventing data loss. However, this new call is currently only implemented for UI event calls and not for copy, duplicate, or save operations. Eventually it will replace the older flatten call entirely.

*Tags: `sequence-data`, `aegp`, `params`, `macos`, `windows`*

---

## How should arbitrary data parameters be saved and loaded in After Effects plugins?

When dealing with an ARB (arbitrary) parameter, you should honor the PF_Arbitrary_UNFLATTEN_FUNC and PF_Arbitrary_FLATTEN_FUNC callbacks. The sequence data commands (PF_Cmd_SEQUENCE_FLATTEN and PF_Cmd_SEQUENCE_RESETUP) have nothing to do with ARB parameter persistence. ARB data may get flattened/unflattened during various events such as copy/paste, not only during save/load. If data doesn't reconstruct correctly after honoring these callbacks, the issue is likely with how the flatten/unflatten operations are implemented.

*Tags: `arb-data`, `params`, `aegp`, `memory`*

---

## Why does adding keyframes from a modeless dialog fail in After Effects CC2015 and later?

Since After Effects 2015, there is a known issue with calling AE's suites while a modal loop is running. To work around this, retrieve the information you need from After Effects before running the modal loop, rather than attempting to call AE suites from within the modal dialog.

*Tags: `aegp`, `ui`, `debugging`, `cross-platform`*

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

## Can an After Effects plugin be used as-is in Premiere Pro?

Yes, an AE plugin can be used in Premiere Pro if it meets two criteria: (1) it supports the old 'render' and 'frame setup' calls, not only SmartFX 'smart render' and 'pre-render' calls (since Premiere doesn't support SmartFX), though a plugin can support both in parallel; (2) it doesn't use any AEGP suites, which Premiere doesn't support. Additionally, Premiere Pro requires support for BGRA pixel format in addition to AE's ARGB format.

*Tags: `premiere`, `aegp`, `smartfx`, `cross-platform`*

---

## How can I draw custom curves outside the composition bounds in After Effects?

To draw custom UI elements like curves outside the composition, you need to use PF_Event and Drawbot APIs. Look up "PF_Event" and "Drawbot" in the After Effects SDK guide for implementation details.

*Tags: `ui`, `aegp`, `custom-ui`, `drawing`*

---

## How can I access suite functions from within a pixel iteration function?

Use the refcon (reference context) argument of the iterate() function to pass a struct containing pointers to the suites and any other data you need. Define a struct with your suite references and other variables, pass a pointer to an instance of this struct as the refcon argument to iterate(), and then cast it back to your struct type inside the pixel function to access the suites.

```cpp
typedef struct {
  PF_in_data *in_data;
  AEGP_Suites &suites;
  bool someVariable;
  float someOtherVariable;
} myIterationData;

// In calling function
myIterationData data;
data.suites = &suites;
data.someVariable = FALSE;
suites.Iterate16Suite1()->iterate(
  arg,
  arg,
  arg,
  (void*)&data
);

// In pixel function
myIterationData &data = *(myIterationData*)refcon;
if(data.someVariable) {
  data.suites.whateverSuite()->
}
```

*Tags: `aegp`, `mfr`, `reference`, `render-loop`*

---

## How can I access mask position and tangent information at a particular parameter for a stroke effect plugin?

Use the PF_PathEvalSegLengthDeriv1 function from the After Effects SDK. This function allows you to evaluate path segments and obtain derivative information, which provides the position and tangent data needed for stamping patterns along mask paths without having to write your own Bezier curve rasterizer.

*Tags: `aegp`, `mask`, `params`, `reference`*

---

## Why does data passed via AEGP_IdleRefcon become corrupted when retrieved in the idle hook?

The issue occurs when you save the address of a locally-defined variable in the entry point function and cast it to AEGP_IdleRefcon. Once the function scope ends, that stack memory becomes invalid and may be reused by other code, causing the data to appear corrupted when accessed later in the idle hook. Solution: Use dynamic allocation (new/delete or preferably AE's memory suite) to allocate the data structure on the heap, which remains valid until explicitly freed. Always deallocate the memory in global setdown.

```cpp
// Wrong - local variable on stack:
my_global_data myStruct;
myStruct.my_int = 5;
my_global_dataP globP = &myStruct;  // invalid after function ends

// Correct - heap allocation:
my_global_dataP globP = new my_global_data();
globP->my_int = 5;
AEGP_IdleRefcon idleRefcon = reinterpret_cast<AEGP_IdleRefcon>(globP);
// Later in global setdown:
delete globP;
```

*Tags: `aegp`, `memory`, `refcon`, `debugging`*

---

## Where can I find old but foundational tutorials on After Effects plugin development?

MacTech published a foundational tutorial on After Effects Plugins that, while older, provides essential learning material for plugin developers. URL: http://www.mactech.com/articles/mactech/Vol.15/15.09/AfterEffectsPlugins/index.html

*Tags: `reference`, `tutorial`, `aegp`*

---

## Is there a flag to disable multithreading support for an After Effects plugin?

No, there is no flag to disable multithreading for plugins. All plugins are expected to handle multi-threaded rendering. If your render code is not thread-safe, you could use a mutex at the top of your rendering function to allow only one thread at a time, but this would severely cripple performance. The recommended approach is to make your plugin's render function multi-threadable instead.

*Tags: `threading`, `render-loop`, `aegp`*

---

## How can an After Effects plugin determine the index of the layer it is currently applied to?

Use the PFInterfaceSuite1 to get the effect layer, then use LayerSuite8 to get the layer index. First call AEGP_GetEffectLayer(in_data->effect_ref, &layerH) to get the layer handle, then call AEGP_GetLayerIndex(layerH, layer_indexPL) to retrieve the layer's index.

```cpp
suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
suites.LayerSuite8()->AEGP_GetLayerIndex(layerH, layer_indexPL);
```

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How can a C/C++ plugin propagate internal data to JavaScript in After Effects?

Use the AEGP_ExecuteScript() function from the Utility Suite to execute JavaScript code from the C/C++ plugin side. For simple cases, call suites.UtilitySuite5()->AEGP_ExecuteScript(NULL, "alert('hello world!')", FALSE, NULL, NULL); For more elaborate work where you need to retrieve return values from the script, refer to the detailed example in the Adobe forum thread at https://forums.adobe.com/message/3625857

```cpp
suites.UtilitySuite5()->AEGP_ExecuteScript(NULL, "alert('hello world!')", FALSE, NULL, NULL);
```

*Tags: `aegp`, `scripting`, `c-plugin`, `javascript`, `data-communication`*

---

## How can you exchange data between an effect plugin and an IdleHook callback function?

While an IdleHook can reside in an effect plugin, the callback doesn't have direct access to the effect's in_data structure or parameters. The recommended approach is to allocate and lock a shared piece of RAM during global setup, store it in the global data structure, and pass it as the IdleRefCon to AEGP_RegisterIdleHook. AE will then pass this shared memory to the hook handling function on each idle call. Since this RAM is shared between all instances and threads, use a mutex to prevent race conditions. During idle calls, you can read/write parameter values using AEGP_GetNewStreamValue(), passing the instance reference data through the shared RAM.

*Tags: `aegp`, `memory`, `threading`, `params`*

---

## Why doesn't PF_Event_IDLE run after restoring a project until a control is touched?

PF_Event_IDLE is only called when an effect instance is the sole selected item. When you apply an effect from the menu, it automatically becomes selected alone, so idle events trigger. However, when loading a project, effects are not automatically selected, so idle events don't fire until a control is manually touched. This is expected behavior and cannot be overridden.

*Tags: `aegp`, `ui`, `debugging`, `params`*

---

## How can I handle UI synchronization tasks that need to run even when an effect isn't selected after project restore?

Use AEGP_RegisterIdleHook() to register a general-purpose idle hook that receives regular calls not tied to a specific effect instance. During these calls, you can scan the current comp or entire project for your effect instances using AEGP_GetActiveItem(), AEGP_GetFirstProjItem(), AEGP_GetNextProjItem(), AEGP_GetCompFromItem(), AEGP_GetCompLayerByIndex(), AEGP_GetLayerEffectByIndex(), and AEGP_GetInstalledKeyFromLayerEffect() to identify your effects. You can then programmatically select your effect using AEGP_NewCollection(), AEGP_CollectionPushBack(), and AEGP_SetSelection() so your regular effect process picks it up.

*Tags: `aegp`, `ui`, `params`, `debugging`*

---

## How can you differentiate between an adjustment layer and a precomp in After Effects plugin development?

Use AEGP_GetLayerFlags with the AEGP_LayerFlag_ADJUSTMENT_LAYER flag to determine if the adjustment switch is on or off for a layer. Both adjustment layers and precomps are considered video layers by AEGP_GetLayerObjectType, so this flag is necessary to distinguish them.

```cpp
AEGP_LayerH layerH;
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH));
ERR(suites.LayerSuite5()->AEGP_GetLayerFlags(layerH, &layerFlags));
// Check if AEGP_LayerFlag_ADJUSTMENT_LAYER is set in layerFlags
```

*Tags: `aegp`, `layer-checkout`, `params`*

---

## Why does my After Effects plugin display a white screen after parameter changes, requiring a purge to render correctly?

This is typically a buffer clearing issue. You need to fill the output buffer with empty pixels using PF_FillMatteSuite2()->fill(). Another common cause is accidentally writing data to the input buffer and then using it as an intermediate processing buffer, which causes After Effects to cache the tampered data. Always clear AEGP worlds before using them, as After Effects may reuse the same RAM block from a previous world call, which could contain junk data or previous frame data.

```cpp
PF_FillMatteSuite2()->fill()
```

*Tags: `smart-render`, `memory`, `caching`, `aegp`, `output-rect`*

---

## Should I use AEGP_World for floating-point processing in After Effects plugins?

It is not necessary to use AEGP_World for 32-bit floating-point processing. Even in non-smart plugins that only support 8 and 16-bit, you can create a 32-bit PF_EffectWorld internally for processing in floating point and avoid the complexity and potential issues associated with AEGP worlds.

*Tags: `smart-render`, `aegp`, `memory`*

---

## How do you create a new composition from an imported image or footage item in After Effects?

There is no single command to create a composition directly from footage. Instead, you need to gather the item's data using AEGP_ItemSuite functions like AEGP_GetItemDimensions and AEGP_GetItemDuration to retrieve the necessary information, then use AEGP_CreateComp to create the new composition with that data.

*Tags: `aegp`, `scripting`, `reference`*

---

## How can you keep UI and render threads in sync when automatically applying multiple effects in CC2015+?

When using Sequence Setup to automatically add effects to a layer, the UI and render copies of the project can become out of sync in CC2015+, causing verification failures and missing data effects. The solution is to use an AEGP (After Effects General Plug-in) with an idle hook instead of modifying the layer during Sequence Setup. The effect can set a flag when it's applied, and the AEGP's idle hook will then process the pending operation to add the additional effects. The AEGP can be bundled in the same binary as the effect plugins, but the AEGP's PiPL must be listed first in the bundle.

*Tags: `aegp`, `pipl`, `ui`, `smart-render`, `threading`, `plugin-architecture`*

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

## How can I optimize subpixel sampling performance in After Effects plugins?

To optimize subpixel sampling, acquire the sampling suite once before your rendering loop using AEFX_AcquireSuite, get a direct pointer to it, and release it afterwards. Avoid acquiring and releasing the suite for each sample operation. Alternatively, define the AEGP_SuitesHandler before the loop and pass a pointer to it to the sampling function, though acquiring the suite directly is likely faster since it reduces function call overhead.

*Tags: `mfr`, `aegp`, `performance`, `sampling`, `render-loop`*

---

## What is the difference between PF_SUBPIXEL_SAMPLE and subpixel_sample in After Effects plugins?

PF_SUBPIXEL_SAMPLE is a macro that wraps the in_data->utils->subpixel_sample() function. Both provide the same sampling functionality. The macro approach can be used even though some documentation may mark it as unsupported. The key performance difference is not in which function you use, but in how you manage suite acquisition—acquire the sampling suite once before your loop rather than repeatedly inside it.

*Tags: `mfr`, `aegp`, `sampling`, `params`*

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

## How can you dynamically change effect parameter popup entries in After Effects plugins?

You cannot change the number of popup entries dynamically, but you can change the names of entries. The recommended approach is to create multiple hidden popups with different numbers of options (1, 2, 3, 4, 5, etc.) up to the maximum number of options needed, then show/hide the appropriate popup and populate its strings during ParamChange() or UI() callbacks. This allows the effect to appear to have dynamic popup content while working within SDK constraints.

*Tags: `params`, `ui`, `aegp`, `reference`*

---

## How can an effect plugin listen to changes in another layer and react to them?

An effect plugin can monitor changes in another layer by creating an invisible parameter with an expression that links to a source parameter on the target layer. When the source parameter changes (e.g., text content), the invisible parameter will update and trigger a re-render. To ensure the expression evaluates correctly, use PF_PUI_INVISIBLE to mark the parameter as invisible rather than AEGP_GetDynamicStreamFlags.

*Tags: `params`, `ui`, `render-loop`, `aegp`*

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

## How can you get a layer handle from a composition handle to access composition settings like blending mode when working bottom-up through nested compositions?

There is no direct API to convert a composition handle (AEGP_CompH) to a layer handle (AEGP_LayerH). Instead, you must scan the project top-down: use AEGP_GetItemFromComp() to get the project item for the composition you're searching for, then scan all project items using AEGP_GetNextProjItem(). For each composition found, scan its layers using AEGP_GetLayerSourceItem() and match the layer's source with your target composition's source. This allows you to identify which layer in a parent composition references your nested composition, giving you the required AEGP_LayerH.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How can you get the zoom factor in a UI doClick handler in After Effects?

Use event_extraP->cbs.frame_to_source() to convert frame coordinates to source coordinates. Check the coordinates of (100, 0) and divide the resulting X value by 100 to get the scale factor.

```cpp
// Get zoom factor using frame_to_source
A_Ratio x, y;
event_extraP->cbs.frame_to_source(100, 0, &x, &y);
A_Ratio scale_factor = x / 100.0;
```

*Tags: `ui`, `params`, `aegp`*

---

## How can I safely use threads in an After Effects plugin while calling AEGP suite functions?

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

*Tags: `threading`, `aegp`, `plugin-development`, `mainthread`*

---

## How can I get composition dimensions from within an After Effects plugin?

To get composition dimensions, use the following AEGP API sequence: (1) AEGP_GetEffectLayer() to get the effect layer, (2) AEGP_GetLayerParentComp() to get the parent composition, (3) AEGP_GetItemFromComp() to get the item from the composition, (4) AEGP_GetItemDimensions() to retrieve the dimensions. Note that this returns the comp size at full resolution, so you may need to account for downsample factors depending on your use case.

*Tags: `aegp`, `params`, `output-rect`*

---

## Why do project files crash after adding new parameters to an effect plugin?

Project files crash when parameters are added because After Effects uses DISK_ID values to associate saved parameter data with plugin parameters. If you change the order of parameter definitions in your enum or reassign DISK_IDs, After Effects cannot correctly map the old saved data to the new parameters. For example, if a slider parameter with DISK_ID=3 is replaced by a checkbox parameter in the new version, the saved slider data will be incorrectly applied to the checkbox, causing crashes. The solution is to ensure that existing parameters retain their exact same DISK_ID values from the old version, and only assign new DISK_IDs to newly added parameters. The DISK_ID order has nothing to do with the visual parameter order in the UI.

*Tags: `params`, `arb-data`, `deployment`, `aegp`*

---

## What is the relationship between parameter definition order and DISK_IDs in After Effects plugins?

There are two separate enums in After Effects plugin parameter setup: one that correlates with the order of definitions in param_setup (used to access params by index), and another for DISK_IDs (used as a unique identifier for each parameter). These two do not need to match. A parameter can be in index #5 in the UI but still have DISK_ID=3 if it had DISK_ID=3 in the previous version. The DISK_ID is what matters for backward compatibility and project file loading, not the parameter order or index.

*Tags: `params`, `arb-data`, `aegp`*

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

## Can you keep AEGP handles like AEGP_ItemH, AEGP_CompH, and AEGP_LayerH across multiple plugin calls?

No, you cannot store these handles across plugin calls. AEGP_ItemH, AEGP_CompH, AEGP_LayerH, AEGP_StreamRefH, and AEGP_EffectRefH are only valid during a single call from After Effects to your effect. Once your plugin returns and AE regains control, these handles may become invalid. To track items across calls, store the item ID instead and search for it again on the next call. The same applies for layer handles using layer IDs.

*Tags: `aegp`, `memory`, `reference`*

---

## How can a plugin detect when a RAM preview starts in After Effects?

Use command_hook with command number 2285 to detect when a RAM preview begins. Command 2415 can be used to detect Play (spacebar) events. Note that PF_Cmd_RENDER may not be called for cached frames during RAM preview.

*Tags: `aegp`, `render-loop`, `debugging`, `caching`*

---

## How can a plugin clear the frame cache in After Effects?

Use AEGP_DoCommand() with command number 2372 to clear the frame cache. This allows subsequent playback to trigger PF_Cmd_RENDER for every frame instead of using cached frames.

*Tags: `aegp`, `caching`, `compute-cache`*

---

## How does the A_Time structure work and how do you calculate the value and scale components?

A_Time is represented as a ratio where value/scale equals time in seconds. You can use any whole numbers that express the desired time as a numerator/denominator ratio. For example, {1500, 500} equals 1.5 seconds, and {94208, 25600} equals approximately 3.68 seconds. To set A_Time from a double value like 1.875 seconds, you need to find appropriate whole numbers whose ratio equals that decimal value.

*Tags: `aegp`, `params`, `time`, `keyframe`*

---

## How do you insert keyframes at specific times using AEGP_InsertKeyframe with the correct A_Time values?

Use the AEGP_InsertKeyframe function from the KeyframeSuite3, passing a properly formatted A_Time structure where the ratio of value to scale equals your desired time in seconds. For example, to insert a keyframe at 1.2 seconds, you could use A_Time {12, 10} or {60, 50}. The function signature is: AEGP_InsertKeyframe(position_streamH, AEGP_LTimeMode_LayerTime, &timePT, &new_indexL) where timePT is your A_Time structure.

```cpp
ERR(suites.KeyframeSuite3()->AEGP_InsertKeyframe(position_streamH,
  AEGP_LTimeMode_LayerTime,
  &timePT,
  &new_indexL));
```

*Tags: `aegp`, `keyframe`, `params`, `time`*

---

## How can I draw gradients in an After Effects effect UI using DrawBot?

DrawBot does not have a direct API function for drawing gradients. The recommended approach is to draw the gradient into a buffer using any available means, then convert that buffer to a DrawBot image using the NewImageFromBuffer() function. This allows you to create dynamic gradient fills like color wheels for custom effect panels.

*Tags: `ui`, `drawbot`, `aegp`, `reference`*

---

## How can I retrieve keyframe values and interpolation data from After Effects parameters in a plugin?

Use the AEGP_GetNewStreamValue() function to get parameter values at any composition time. This function allows you to retrieve interpolated values based on keyframes and their temporal interpolation settings (such as Bezier). It is available in several SDK sample projects and will return the calculated value at any given frame, taking into account all keyframe interpolation between start and end frames.

*Tags: `aegp`, `params`, `reference`, `sdk`*

---

## How can an After Effects plugin programmatically create keyframes on its own parameters when a button is pressed?

During the USER_CHANGED_PARAM event for the button parameter, use AEGP_GetNewEffectForEffect to get the effect reference, then use AEGP_GetNewEffectStreamByIndex with the target parameter index to get the parameter stream. Once you have the stream reference, you can use keyframe manipulation functions to set keyframes as needed.

*Tags: `params`, `aegp`, `keyframes`, `sdk`*

---

## What are the best practices for running computationally intensive algorithms during the PF_Cmd_RENDER call in After Effects plugins?

During a render call, you can execute any code except project modifications. For memory allocation, use After Effects' memory and handle suites rather than standard system allocation, otherwise you compete with AE for resources and risk crashes. If you get a hard crash error like 'Crash in progress', it often indicates failed memory allocation. Debug your code during an AE session to identify issues. Ensure your algorithm works correctly and uses AE's memory management APIs.

*Tags: `render-loop`, `memory`, `debugging`, `aegp`*

---

## Why doesn't changing a layer parameter's path trigger a re-render in After Effects?

Layer parameters track changes in the layer's source pixels only, not transforms, masks, or effects. This is intended behavior in After Effects. A layer selector provides the selected layer's source pixels, and only changes to those pixels trigger re-renders. To work around this, you can use an expression that checks the path value and drives a slider parameter to force updates, or alternatively create a light parented to the tracked layer with the I_USE_3D_LIGHTS flag set (though this approach has its own complications).

*Tags: `params`, `layer-checkout`, `render-loop`, `aegp`*

---

## How do you hide a layer parameter in After Effects without breaking parameter evaluation?

To hide a layer parameter while maintaining evaluation, you must set the ui_flags during parameter setup using PF_PUI_NO_ECW_UI. If you hide the parameter later using AEGP_SetStreamFlag(), the parameter will stop evaluating. The correct approach is to hide it at initialization time rather than at runtime.

```cpp
def.ui_flags = PF_PUI_NO_ECW_UI;
```

*Tags: `params`, `ui`, `aegp`, `pipl`*

---

## How can I detect when a user selects a layer in After Effects?

There is no direct layer selection event in the AEGP API. However, you can use the idle_hook to periodically check if the composition's layer collection has changed, which allows you to detect when layer selection has occurred.

*Tags: `aegp`, `ui`, `reference`*

---

## How can you update a layer's mask using the After Effects SDK?

Use the AEGP_MaskSuite and AEGP_MaskOutlineSuite to update layer masks. Note that these suites require copying mask vertices individually. For rare operations or simpler mask copying between layers, consider using AEGP_ExecuteScript() with JavaScript, which may offer a more straightforward solution.

*Tags: `aegp`, `mask`, `sdk`, `layer-checkout`*

---

## What is the simplest way to copy a full mask from one layer to another in After Effects SDK?

While AEGP_MaskSuite and AEGP_MaskOutlineSuite allow mask manipulation, they require vertex-by-vertex copying. For a simpler approach to full mask copying, use AEGP_ExecuteScript() to execute JavaScript code, which typically offers more straightforward mask operations than the C SDK.

*Tags: `aegp`, `mask`, `scripting`, `sdk`*

---

## What is a good approach for creating user option dialogs in AEIO plugins that work on both Windows and macOS?

You can use AEGP_ExecuteScript() to implement user option dialogs in JavaScript, which provides a cross-platform solution that works identically on Windows and macOS. This avoids the need to maintain separate platform-specific dialog code. For implementation details, see the example at https://forums.adobe.com/message/3625857#3625857 and search the Adobe After Effects scripting community forum at https://forums.adobe.com/community/aftereffects_general_discussion/ae_scripting for additional script examples with checkboxes and dropdown lists.

*Tags: `aeio`, `scripting`, `ui`, `cross-platform`, `aegp`*

---

## How can I set time-variant stream values for every frame in an AEGP plugin without creating keyframes at each frame?

According to shachar carmi, expressions are the only way to modify parameter values between keyframes without creating keyframes at each frame. Using AEGP_SetLayerStreamValue() or the Keyframe Suite will automatically create a keyframe at the current time if the stream is already keyframed. On AE 13.5+, only the UI thread can safely change parameter values. The SetStreamValue() function sets the value at the current item time, but if keyframes already exist on that stream, it will create or modify a keyframe at that time.

*Tags: `aegp`, `params`, `keyframe`, `threading`*

---

## What is the recommended approach for animating parameters in AEGP plugins?

Use the Keyframe Suite to animate parameters. This is the proper API for managing keyframe-based animation. Direct SetStreamValue() calls will create keyframes on already-keyframed streams, so the Keyframe Suite should be used for intentional parameter animation.

*Tags: `aegp`, `params`, `keyframe`*

---

## What is the best way to store all pixels from a source layer into an array for processing?

Use the MemorySuite to allocate an int array of any size, then lock the memory handle onto an int pointer which will behave as an array. You can pass a pointer to your custom data as the refcon parameter to the Iterate8Suite, cast it in your iteration function, and fetch your int data according to the x,y coordinates provided by the suite. Alternatively, manually iterate through pixels using getXY() helper function to access PF_Pixel data by calculating the memory offset using rowbytes.

```cpp
static PF_Pixel *getXY(PF_EffectWorld &def, int x, int y){
  return (PF_Pixel*)((char*)def.data + (y * def.rowbytes) + (x * sizeof(PF_Pixel)));
}

// In Render function:
for(int i = 0; i < tInfo->width; i++){
  for(int j = 0; j < tInfo->height; j++){
    PF_Pixel currentPixel = *getXY(*tInfo->input, i, j);
    int alpha = currentPixel.alpha;
    int red = currentPixel.red;
    int green = currentPixel.green;
    int blue = currentPixel.blue;
  }
}
```

*Tags: `memory`, `aegp`, `params`, `reference`*

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

## How can I allow users to abort a long-running keyframe baking process triggered by PF_Cmd_USER_CHANGED_PARAM?

Instead of blocking the entire baking process in a single PF_Cmd_USER_CHANGED_PARAM call, implement incremental processing: get a clock reading at the beginning and process keyframe changes in ~1 second chunks. Let the AEGP handle continuation during idle time. This allows users to interrupt for interaction, and processing resumes immediately after. This approach enables user interaction while the baking process continues in segments rather than blocking on a single long operation.

*Tags: `params`, `aegp`, `ui`, `threading`*

---

## Where can I find the source code for AEGP_GetNewStreamValue() and how to work with streams in After Effects plugins?

The After Effects SDK documentation and sample projects are available from Adobe's developer portal. The SDK Guide PDF and sample projects like ProjDumper, Streamie, and Mangler provide good examples of how streams are implemented. You can download the latest SDKs from http://www.adobe.com/devnet/aftereffects.html, and each SDK comes with an SDK guide in PDF format. For CS6 specifically, the download links are available at http://www.adobe.com/devnet/aftereffects/sdk/cs6_eula.html

*Tags: `aegp`, `reference`, `sdk`, `streams`*

---

## What are good sample projects to study for implementing stream data in After Effects plugins?

The ProjDumper, Streamie, and Mangler sample projects included in the After Effects SDK are recommended as good places to see streams implemented. These projects are included in the SDK download packages available from Adobe's developer portal.

*Tags: `aegp`, `reference`, `open-source`, `sample`*

---

## How can I resize the output buffer in an After Effects plugin and determine the correct size before rendering?

You cannot access the input image during Frame_Setup, but you have two options: (1) Save necessary data during the render call and trigger a re-render using PF_ForceRerender from AdvItemSuite, during which you can change the output size; or (2) Use the layer dimensions from in_data->width and in_data->height, which represent the unaffected layer dimensions. For more advanced control, consider migrating to SmartFX, which uses pre-render calls instead of Frame_Setup and supports 32-bit color depth.

*Tags: `output-rect`, `frame-setup`, `render-loop`, `aegp`*

---

## Can I change the output size during a re-render in After Effects plugins?

Yes, you can change the output size during a re-render. When you call for a re-render using PF_ForceRerender, you receive a Frame_Setup call first, during which you can modify the size even if the frame was previously rendered. This allows you to adjust output dimensions based on calculations from the previous render pass.

*Tags: `output-rect`, `frame-setup`, `aegp`, `render-loop`*

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

## How do you output the current frame number being rendered in an After Effects plugin using printf?

When accessing the current frame in a PF_Cmd_SMART_RENDER command, use the correct printf format specifier. The in_data->current_time field is an integer, so use %i instead of %s. Using %s (string format) will cause undefined behavior, including null values, blank output, and corrupted data like random ASCII characters appearing in the output.

```cpp
printf("\nRendering frame %i", in_data->current_time);
```

*Tags: `smart-render`, `debugging`, `aegp`*

---

## How can I open a file browser dialog in an After Effects plugin to let users select a file?

There is no direct C API call in the SDK to open a file browser dialog. However, you can use AEGP_ExecuteScript() to execute a script that opens a file browser and retrieves the selected file path, then pass that information back to your plugin.

*Tags: `aegp`, `ui`, `scripting`, `reference`*

---

## Can I set a higher priority for my custom AEIO to override Adobe's built-in handler for a format like FLV?

No, the After Effects SDK does not support overriding the priority of built-in AEIOs. After Effects will ignore custom AEIOs that associate to formats it already handles natively. The recommended workaround is to ask users to change the file extension to a different one (e.g., .flv to .custom_flv), and optionally create an import menu entry to automate this extension change for better user experience.

*Tags: `aeio`, `sdk`, `aegp`, `deployment`*

---

## How can I access and modify pixels in DrawSparseFrame of an AEIO without using Iterate8Suite?

You can create a PF_InData struct and fill it with whatever data you have available, which should allow you to use the iterate suites. Alternatively, you can iterate through the buffer's pixels directly using nested loops to access individual pixels via sampleIntegral32. However, be careful with coordinate ordering—ensure you're passing coordinates in the correct order (x,y) rather than swapped (y,x), and account for thumbnail resolution differences which can cause crashes or unexpected behavior.

```cpp
for (A_long i = 0; i < 1000; i++)
for (A_long j = 0; j < 1000; j++){
  PF_Pixel *pixel = sampleIntegral32(wP, i, j);
  pixel->alpha = PF_MAX_CHAN8;
  pixel->red = PF_MAX_CHAN8;
  pixel->green = PF_MAX_CHAN8;
  pixel->blue = PF_MAX_CHAN8;
}
```

*Tags: `aeio`, `aegp`, `memory`, `output-rect`, `debugging`*

---

## How can I get a list of the selected layers when my plugin is activated?

You can check the currently selected layers using the AEGP function AEGP_GetNewCollectionFromCompSelection(). This allows you to retrieve the user's current layer selection without modifying the scene.

*Tags: `aegp`, `layer-checkout`, `sdk`*

---

## How can I detect if an effect instance is already applied to a layer to prevent duplicate applications?

To detect if another instance of your effect is present on a layer, you need to scan the layer's effects using AEGP_GetLayerNumEffects and AEGP_GetLayerEffectByIndex, then check each effect's installed key using AEGP_GetInstalledKeyFromLayerEffect and match it to your own. If an effect is already present in a saved effect, you'll receive a SequenceResetup() or Unflatten() call instead of SequenceSetup(). Note that an effect cannot directly remove another effect from the same layer; you can either use a separate AEGP with an idle hook for deletion, or set a flag on the second instance to disable its processing.

*Tags: `aegp`, `params`, `sdk`*

---

## How can I toggle a stopwatch or create keyframes for slider parameters programmatically in an Effects plugin?

You can create keyframes for any parameter using AEGP_KEYFRAMESUITE3. Most AEGP suites work for both AEGP plugins and Effects plugins, including the keyframe suite. The "CheesyCheese" sample project demonstrates how to implement this functionality.

*Tags: `aegp`, `params`, `keyframe`, `effects`*

---

## Is there a sample project demonstrating how to use AEGP_KEYFRAMESUITE3 for keyframe creation?

The "CheesyCheese" sample project is included in the Adobe After Effects SDK and demonstrates how to implement keyframe creation using AEGP_KEYFRAMESUITE3. This example shows practical usage of the keyframe suite for both AEGP and Effects plugins.

*Tags: `reference`, `aegp`, `sample-code`, `keyframe`*

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

## How can I ensure a function runs after parameter values are updated in an After Effects plugin?

Instead of directly modifying parameter values and setting change flags, use AEGP_SetStreamValue() for instantaneous changes. There is no function guaranteed to be called after values change, but UPDATE_PARAMS_UI might get called and a render call is likely. Alternatively, you can set a flag in the sequence data to notify yourself of post-change operations that need to take place during UPDATE_PARAMS_UI or render calls.

```cpp
params[POINT]->u.td.x_value = FLOAT2FIX(pointX);
params[POINT]->u.td.y_value = FLOAT2FIX(pointY);
params[POINT]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `sequence-data`, `render-loop`*

---

## How can I check out only a portion of a frame at a different time instead of the entire frame?

You can use checkout_layer_pixels() which takes a request info structure as an argument that allows you to request a smaller rectangle. Alternatively, use the render suite to request the source item directly and set an ROI (region of interest) to get only the part you need. The render suite method doesn't hint AE in advance about your intentions, so AE won't cache items in advance—test both methods to see which is faster for your use case, as getting a potentially cached full frame might sometimes be faster than requesting a partial frame without caching.

*Tags: `layer-checkout`, `aegp`, `caching`, `output-rect`, `memory`, `render-loop`*

---

## How can an AEGP plugin modify a composition's current time indicator?

Use the AEGP_SetItemCurrentTime() function from the SDK. This function allows you to set the current time for a given composition item from an AEGP plugin, as opposed to effect plugins which use PF_MoveTimeStep().

*Tags: `aegp`, `scripting`, `reference`*

---

## How do you get input layer data with effects applied during an arbitrary draw event in After Effects?

Use extra->cb->checkout_layer_pixels() instead of PF_CHECKOUT_PARAM(), as the latter only fetches the layer's source without previous effects. However, checkout_layer_pixels() requires special suites available only during smart render calls and cannot be called during UI events. A better approach is to cache histogram data in sequence data during render calls and use PF_GetCurrentState and PF_HasParamChanged during draw events to detect parameter changes, then force re-render with PF_OutFlag_FORCE_RERENDER.

```cpp
typedef struct {
  bool didRender;
  /* histogram data */
} Histogram;
typedef struct {
  Histogram* histograms;
} my_sequence_data, *my_sequence_dataP, **my_sequence_dataH;

/* During render, cache at current time */
sequence->histograms[in_data->current_time / in_data->time_step]

/* During draw event, detect changes */
if (PF_HasParamChanged(in_data, extra)) {
  event_extraP->evt_out_flags |= PF_OutFlag_FORCE_RERENDER;
}
```

*Tags: `sequence-data`, `caching`, `smart-render`, `ui`, `aegp`*

---

## How can I validate image files before importing them into After Effects through a plugin?

There are two approaches: (1) Check the file using OS tools—both GDI+ (Windows) and Quartz (macOS) offer tools for reading image files in various formats to validate them before importing. (2) Use After Effects to check—use AEGP_StartQuietErrors to suppress user-visible errors, perform a test render to detect errors silently, then call AEGP_EndQuietErrors when done. Based on the result, you can decide whether to dispose of the image or notify the user.

*Tags: `aegp`, `image`, `validation`, `windows`, `macos`, `debugging`*

---

## Why does an After Effects effect plugin crash when dragging camera geometry values instead of typing them?

The crash occurs when keyframing masks during frameSetup after getting camera data, especially when the camera layer is positioned before the current layer in the composition hierarchy. The issue was specific to After Effects CS5.5 and disappeared in CS6 and later versions. A workaround is to check the version and deselect the camera or layer if needed when working with older versions.

```cpp
ERR(suites.PFInterfaceSuite1()->AEGP_ConvertEffectToCompTime(	in_data->effect_ref,
	in_data->current_time,
	in_data->time_scale,
	&comp_timeT));
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectCamera(	in_data->effect_ref,
	&comp_timeT,
	&camera_layerH));
if (camera_layerH){
	ERR(suites.LayerSuite6()->AEGP_GetLayerToWorldXformFromView(	camera_layerH,
		&comp_timeT,
		&comp_timeT,
		&matrix));
	AEGP_StreamVal2		camVal;
	ERR(suites.StreamSuite4()->AEGP_GetLayerStreamValue( camera_layerH, AEGP_LayerStream_ZOOM, AEGP_LTimeMode_CompTime, &comp_timeT, FALSE, &camVal, NULL));
	zoom = camVal.one_d;
}
```

*Tags: `aegp`, `debugging`, `camera`, `macos`, `windows`*

---

## How should you safely read camera data from an effect plugin to avoid crashes during parameter dragging?

Use PF_ABORT before reading camera data to check if an abort signal has been triggered. If you get an abort signal, skip reading the data. This helps prevent crashes that can occur during parameter dragging operations, particularly when dealing with composition cameras and keyframing operations.

*Tags: `aegp`, `params`, `debugging`, `render-loop`*

---

## How can you save and restore arbitrary data from a 3rd party plugin's property without using presets?

You can use the AEGP (After Effects General Plugin) approach: have an effect plugin communicate with an AEGP via a special suite (see "checkout" and "sweetie" samples). When the effect needs to apply changes, it sends data to the AEGP and stores it there without executing changes immediately. Let the effect finish execution and return. Then, during the AEGP's idle_hook call, check for messages from the effect and execute the changes after the effect is no longer in mid-call. This avoids crashes from modifying the scene while an effect is operating on it. Alternatively, you can bundle a preset file within your plugin and apply it via applyPreset() with a file path, avoiding the need to keep presets in Adobe's preset directory.

```cpp
var thePreset = File("C:/Program Files/Adobe/Adobe After Effects CS5.5/Support Files/Presets/Behaviors/wigglerama.ffx");
theLayer.applyPreset(thePreset);
```

*Tags: `aegp`, `arb-data`, `params`, `plugin-communication`*

---

## Why does calling AEGP_ExecuteScript from within an effect plugin crash After Effects?

Executing script commands that modify the scene state (like applying a preset to a layer) from within an effect plugin causes a crash because the script invalidates locked/checked-out memory objects that are currently passed to the effect plugin via in_data. The effect is in mid-call when the scene state changes, creating memory conflicts. You cannot add an effect to a layer while your effect is currently executing. The solution is to defer scene modifications until after the effect has finished execution.

*Tags: `aegp`, `debugging`, `memory`, `render-loop`*

---

## How can you pass platform-specific data from an effect plugin to an AEGP if PF_GET_PLATFORM_DATA() is unavailable in AEGPs?

Since AEGPs do not receive a valid in_data object and cannot use PF_GET_PLATFORM_DATA(), have the effect plugin obtain the platform-specific data it needs and pass that data directly to the AEGP when it sends it a message. The AEGP can then use this pre-obtained data without needing to call PF_GET_PLATFORM_DATA() itself. Alternatively, the effect can store the data in hidden properties or session data that the AEGP can query.

*Tags: `aegp`, `cross-platform`, `plugin-communication`*

---

## How can I run an idle hook function only once and then stop it from being called?

There is no built-in unregister function to stop receiving idle hook calls after AEGP_RegisterIdleHook(). Instead, you should set a flag in your plug-in that bypasses the idle hook code after the first run. For example, use a boolean SWITCH variable that you set to FALSE after the first execution, so subsequent idle hook calls skip the processing logic.

```cpp
BOOL SWITCH = TRUE;
static A_Err IdleHook(
  AEGP_GlobalRefcon plugin_refconP,
  AEGP_IdleRefcon refconP,
  A_long *max_sleepPL)
{
  *max_sleepPL = 500;
  A_Err err = A_Err_NONE;
  if (SWITCH) {
    // do something...
    SWITCH = FALSE;
  }
  return err;
}
```

*Tags: `aegp`, `idle-hook`, `plugin-architecture`, `threading`*

---

## Can I call AEGP functions from an external thread, and what is the correct pattern for using idle hooks with external threads?

You can call any AE function from an external thread, but only when AE is ready for it. The correct pattern is: (1) register the idle hook and launch the external thread from the entry point function, (2) have your external thread store its data in a global scope structure, (3) in the idle hook function, check if the global structure is filled and ready; if so, execute processing code; if not, check again next time, (4) set a skip idle hook flag to true when done and kill the external thread. Alternatively, make a synchronized call to your server from within the idle hook and avoid the external thread complexity.

*Tags: `aegp`, `threading`, `idle-hook`, `synchronization`, `plugin-architecture`*

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

## How can I efficiently update many parameters without causing UI slowness from repeated PF_ChangeFlag_CHANGED_VALUE flags?

Instead of setting PF_ChangeFlag_CHANGED_VALUE for each of many parameters, use AEGP_SetStreamValue() from the AEGP Suite to update parameter values more efficiently. Alternatively, if the parameters don't need to be directly exposed as individual sliders in the UI, store them in sequence data or arbitrary data, which are automatically saved with the project without requiring change flags for each value.

```cpp
params[paramId]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `aegp`, `ui`, `performance`*

---

## How can I hide layers in a composition using AEGP without getting internal verification errors?

Use AEGP_SetDynamicStreamFlag to hide layers, but suppress errors with AEGP_StartQuietErrors() and reset the error variable (err = NULL) after each call. The ERR() macro will skip subsequent commands if an error has occurred, so resetting err is necessary to continue loop execution. Additionally, check the stream type before calling AEGP_SetDynamicStreamFlag() to ensure you're only operating on valid layer streams (AEGP_StreamType_LAYER_ID).

```cpp
ERR(suites.DynamicStreamSuite4()->AEGP_SetDynamicStreamFlag(streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, TRUE));
err = NULL;
ERR2(suites.StreamSuite4()->AEGP_DisposeStream(streamH));
```

*Tags: `aegp`, `layer-checkout`, `debugging`*

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

## How do I define a mask parameter in an effect plugin?

Use the PathMaster sample project as a reference, which demonstrates how to add a path parameter to your effect. Note that mask selector parameters are limited to masks on the same layer as the effect only. There is no built-in API for a mask selector on a different layer. To access masks on other layers, use the AEGP suites, but be aware that the API function to render a mask will only work on the local effect layer with mask selector parameters. For cross-layer mask selection, you must either implement your own path rasterizing function or use workarounds such as invisible mask selectors pointing to temporary masks.

*Tags: `mask`, `params`, `aegp`, `ui`, `reference`*

---

## How do I rasterize a mask and use it to restrict pixel operations in my effect plugin?

You can rasterize a mask by filling a dummy buffer with color and applying the mask using the MaskWorldWithPath function from the mask suite. However, this requires the mask mode to be set to something other than NONE. A workaround is to use a temporary buffer for rasterization without affecting the project state. Note that calling AEGP_SetMaskMode is undoable and will prompt users to save changes. For querying whether pixels are contained in a mask, rasterize the mask extent into a temporary buffer and query it directly.

*Tags: `mask`, `aegp`, `memory`, `render-loop`*

---

## How can I allocate memory for a Windows GDI+ Bitmap object in an After Effects plugin without using new/delete to avoid crashes?

Use placement new syntax to construct an object in pre-allocated memory. First allocate memory using host_new_handle(), lock it with host_lock_handle(), then use the placement new operator: `Bitmap *tmpImg = new(memV) Bitmap(width, height, bitmap_bytes_per_rowL, PixelFormat32bppPARGB, reinterpret_cast<BYTE *>(bitmap_dataP));` This avoids mixing memory allocation strategies between the host and the plugin, which can cause crashes especially in Windows release builds.

```cpp
PF_Handle bmpH = suites.HandleSuite1()->host_new_handle(sizeof(Bitmap));
void *memV = suites.HandleSuite1()->host_lock_handle(bmpH);
Bitmap *tmpImg = new(memV) Bitmap(width, height, bitmap_bytes_per_rowL, PixelFormat32bppPARGB, reinterpret_cast<BYTE *>(bitmap_dataP));
```

*Tags: `memory`, `windows`, `aegp`, `debugging`*

---

## How can I access the path property of a shape layer using the AEGP API?

To access shape layer paths via AEGP API, you cannot use standard AEGP_GetNewLayerStream since shape properties are added dynamically and cannot be pre-indexed. Instead, use the Dynamic Stream Suite to recursively navigate the layer's streams. Start by getting the first stream of the shape layer, then iterate through streams by querying their names, types, and parent groups. You will eventually find the "content" group, which contains "shape" groups, and within those you can access the path data. Use suites like MaskOutlineSuite, PathQuerySuite, and PathDataSuite (which are AEGP-compatible despite their naming) to analyze the mask stream once located. This approach is necessary because the order and presence of shape properties are not guaranteed or pre-determined.

*Tags: `aegp`, `shape-layers`, `dynamic-streams`, `path-data`*

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

## How do you hide and show parameter groups dynamically in After Effects plugins when a checkbox is toggled?

Use AEGP_GetNewEffectStreamByIndex to get stream references for parameters in each group, then call AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to toggle visibility. Handle this in response to PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_UPDATE_PARAMS_UI commands. Note that you must hide/unhide every individual parameter within a group, not just the group topic itself, otherwise bugs can occur such as parameters appearing in the timeline when keyframed.

```cpp
AEGP_StreamRefH wfm_streamH = NULL;
A_Boolean hide_wfmB = !params[QPGA_SCOPE_OPTION_WFM]->u.bd.value;
AEGP_EffectRefH meH = NULL;
AEGP_SuiteHandler suites(in_data->pica_basicP);
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(NULL, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(NULL, meH, QPGA_WFM_TOPIC_START, &wfm_streamH));
ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(wfm_streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, hide_wfmB));
if (meH){
  ERR2(suites.EffectSuite2()->AEGP_DisposeEffect(meH));
}
if (wfm_streamH){
  ERR2(suites.StreamSuite2()->AEGP_DisposeStream(wfm_streamH));
}
```

*Tags: `params`, `ui`, `aegp`, `debugging`*

---

## Why do parameter groups not stay hidden when applying an effect preset in After Effects?

When applying an effect preset, parameter visibility flags are not preserved if only the group topic is hidden. You must explicitly hide every individual parameter within each group using AEGP_SetDynamicStreamFlag on each stream reference. Hiding only the topic without hiding each parameter will cause them to appear in the UI without their group containers. Sequence data does not affect this behavior.

*Tags: `params`, `ui`, `preset`, `aegp`*

---

## How do I properly handle AEGP_EffectRefH and AEGP_StreamRefH disposal when SmartRender operations are interrupted?

Every Get() call must be balanced by a Dispose() call, even when render is interrupted. The key issue is that if an interrupt causes your error variable to have a value, all commands wrapped in the ERR() macro will be skipped, preventing proper disposal. Solution: either set an interrupt flag and nullify the err var (remembering to return the canceled code to AE), or use ERR2() for interrupt checks. Additionally, ensure PF_ABORT is called during iteration and proper disposal happens before returning on abort.

```cpp
if (err = PF_ABORT(in_data)){
  if(outputWorld) (suites.WorldSuite3()->AEGP_Dispose(outputWorld));
  if(effectWorld) (suites.WorldSuite3()->AEGP_Dispose(effectWorld));
  if(streamH) (suites.StreamSuite4()->AEGP_DisposeStream(streamH));
  if(effectH) (suites.EffectSuite2()->AEGP_DisposeEffect(effectH));
  return err;
}
```

*Tags: `aegp`, `smart-render`, `memory`, `debugging`*

---

## How can I disable plugin parameters based on the layer type they are applied to?

Use the UPDATE_PARAMS_UI command instead of USER_CHANGED_PARAM to dynamically hide/unhide parameters. First, get the effect layer reference using AEGP_GetEffectLayer from PFInterfaceSuite1, then check the layer type using AEGP_GetLayerObjectType from LayerSuite5. The layer type will be one of: AEGP_ObjectType_AV (footage/solid/adjustment), AEGP_ObjectType_LIGHT, AEGP_ObjectType_CAMERA, AEGP_ObjectType_TEXT, or AEGP_ObjectType_VECTOR. Base your parameter visibility decisions on this layer type. The Supervisor sample demonstrates this pattern.

```cpp
AEGP_LayerH layerH;
ERR(suites.PFInterfaceSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH));
AEGP_ObjectType layerType = NULL;
ERR(suites.LayerSuite5()->AEGP_GetLayerObjectType(layerH, &layerType));
```

*Tags: `params`, `ui`, `aegp`, `layer-checkout`*

---

## What is a good reference sample for implementing dynamic parameter visibility in After Effects plugins?

The 'Supervisor' sample in the After Effects SDK demonstrates how to dynamically hide and unhide parameters at runtime. It shows the proper use of the UPDATE_PARAMS_UI command and how to leverage AEGP API calls to inspect layer properties and adjust the UI accordingly.

*Tags: `reference`, `ui`, `params`, `aegp`, `open-source`*

---

## Why does buffer height calculation blow up to around 500 when iterating through rows with zero gutter values?

The issue was caused by incorrect casting of the world as the base address for pixels. The correct approach is to use AEGP_GetBaseAddr32(worldH, &baseAddress) to properly obtain the base address for pixel access, rather than casting the world handle directly.

```cpp
ERR(ws3P->AEGP_GetBaseAddr32(worldH, &baseAddress));
```

*Tags: `memory`, `buffer`, `render-loop`, `aegp`, `debugging`*

---

## What flag should be used when creating a new world buffer for deep color pixel processing?

When creating a new world for processing, use the PF_NewWorldFlag_DEEP_PIXELS flag instead of PF_NewWorldFlag_NONE to properly allocate buffers for 16-bit and 32-bit color depths. This ensures the world is correctly initialized for deep color operations.

```cpp
ERR(in_data->utils->new_world(in_data->effect_ref, width, height, PF_NewWorldFlag_DEEP_PIXELS, &outputWorld));
```

*Tags: `memory`, `params`, `render-loop`, `aegp`*

---

## How can you handle PF_ColorDef values in 16-bit and 32-bit projects when the standard color picker uses 8-bit values?

Instead of using the standard PF_ColorDef color picker which stores A_u_char values, sample the color picker as an A_FpLong directly from the parameter stream during render, then convert the floating-point value to your target 16-bit or 32-bit format.

```cpp
// Sample color picker as A_FpLong and convert to 16bit value
A_FpLong colorValue;
// Read colorValue from stream as A_FpLong
// Convert to 16bit representation
```

*Tags: `params`, `ui`, `render-loop`, `aegp`*

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

## Is it possible to override or modify the After Effects Export to XFL command through the API?

No, the After Effects API does not offer means of overriding or modifying the export to XFL command. However, as an alternative, you could apply a script to the generated XML to correct it automatically after export, or run a script that exports data from AE directly in the format you need.

*Tags: `scripting`, `xfl`, `export`, `aegp`, `api`*

---

## Is it possible to create a custom output device in After Effects beyond the built-in options?

Yes, it is possible to create a custom output device in After Effects. The recommended approach is to base your plugin on the EMP (External Monitor Preview) sample plugin that is included in the SDK. This sample demonstrates how to implement custom output device functionality.

*Tags: `output-rect`, `aegp`, `reference`, `deployment`*

---

## What sample plugin should be used as a reference for building custom output device plugins?

The EMP (External Monitor Preview) sample plugin is the recommended reference for creating custom output devices in After Effects. This sample has been discussed in the Adobe forums and is included in the SDK documentation. On macOS, developers may also want to examine the Quicktime VOUT sample for additional reference.

*Tags: `output-rect`, `aegp`, `reference`, `macos`, `open-source`*

---

## How can you create dependent parameters that work correctly with keyframing and expressions in After Effects plugins?

When a parameter is keyframed or uses expressions, the PF_Cmd_USER_CHANGED_PARAM callback is not called during timeline scrubbing, making it difficult to synchronize dependent parameters. The PF_ParamFlag_SUPERVISE flag works for user-initiated changes but not for time-varying data. If the dependent parameter doesn't affect rendering (PUI_ONLY flag), you can use it. Otherwise, one practical solution is to use Motion Script expressions to express dependencies between sliders and attach these expressions during sequence setup, allowing the effect to use the computed parameters during render.

```cpp
params[SLIDER2]->u.fs_d.value = params[SLIDER1]->u.fs_d.value;
params[SLIDER2]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `params`, `keyframing`, `expressions`, `scripting`, `aegp`*

---

## How can I access the plugin's file path from within a Mac After Effects AEGP plugin to load bundled resources?

For EFFECT plugins, use PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH, path) after including AE_EffectCB.h. This requires the in_data structure which is only available to EFFECT plugins, not AEGP plugins. You may need to convert colons to forward slashes and add "VOLUMES/" at the beginning on Mac. For AEGP plugins (which don't receive in_data), alternative approaches include using getcwd() to read/write files by name only without a full path, or on Windows using GetModuleFileName((HINSTANCE)&__ImageBase, folder, sizeof(folder)).

```cpp
char path[1024] = {'\0'};
PF_GET_PLATFORM_DATA(PF_PlatData_EXE_FILE_PATH, path);
```

*Tags: `macos`, `aegp`, `plugin-resources`, `file-access`, `cross-platform`*

---

## How can I execute scripts from within an After Effects plugin?

You can use the AEGP_ExecuteScript() function call to run scripts directly from a plugin without needing to store them as separate files on disk. This is a built-in AEGP suite function for script execution.

*Tags: `aegp`, `scripting`, `api`*

---

## Why is a newly added layer not visible in the composition, showing the source layer instead on the current frame?

Adding a layer during a PF_Cmd_RENDER call causes rendering issues because the layers involved in the render are listed and checked prior to the render call. When you add a new layer during rendering, you mess with pre-made lists of render items. The solution is to add the new layer to the composition during other calls, preferably during PF_Cmd_UPDATE_PARAMS_UI or during idle process, not during render.

```cpp
ERR(suites.LayerSuite7()->AEGP_AddLayer(res_item, cResCompH, &res_layer));
```

*Tags: `aegp`, `layer-checkout`, `render-loop`, `debugging`, `params`*

---

## What AEGP SDK function can be used to apply a preset to a layer from a plugin?

There is no native C API method for applying an effect preset in the AEGP SDK. The recommended approach is to use AEGP_ExecuteScript to execute the Layer.applyPreset() JavaScript method instead.

*Tags: `aegp`, `scripting`, `sdk`, `reference`*

---

## Can I acquire multiple versions of the same After Effects suite in a single plugin?

Yes, you can acquire multiple versions of the same suite. Adobe's documentation states that new suite versions supersede old ones but never remove previously shipped suites. You can use AEFX_AcquireSuite() to explicitly request a specific suite version. However, be aware that rare exceptions exist where suites were deprecated between versions.

*Tags: `suites`, `aegp`, `cross-version`*

---

## How can I set a slider parameter to match the height or width of the layer the plugin is applied to?

PF_Cmd_PARAMS_SETUP occurs before the plugin is applied to the layer, so it cannot read layer parameters or even identify which layer it's on. Instead, use the PF_Cmd_UPDATE_PARAMS_UI callback that occurs afterwards to set parameter values. While the documentation recommends only cosmetic changes during this phase, you can use the Stream Suite to update parameter values programmatically rather than direct parameter array access.

*Tags: `params`, `aegp`, `effect_api`, `sdk`*

---

## Why does After Effects report 'could not locate entrypoint' when loading a plugin?

The entrypoint function must be exported with the DllExport keyword on Windows. Without this declaration, After Effects cannot locate the EntryPointFunc symbol in the compiled plugin binary.

```cpp
DllExport PF_Err EntryPointFunc (
    PF_Cmd cmd,
    PF_InData *in_data,
    PF_OutData *out_data,
    PF_ParamDef *params[],
    PF_LayerDef *output,
    void *extra )
```

*Tags: `aegp`, `pipl`, `debugging`, `windows`, `build`*

---

## What causes 'parameter count mismatch in plug-in effect' error in After Effects plugins?

This error occurs when out_data->num_params does not match the actual number of parameters defined in ParamsSetup. The num_params value must equal the last entry in your parameter enumeration (e.g., SHIFT_NUM_PARAMS), which counts all parameters including the base layer input at index 0. Additionally, each parameter must have a unique stream ID assigned for proper data persistence across plugin versions.

```cpp
enum {
    BASE = 0,           // layer input always at 0
    MY_SLIDER_1,
    MY_SLIDER_2,
    MY_POINT,
    NUM_PARAMS          // use this for out_data->num_params
};

out_data->num_params = NUM_PARAMS;
```

*Tags: `params`, `aegp`, `debugging`*

---

## How should parameter stream IDs be managed when modifying plugin parameter order across versions?

Maintain separate enumerations: one for UI order (parameter definition order) and one for disk IDs (stream identifiers). When adding or reordering parameters in a new plugin version, keep the disk ID enumeration unchanged from previous versions. This allows After Effects to correctly map saved project data to parameters even if their UI positions change. For example, if a slider had disk_id=7 in v1, keep it at 7 in v2 even if it moves to a different position in the UI enumeration.

```cpp
// UI enumeration (definition order)
enum {
    BASE = 0,
    SLIDER_1,
    SLIDER_2,
    POINT,
    NUM_PARAMS
};

// Disk ID enumeration (never change these values)
enum {
    SLIDER_1_ID = 1,
    SLIDER_2_ID = 2,
    POINT_ID = 3
};
```

*Tags: `params`, `aegp`, `deployment`*

---

## How can I access pixel data from an AEGP plugin?

An AEGP plugin cannot directly access layer pixels with effects, masks, or transformations applied. Instead, you can access the source pixels of a layer by getting the source item and then using AEGP_RenderAndCheckoutFrame() to render and checkout the frame, followed by AEGP_GetReceiptWorld() to retrieve the pixel data. The source provides pixels without any masks, effects, or transformation applied.

*Tags: `aegp`, `pixel-data`, `layer-checkout`, `reference`*

---

## How do you open and read files in an AEIO plugin using the SDK API?

The After Effects SDK does not provide built-in file I/O functions. Instead, you use standard C library functions like fopen, fread, and fclose to open and read files. When you receive a file path as A_UTF16Char in functions like VerifyFileImportable(), you can use it directly with these standard functions.

*Tags: `aeio`, `import`, `sdk`, `aegp`*

---

## How do you get the position stream for a nested composition layer in an AEGP plugin?

When you nest a composition (compA) into another composition (compB), the nested composition becomes a layer in compB. To get the position stream for that layer, you need to: 1) Use AEGP_GetCompLayerByIndex() on compB to get the AEGP_LayerH for the layer containing the nested composition, or receive it directly when nesting. 2) Then use AEGP_GetNewLayerStream() with AEGP_LayerStream_POSITION on that layer handle to get the position stream. Note that compositions themselves don't have positions—only layers within compositions do. If you're trying to find which comp a nested composition is being used in, you must scan the project and check each layer's source, as there is no direct API to query nesting relationships.

*Tags: `aegp`, `nested_comp`, `layer-checkout`, `reference`*

---

## Why don't animations on a nested composition appear when querying the source composition in an AEGP plugin?

When you have a composition (compA) with animated text layers that is then nested as a layer in another composition (compB) with animation applied to the nested layer itself, you will only see the text layer animations if you query compA directly. The animation applied to the nested composition layer in compB is separate and stored with that layer in compB, not in compA. To see both animations combined, you need to query the layer in compB that contains the nested composition and retrieve its position stream using AEGP_GetNewLayerStream() with AEGP_LayerStream_POSITION.

*Tags: `aegp`, `nested_comp`, `layer-checkout`, `params`*

---

## Is there a direct C++ API method to replace a layer's source in After Effects?

There is no direct C++ AEGP API method for replacing a layer's source. The functionality exists in the Java API as layer.source.replace(), but in C++ it must be accessed indirectly using DoCommand with command ID 2299. However, this indirect method requires making the composition the frontmost window, which is difficult to achieve from an AEGP plugin. The feature simulates alt+dragging a new source onto a selected layer in the composition window.

*Tags: `aegp`, `api`, `layer`, `sdk`*

---

## How do you disable, enable, and hide parameter controls based on a popup selection in an After Effects plugin?

To disable a parameter, use PF_ParamFlag_SUPERVISE on the popup and handle PF_Cmd_USER_CHANGED_PARAM. In the UserChangedParam callback, disable controls by setting ui_flags |= PF_PUI_DISABLED and enable by clearing the flag with &= ~PF_PUI_DISABLED. To hide parameters, use the DynamicStreamSuite2 with AEGP_SetDynamicStreamFlag and AEGP_DynStreamFlag_HIDDEN. First get the effect handle with AEGP_GetNewEffectForEffect, then get the stream with AEGP_GetNewEffectStreamByIndex, then call AEGP_SetDynamicStreamFlag. When hiding topics, hide all parameters within the topic individually; the topic end param does not need to be hidden. Update the UI with PF_UpdateParamUI after making changes.

```cpp
case PF_Cmd_USER_CHANGED_PARAM:
  ERR(UserChangedParam(in_data, out_data, params, output, reinterpret_cast<PF_UserChangedParamExtra *>(extra)));
  break;

// In UserChangedParam:
if (extra->param_index == POPUP_MODE_ID) {
  if (params[POPUP_MODE_ID]->u.cd.value == 2) {
    params[CONTROL_ID]->ui_flags |= PF_PUI_DISABLED;
  } else {
    params[CONTROL_ID]->ui_flags &= ~PF_PUI_DISABLED;
  }
  AEGP_SuiteHandler suites(in_data->pica_basicP);
  ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(
    in_data->effect_ref,
    CONTROL_ID,
    params[CONTROL_ID]
  ));
}

// For hiding with DynamicStreamSuite:
AEGP_SuiteHandler suites(in_data->pica_basicP);
ERR(suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(globP->my_id, in_data->effect_ref, &meH));
ERR(suites.StreamSuite2()->AEGP_GetNewEffectStreamByIndex(globP->my_id, meH, SUPER_FLAVOR, &flavor_streamH));
ERR(suites.DynamicStreamSuite2()->AEGP_SetDynamicStreamFlag(flavor_streamH, AEGP_DynStreamFlag_HIDDEN, FALSE, hide_themB));
```

*Tags: `params`, `ui`, `aegp`, `reference`*

---

## Is there a sample plugin that demonstrates conditional parameter visibility and disabling in After Effects?

The 'Supervisor' sample project included in the After Effects SDK demonstrates how to hide, show, and disable parameters based on user interaction. It shows the proper use of AEGP_GetNewEffectForEffect, AEGP_GetNewEffectStreamByIndex, and AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to control parameter visibility.

*Tags: `ui`, `params`, `reference`, `open-source`, `aegp`*

---

## Why can't I access parameter input layer pixel data in the DoClick function?

The incoming image data is only available during the render call, not in DoClick. However, there are several workarounds: (1) Store input/output pixels in sequence data during render for later access, though cached frames won't update; (2) Use AEGP_GetReceiptWorld() to fetch pixels of any video item at any time, though rendering composites can be slow; (3) In CS4 and earlier, use the UI drawing mechanism during the draw event; (4) Best practice: store click location and flags in sequence data during DoClick, then trigger a re-render to process pixels during the render call.

*Tags: `doclick`, `pixel`, `params`, `sequence-data`, `render-loop`, `aegp`*

---

## What is the correct way to create custom hidden streams and store data on layers using AEGP?

There are limited options for storing custom data on layers with AEGP. Dynamic streams (like strokes from the paint tool and text animators) exist but are very specific. For persistent data across sessions, the recommended approach is to keep the data within the AEGP itself rather than on the layer, storing each layer's ID and its comp's project item ID to identify and retrieve the associated data. For temporary data that can be regenerated, storing it in the AEGP is also preferred. Alternative approaches include using effect parameters, layer comments, or layer marker comments, though these have limitations with the SDK.

*Tags: `aegp`, `arb-data`, `layer-checkout`, `sdk`*

---

## How can non-layer effect data be persisted and saved with an After Effects project?

Refer to the Adobe forum discussion at http://forums.adobe.com/message/2837630#2837630 which discusses techniques for saving non-layer effect data with the project file. This resource addresses the challenge of maintaining custom data across sessions in AEGP plugins.

*Tags: `aegp`, `arb-data`, `reference`*

---

## How can I detect when a composition's color depth or pixel format is changed by the user?

There is no direct event for color depth changes, but the frame is re-rendered when the comp depth is changed. You can check the current pixel format using PF_GetPixelFormat() during the render call. To update parameter UI in response, use PF_UpdateParamUI during the render call, and you can hide/unhide parameters at any time using AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN. Parameter property changes can be made when receiving events or PARAMS_UI_CHANGED.

*Tags: `params`, `ui`, `aegp`, `pixel_format`, `bit_depth`*

---

## How can two AEGPs communicate with each other?

Custom suites are the way to enable inter-AEGP communication. The key is to ensure all AEGPs have loaded before attempting to use the suite, since AE loads AEGPs before effects. The first idle call is a good time to load and use the suite. Refer to the 'sweetie' sample in the SDK to see how to create a custom suite, and the 'checkout' sample (globalSetup() function) to see how to use a custom suite.

*Tags: `aegp`, `custom-suite`, `reference`*

---

## What are good SDK samples to learn about custom suites in After Effects?

The 'sweetie' sample demonstrates how to create a custom suite for inter-AEGP communication. The 'checkout' sample shows how to use a custom suite, particularly in the globalSetup() function. These samples are included in the After Effects SDK and serve as the primary documentation for custom suite implementation.

*Tags: `aegp`, `open-source`, `reference`, `sdk`*

---

## Is there an API to draw standard After Effects controls like buttons and checkboxes outside the Effect Controls Window?

No, there is no API available to draw standard AE controls outside the ECW (Effect Controls Window). Effect controls can only be created during param_setup in effects and will display only in the ECW and timeline. Custom controls would need to be reimplemented with custom drawing code.

*Tags: `ui`, `params`, `aegp`, `reference`*

---

## What does host_lock_handle do and how should memory be allocated in After Effects plugins?

host_lock_handle tells After Effects that a memory chunk is in use and should not be moved by the operating system. In After Effects plugins, you should use the memory suites API (host_create_handle + host_lock_handle) rather than malloc/new, to allow AE to make RAM allocation decisions at the application scope. The pattern host_create_handle + host_lock_handle is functionally equivalent to malloc. For handles you allocate, you must lock, unlock, and dispose of them. Some handles are allocated and locked for you by AE (like those from ExecuteScript), while others like global_data and sequence_data are managed by AE automatically when the plugin is called.

*Tags: `memory`, `aegp`, `plugin-api`*

---

## Does host_lock_handle provide thread safety and is it recursive?

host_lock_handle does NOT provide thread safety—it only prevents AE from moving the memory chunk. Regarding recursiveness (multiple locks on the same handle), the behavior is unclear. Calling host_lock_handle twice followed by a single host_unlock_handle may or may not leave the handle locked, depending on whether the locking mechanism is reference-counted. It's safest to maintain a 1:1 ratio of lock to unlock calls per handle.

*Tags: `memory`, `threading`, `aegp`*

---

## How can I catch mouse wheel events in After Effects plugins?

After Effects does not supply effects with mouse wheel events through its standard API. The SDK only provides clicks, drags, and cursor enter/exit events for UI regions. If you need mouse wheel support on Windows, you can use the HWND descriptor approach, but this is only available for AEGP Palette plugins, not standard effects.

*Tags: `ui`, `aegp`, `windows`, `debugging`*

---

## How can I detect whether a layer is audio-only in After Effects?

There are multiple approaches to detect audio-only layers: (1) Use AEGP_GetLayerFlags() to check AEGP_LayerFlag_VIDEO_ACTIVE and AEGP_LayerFlag_AUDIO_ACTIVE flags, though this only tells if video is off, not if it's a true audio layer. (2) Use AEGP_GetLayerSourceItem() to get the source item, then call AEGP_GetItemFlags() and check AEGP_ItemFlag_HAS_VIDEO and AEGP_ItemFlag_HAS_AUDIO—if the source has audio but no video, it's definitely audio-only. (3) As a last resort, use AEGP_ExecuteScript() with JavaScript for rare checks. Additionally, audio layers typically have dimensions of 0x0 in the project panel, which can serve as another detection criterion.

*Tags: `aegp`, `layer-checkout`, `reference`*

---

## How can an AEGP plugin check if there is an active camera in the current composition?

Use the AEGP_GetCameraType() function with a layer handle to determine the camera type. This function returns one of three values: AEGP_CameraType_NONE (value -1) if no camera, AEGP_CameraType_PERSPECTIVE, or AEGP_CameraType_ORTHOGRAPHIC. Alternatively, iterate through the composition's layers and identify the first camera layer that is not out of trim at the current time—this will be the active camera. If no camera layers are found, the composition has no active camera.

```cpp
AEGP_GetCameraType(layerH) returns:
AEGP_CameraType_NONE = -1
AEGP_CameraType_PERSPECTIVE
AEGP_CameraType_ORTHOGRAPHIC
```

*Tags: `aegp`, `camera`, `composition`, `layer-checkout`*

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

## Why does calling PF_ForceRerender() on a text layer result in 'layer does not have a source' error?

Text layers and shape layers do not have a source, unlike solids. Any source-related operation performed on a text layer will generate this error. As a workaround, instead of using PF_ForceRerender(), you can add a dummy parameter to the effect and call AEGP_SetStreamValue() to trigger rendering of all layers with the effect.

*Tags: `aegp`, `text-layer`, `params`, `debugging`*

---

## How can I set a camera to One-Node type when creating it via AEGP plugin?

Use AEGP_SetLayerFlag() with the flags AEGP_LayerFlag_LOOK_AT_POI or AEGP_LayerFlag_AUTO_ORIENT_ROTATION to control camera node type. Disabling these flags will give you one-node camera behavior. Alternatively, you can execute a script using AEGP_ExecuteScript() to set autoOrient properties directly.

```cpp
AEGP_ExecuteScript("app.project.item(compItem).layer(camLayerIndex).autoOrient = AutoOrientType.NO_AUTO_ORIENT");
```

*Tags: `aegp`, `camera`, `layer-checkout`*

---

## How can I access the composited color of layers beneath the current layer in After Effects?

There is no direct API to access intermediate composition buffers during normal effect rendering, as After Effects does not render bottom-to-top and optimizes by skipping rendering of fully opaque regions. However, several workarounds exist: (1) Write an AEGP artisan-type plugin (see the 'arti' sample) which handles composition rendering and has access to intermediate buffers, though this is very difficult; (2) Create a duplicate composition with only needed layers and render it using AEGP_GetReceiptWorld(); (3) Apply your effect on an adjustment layer, which receives the composited buffer of underlying layers, then use checked-out layer params to access original sources; (4) Use sampleImage() expressions on hidden parameters to sample pixel data, though this is slow and limited; (5) Ask the user to provide the background as a layer parameter, a common practice in effects that need background information.

*Tags: `aegp`, `render-loop`, `layer-checkout`, `sdk`, `params`*

---

## What is the AEGP artisan sample plugin and how does it relate to composition rendering?

The 'arti' sample is an AEGP (After Effects General Plugin) of type 'artisan' that demonstrates how to write a custom composition renderer. Artisan plugins replace After Effects' default renderer (like 'advanced3D') and have direct access to intermediate composition rendering results. This makes them capable of accessing composited buffers at different rendering stages, unlike standard effect plugins which only see the final result. This approach is recommended for advanced scenarios requiring intermediate buffer access, though it is noted as being very difficult to implement correctly.

*Tags: `aegp`, `reference`, `render-loop`, `open-source`*

---

## How do you retrieve shape layer content parameters (like Polystar, ZigZag, Repeater) in AEGP?

Shape layer contents are not accessed the same way as effects. For path data, use PF_PathDataSuite1 to retrieve path vertices. For rendered pixels, you cannot directly read the source since shape layers don't have one. As an alternative, you can get the shape and render it yourself, or duplicate the composition, delete everything except the shape layer, and use AEGP_GetReceiptWorld() to render that new composition. The specific method was discussed in detail in an Adobe forum thread.

*Tags: `aegp`, `shape-layer`, `params`, `reference`*

---

## What is the correct AEGP API method to access shape layer contents versus effects?

To get layer effects, use AEGP_GetLayerEffectByIndex. However, shape layer contents (like Polystar, ZigZag, Repeater) are not accessed through the stream suite or effect methods. Instead, use PF_PathDataSuite1 for path geometry, or employ workarounds such as rendering the shape yourself or using AEGP_GetReceiptWorld() on a duplicate composition containing only the shape layer.

*Tags: `aegp`, `shape-layer`, `params`, `debugging`*

---

## How can I create an After Effects plugin that samples the current frame and displays information graphically in a separate window?

You should build an AEGP (After Effects General Plugin) rather than an effect plugin. Use the 'panelator' sample as a base to create a dockable panel for displaying your graphical output. To access frame data, you have two approaches: (1) Use the 'EMP' (external monitor preview) sample which provides the MyBlit() function to receive the currently viewed composition buffer, registered during entryPoint(). (2) Use AEGP functions to render project items on demand: AEGP_GetMostRecentlyUsedComp, AEGP_RenderAndCheckoutFrame, and AEGP_GetReceiptWorld. The second approach can leverage AE's render cache to avoid re-rendering if the frame has already been processed.

*Tags: `aegp`, `ui`, `output-rect`, `reference`*

---

## What is the panelator sample plugin used for?

The 'panelator' sample is an AEGP that demonstrates how to create a dockable panel in After Effects, similar to built-in palettes like the Info palette. It serves as a template for AEGP plugins that need to display custom user interfaces and allows you to draw arbitrary content within the panel.

*Tags: `aegp`, `ui`, `reference`, `sample`*

---

## What is the EMP sample and how does it deliver frame data to plugins?

EMP (External Monitor Preview) is a sample plugin template that demonstrates how to receive image data from After Effects. It uses the MyBlit() function as the callback where AE delivers the currently viewed composition buffer to the plugin. This function is registered during the entryPoint() function and is useful for plugins that need real-time access to the composition being viewed.

*Tags: `aegp`, `output-rect`, `reference`, `sample`*

---

## How can you create dynamic stroke parameters similar to the Paint effect in After Effects plugins?

Strokes are handled by the dynamicStreamSuite, which allows you to add, remove, and alter strokes and text layer parameters. However, custom parameters that behave like strokes cannot be created through the API. Two workarounds are: (1) Create your effect with hidden parameter groups (e.g., 50 groups, each with width, length, start, and end sliders) and unhide them as needed—this is simpler but limited by the number of compiled groups. (2) Create a dummy effect with the desired sliders and add new instances of it to the layer whenever more strokes are needed—this is more flexible and allows unlimited strokes but is significantly more complex to implement.

*Tags: `params`, `ui`, `sdk`, `aegp`, `pipl`*

---

## Can you create custom stream group types in After Effects plugins beyond existing types?

No, you can only add stream groups and atoms of existing types through the API. You cannot create custom types of your own. Any custom parameter grouping behavior must be implemented using workarounds such as hidden parameter groups or multiple plugin instances.

*Tags: `params`, `sdk`, `aegp`*

---

## How do you use PF_AppGetColor() to retrieve After Effects application colors in a custom effect?

Use the AppSuite3() to call PF_AppGetColor() with the desired color type and a pointer to store the result. The syntax is: suites.AppSuite3()->PF_AppGetColor(PF_App_Color_HOT_TEXT, &appColor1); where PF_App_Color_HOT_TEXT is the color type you want to retrieve and appColor1 is the variable that will store the retrieved color.

```cpp
suites.AppSuite3()->PF_AppGetColor(PF_App_Color_HOT_TEXT, &appColor1);
```

*Tags: `ui`, `aegp`, `params`, `reference`*

---

## How can I force a render on a custom effect from AEGP without adding the action to the undo/redo stack?

Create an undo group with a NULL instead of a name. Anything that happens in that group will NOT be added to the undo stack. Alternatively, you can use AEGP_SetStreamValue() to change an invisible parameter's value, which will trigger a render without affecting the result, but this approach will add the action to the undo/redo stack unless wrapped in a NULL undo group.

*Tags: `aegp`, `undo`, `render-loop`, `params`*

---

## What is the correct way to trigger a force render on a custom effect from AEGP?

You can use AEGP_SetStreamValue() to modify an invisible parameter value, which will trigger a render without affecting the effect output. Note that setting PF_OutFlag_Force_Rerender in PF_CMD_COMPLETELY_GENERAL does not work for triggering renders from AEGP.

*Tags: `aegp`, `params`, `render-loop`*

---

## When duplicating a layer with custom effects, which effect receives the PF_Cmd_SEQUENCE_FLATTEN command—the old or new effect?

The FLATTEN command is sent only to the old effect. It tells the effect to collect and prepare all its data for storing. Both the old and new effects receive the UNFLATTEN command afterward. To distinguish between old and new effects during UNFLATTEN, store identification data during FLATTEN (such as effect index, layer ID, and composition item ID), then check these identifiers during UNFLATTEN or the first updateParamsUI call to determine which effect is which.

*Tags: `sequence-data`, `params`, `aegp`*

---

## How do I distinguish between the original and duplicated effect when both have identical sequence data after duplication?

Store identifying information about the effect during the FLATTEN command, such as its effect index, layer ID, and composition item ID. During the UNFLATTEN command (or preferably at the first updateParamsUI call), compare the current effect's properties against the stored identification data. If the properties match, it is the old effect; if any differ, it is the new duplicated effect.

*Tags: `sequence-data`, `params`, `aegp`*

---

## How can I set an effect parameter value on a layer in an After Effects plugin?

Effect parameters in After Effects are referred to as streams. Use the AEGP_SetStreamValue() function to set parameter values. To determine which stream index corresponds to which parameter, you need to experiment with different effects and learn their stream numbers. If you're applying effects you've written yourself, you should know the parameter index numbers directly.

*Tags: `aegp`, `params`, `sdk`*

---

## How can I prevent a slider parameter from resetting to its default value when the Effect Reset button is clicked?

You cannot tell a parameter to ignore the reset operation directly. However, you can store the parameter value in sequence data and restore it when the SEQUENCE_RESETUP callback is triggered. Alternatively, if the parameter is used as a unique identifier, you can skip storing it as a parameter altogether and maintain it only in sequence data, retrieving it via AEGP_EffectCallGeneric() when needed. You could also try setting a different default value during the param_setup call, though param_setup is typically called only once when the effect is first applied.

*Tags: `params`, `sequence-data`, `aegp`, `sdk`*

---

## How can an effect force full resolution rendering when the user has downsampled the composition?

Use the RenderAndCheckoutFrame AEGP function to render at full resolution. However, be careful to render the composition your effect is applied IN (not TO) to avoid infinite loops. You can check the downsample factors using AEGP_GetDownsampleFactor and conditionally render full resolution only when downsampling is detected. Make sure to use AEGP_GetEffectLayer instead of AEGP_GetActiveLayer to get your effect's layer, as 'active' doesn't necessarily refer to your layer.

```cpp
if (1 != in_data->downsample_x.num || 1 != in_data->downsample_x.den || 1 != in_data->downsample_y.num || 1 != in_data->downsample_y.den) {
  AEGP_ItemH itemH = NULL;
  AEGP_LayerH layerH = NULL;
  suites.EffectSuite1()->AEGP_GetEffectLayer(in_data->effect_ref, &layerH);
  // Set downsample factor to 1,1 for full resolution
  suites.RenderOptionsSuite2()->AEGP_SetDownsampleFactor(roH, 1, 1);
  suites.RenderSuite2()->AEGP_RenderAndCheckoutFrame(roH, NULL, NULL, &receiptH);
}
```

*Tags: `aegp`, `smart-render`, `layer-checkout`, `memory`*

---

## Why am I getting black pixels (0,0,0,0 RGBA) when rendering a full resolution frame with RenderAndCheckoutFrame?

The issue is likely that you're using AEGP_GetActiveItem and AEGP_GetActiveLayer, which don't necessarily refer to your effect's layer. Use AEGP_GetEffectLayer instead, which is guaranteed to return your effect's actual layer. The black pixels suggest the render is not finding the correct layer to render.

*Tags: `aegp`, `smart-render`, `debugging`, `memory`*

---

## How do I properly update color parameters in nested precomposition layers from an effect plugin?

When modifying color values in fill effects within precomposition layers, use AEGP_GetEffectLayer instead of AEGP_GetActiveLayer to ensure you're working with the correct effect layer. Do not call AEGP_GetNewStreamValue before AEGP_SetStreamValue unless you specifically need to read the value first—if you do read it, you must dispose of it with AEGP_DisposeStreamValue. Implement proper parameter supervision during PF_Cmd_USER_CHANGED_PARAM callbacks and ensure you're correctly updating only the affected parameters rather than all of them.

```cpp
AEGP_LayerH effectLayer = NULL;
suites.LayerSuite5()->AEGP_GetEffectLayer(pluginID, effectRefH, &effectLayer);
AEGP_ItemH item = NULL;
suites.LayerSuite5()->AEGP_GetLayerSourceItem(effectLayer, &item);
AEGP_CompH preComp;
suites.CompSuite6()->AEGP_GetCompFromItem(item, &preComp);
AEGP_StreamRefH streamRef;
suites.StreamSuite3()->AEGP_GetNewEffectStreamByIndex(pluginID, eff, 3, &streamRef);
AEGP_StreamValue2 val;
val.val.color.alphaF = params[paramIndex]->u.cd.value.alpha;
val.val.color.redF = params[paramIndex]->u.cd.value.red;
val.val.color.greenF = params[paramIndex]->u.cd.value.green;
val.val.color.blueF = params[paramIndex]->u.cd.value.blue;
suites.StreamSuite3()->AEGP_SetStreamValue(pluginID, streamRef, &val);
```

*Tags: `aegp`, `params`, `ui`, `smartfx`*

---

## How do you properly implement inter-effect communication using AEGP_EffectCallGeneric in After Effects plugins?

When calling an effect from an AEGP (like Sweetie calling Checkout), you must handle timing carefully. The key issue is that you cannot call an effect back immediately while it's still processing its original call to your AEGP. Instead, use idle_hook: when the effect calls your AEGP, set a flag and wait for the next idle_hook call, then initiate AEGP_EffectCallGeneric when the effect is guaranteed to be idle. Alternatively, respond immediately via the same custom suite that did the calling, passing data for the calling effect to process without needing effectCallGeneric. Do not use AEGP_GetNewEffectForEffect with a passed effect_ref—instead, use AEGP_GetLayerEffectByIndex to look up the effect on the target layer. For permanent effect tracking across the project lifetime, store the comp itemID and layer layerID rather than relying on EffectRefH which can become invalid.

```cpp
// From AEGP: wait for idle hook before calling effect back
if (AEFX_AcquireSuite(in_data, out_data, kDuckSuite1, kDuckSuiteVersion1, "Couldn't load suite.", (void**)&dsP)) {
  PF_STRCPY(out_data->return_msg, "No Duck Suite!");
} else {
  if (dsP) {
    dsP->Quack(2);
  }
  AEFX_ReleaseSuite(in_data, out_data, kDuckSuite1, kDuckSuiteVersion1, "Couldn't release suite.");
}

// From Effect: pass in_data to AEGP via custom suite for callback
static SPAPI A_Err CalledFromEffect(void* ptr) {
  PF_InData* pIndata = static_cast<PF_InData*>(ptr);
  // Use AEGP_GetLayerEffectByIndex instead of AEGP_GetNewEffectForEffect
}
```

*Tags: `aegp`, `inter-plugin`, `timing`, `idle-hook`, `effect-ref`, `reference`*

---

## What are the reference projects to study for implementing custom AEGP suites and inter-plugin communication?

The Sweetie and Checkout sample projects in the After Effects SDK demonstrate custom suite implementation for effect-to-AEGP communication. ProjectDumper and Shifter are also referenced as examples of inter-plugin communication patterns. These samples show how to use custom suites (like the Duck Suite example) for effects to safely communicate with AEGPs and other plugins.

*Tags: `reference`, `open-source`, `aegp`, `sample-code`, `sdk`*

---

## How can you re-apply a mask after modifying the alpha channel in an After Effects plugin?

There are two main approaches: (1) Use PF_MaskWorldWithPath on each mask and composite them in the correct transfer mode to recreate the layer's mask matte. Note that PF_MaskWorldWithPath does not support mask expansion, so you must render expanded masks manually. Read feather, opacity, and other mask values using AEGP_GetNewMaskStream or AEGP_GetNewMaskOpacity, then access numeric values with AEGP_GetNewStreamValue. (2) A simpler alternative: checkout the layer's original pixels (without masks applied) using a hidden layer parameter and PF_CHECKOUT_PARAM (or checkout_layer for smartFX), then compare the alpha difference between the original and masked input pixels to derive the mask matte. The second method is easier but less precise than fully recreating masks.

*Tags: `aegp`, `params`, `layer-checkout`, `mfr`, `smartfx`*

---

## How can you hide programmatically created layers like cameras from the AE timeline panel?

Use AEGP_GetNewStreamRefForLayer to get a stream reference for the camera layer, then call AEGP_SetDynamicStreamFlag with AEGP_DynStreamFlag_HIDDEN to hide it from the timeline. However, this approach has risks—a better alternative is to keep the layer visible but locked, similar to how plugins like Particular handle added elements, to prevent accidental deletion by users.

```cpp
AEGP_StreamRefH streamPH = NULL;
err = suites.DynamicStreamSuite3()->AEGP_GetNewStreamRefForLayer(NULL, layerPH, &streamPH);
err = suites.DynamicStreamSuite3()->AEGP_SetDynamicStreamFlag(streamPH, AEGP_DynStreamFlag_HIDDEN, false, true);
```

*Tags: `aegp`, `ui`, `layer-checkout`*

---

## Should you dispose of stream references acquired with AEGP_GetNewStreamRefForLayer, and when?

Yes, you must dispose of the stream reference using AEGP_DisposeStream to release the streamPH you acquired. Do not use AEGP_DeleteStream, which is only for deleting streams like paint strokes. Every getStream call must be balanced by a disposeStream call. It is recommended to dispose of the stream as soon as you're done with it, or at minimum before returning from the call that triggered the process.

*Tags: `aegp`, `memory`, `caching`*

---

## How can I hide an effect from the Effects menu while still allowing it to be added programmatically?

Set the PF_OutFlag_I_AM_OBSOLETE flag during global setup, and the effect will not appear in the effects list but can still be added using addEffect from code.

*Tags: `ui`, `pipl`, `aegp`*

---

## How do you access pixel data from multiple layers in an After Effects effect plugin?

To access pixel data from multiple layers in an effect plugin, you need to create a layer parameter (e.g., param number 3) that allows the user to select any layer from the composition. Then use PF_CHECKOUT_PARAM to checkout that parameter. The pixel data can be accessed from the resulting paramDef structure at paramDef->u.ld.data (or similar location in the structure). This approach allows an effect on one layer to access and process pixels from any other layer in the composition.

```cpp
paramDef->u.ld.data
```

*Tags: `params`, `layer-checkout`, `aegp`, `mfr`*

---

## What is the difference between using layer parameters for effects versus using AEGP for multi-layer access?

For effect-type plugins, use a layer parameter with PF_CHECKOUT_PARAM to access multiple layers selected by the user. For AEGP (After Effects General Plugin) development, the approach is more complex and requires using AEGP_RenderAndCheckoutFrame, which involves finding the project item you want to reference. The layer parameter approach is simpler for effects and allows users to dynamically select which layers to access.

*Tags: `aegp`, `params`, `layer-checkout`, `render-loop`*

---

## Can I use sequence_data to store a circular buffer for frame caching in an After Effects temporal filter effect?

Yes, you can use sequence_data to store a circular buffer. Allocate memory using PF_NEW_HANDLE during PF_Cmd_SEQUENCE_SETUP (though you can allocate at any time). Check if in_data->sequence_data == NULL to determine if memory has been allocated. Be aware of important considerations: (1) You cannot reliably know if cached frames are still valid after user edits, (2) Frames may be acquired out of order if the user scrubs randomly—track timestamps to handle this, (3) You won't know when scrubbing ends and sequential rendering begins, (4) If you set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING, the entire buffer is saved in the project file. Modern After Effects caches frames internally, so re-requesting past frames often retrieves cached data rather than forcing re-renders. The sequence_data lifecycle includes: PF_Cmd_SEQUENCE_SETUP (new instance), PF_Cmd_SEQUENCE_RESETUP (project load/paste/save), PF_Cmd_SEQUENCE_FLATTEN (save/copy preparation), and PF_Cmd_SEQUENCE_SETDOWN (cleanup).

```cpp
if (in_data->sequence_data == NULL) {
  // Allocate memory
  in_data->sequence_data = PF_NEW_HANDLE(buffer_size);
} else {
  // Memory already allocated, reuse it
}
```

*Tags: `sequence-data`, `memory`, `caching`, `aegp`, `render-loop`*

---

## How can I render only a single color channel (R, G, B, or A) from a PF_EffectWorld?

There are several approaches to isolate and display a single channel from a PF_EffectWorld without subpixel sampling. The iteration suite is recommended as it is multi-threaded and easy to manage—you iterate through input and output buffers and selectively zero out unwanted channels while preserving the desired one. Alternatively, you can manually access the world's base address and directly manipulate individual PF_Plane channels in memory for maximum efficiency, though this sacrifices multi-threading support. A compositing approach can also work: create a new world filled with the desired color and composite it using composite_rect with multiply transfer mode to suppress other channels. If using SmartFX, you can declare which channels to use and let AE ignore the others, though this approach was noted as potentially risky.

```cpp
outP->alpha = inP->alpha;
outP->red = 0;
outP->green = 0;
outP->blue = inP->blue;
```

*Tags: `aegp`, `render-loop`, `memory`, `smartfx`*

---

## How can I control layer copy-paste operations in an After Effects Effect plugin to restrict pasting within the same composition?

Use an AEGP (After Effects General Plugin) rather than an Effect plugin to intercept copy-paste operations. Register a command hook using AEGP_RegisterCommandHook to be notified whenever the user copies or pastes. This allows you to check the collection being pasted and remove or delete effects as needed. You can use AEGP_Command_ALL with AEGP_RegisterCommandHook to discover the command numbers for copy and paste operations. Attempting this from within an Effect plugin alone is not practical.

*Tags: `aegp`, `pipl`, `layer-checkout`, `scripting`*

---

## Can I draw custom graphics and keyframe indicators on the After Effects TimeGraph?

No, the After Effects SDK does not provide buffers or APIs for drawing on the TimeGraph or timeline. You can only draw into buffers that AE provides for the composition window, layer window, and effects controls window. Custom parameter UIs created in the Effects Controls Window will not appear in the timeline. As an alternative, you can create a custom UI using an arbitrary data type parameter that displays a synchronized alternative timeline view centered on the current time cursor, allowing users to edit your special keyframes with custom interpolation behavior.

*Tags: `ui`, `params`, `arb-data`, `aegp`, `sdk`*

---
