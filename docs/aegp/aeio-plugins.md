# AEIO Plugins: Custom File Format Import/Export

AEIO (After Effects Input/Output) plugins allow you to add support for custom file formats to After Effects. Unlike effect plugins, AEIOs are registered as AEGP plugins and provide a function block that AE calls to import footage, export rendered output, or both. This document covers the complete lifecycle of an AEIO plugin, from registration through frame delivery and export.

## Architecture Overview

An AEIO is an AEGP plugin that calls `AEGP_RegisterIO` during its entry point. It provides a filled-in `AEIO_ModuleInfo` structure describing what it supports, and a function block (`AEIO_FunctionBlock4`) containing callbacks for every operation AE may need.

```
AEGP Entry Point
  -> Fill AEIO_ModuleInfo (capabilities, file types, extensions)
  -> Fill AEIO_FunctionBlock4 (all callback functions)
  -> Call AEGP_RegisterIO(plugin_id, refcon, &module_info, &function_block)
```

AE does not use PiPL for AEIO registration. Instead, the AEGP entry point is discovered via PiPL as any AEGP, and the AEIO module itself is registered programmatically at runtime.

## Registration

### AEGP_RegisterIO

Registration happens during the AEGP entry point function, typically in response to the plugin being loaded:

```cpp
A_Err entryFunc(
    struct SPBasicSuite *pica_basicP,
    A_long major_versionL,
    A_long minor_versionL,
    AEGP_PluginID aegp_plugin_id,
    AEGP_GlobalRefcon *global_refconP)
{
    A_Err err = A_Err_NONE;

    AEIO_ModuleInfo info;
    AEIO_FunctionBlock4 funcs;

    AEFX_CLR_STRUCT(info);
    AEFX_CLR_STRUCT(funcs);

    // Fill in module info and function block...

    AEGP_SuiteHandler suites(pica_basicP);
    ERR(suites.RegisterSuite5()->AEGP_RegisterIO(
        aegp_plugin_id,
        NULL,           // refcon - passed to all callbacks
        &info,
        &funcs));

    return err;
}
```

The `AEGP_RegisterSuite` function signature:

```cpp
SPAPI A_Err (*AEGP_RegisterIO)(
    AEGP_PluginID           aegp_plugin_id,     // your plugin ID
    AEGP_IORefcon           aegp_refconP,       // passed to all callbacks as basic_dataP->aegp_refconPV
    const AEIO_ModuleInfo   *io_infoP,          // module capabilities
    const AEIO_FunctionBlock4 *aeio_fcn_blockP  // function pointers
);
```

## AEIO_ModuleInfo Structure

This structure tells AE everything about your module's capabilities. It is defined in `AE_IO.h`:

```cpp
typedef struct {
    AEIO_ModuleSignature    sig;                            // unique 4-char code
    A_char                  name[AEIO_MAX_MODULE_NAME_LEN+1]; // display name (31 chars max)
    AEIO_ModuleFlags        flags;                          // capability flags
    AEIO_ModuleFlags2       flags2;                         // extended flags
    A_long                  max_width;                      // maximum supported width
    A_long                  max_height;                     // maximum supported height
    A_short                 num_filetypes;                  // mac file type/creator pairs
    A_short                 num_extensions;                 // number of file extensions
    A_short                 num_clips;                      // clipboard types
    A_short                 pad;
    PFILE_FileKind          create_kind;                    // type/creator for new files
    AEIO_FileExt            create_ext;                     // extension for new files
    AEIO_FileKind           read_kinds[AEIO_MAX_TYPES];     // supported read types (max 16)
    A_short                 num_aux_extensionsS;
    AEIO_AuxExt             aux_ext[AEIO_MAX_AUX_EXT];     // auxiliary extensions (max 16)
} AEIO_ModuleInfo;
```

### File Type and Extension Registration

The `read_kinds` array holds up to `AEIO_MAX_TYPES` (16) entries. Mac file type/creator pairs come first, then DOS/Windows extensions, then clipboard types. The counts `num_filetypes`, `num_extensions`, and `num_clips` tell AE how to interpret the array.

```cpp
// Extension registration example
AEIO_FileExt ext;
ext.pad = '.';
ext.extension[0] = 'X';
ext.extension[1] = 'Y';
ext.extension[2] = 'Z';

info.read_kinds[0].ext = ext;   // .XYZ extension
info.num_extensions = 1;
info.num_filetypes = 0;
info.num_clips = 0;

// For the file AE creates on output
info.create_ext = ext;
```

Auxiliary extensions can be specified using `aux_ext` for sidecar files (e.g., `.xmp`, `.wav` alongside a video file).

### Module Flags

The primary flags (`AEIO_ModuleFlags`) define the core capabilities:

| Flag | Value | Description |
|------|-------|-------------|
| `AEIO_MFlag_INPUT` | `1<<0` | Module can import files |
| `AEIO_MFlag_OUTPUT` | `1<<1` | Module can export files |
| `AEIO_MFlag_FILE` | `1<<2` | Direct correspondence to a file on disk |
| `AEIO_MFlag_STILL` | `1<<3` | Supports still images |
| `AEIO_MFlag_VIDEO` | `1<<4` | Supports video (multiple frames) |
| `AEIO_MFlag_AUDIO` | `1<<5` | Supports audio |
| `AEIO_MFlag_NO_TIME` | `1<<6` | Time-independent (always true for stills) |
| `AEIO_MFlag_INTERACTIVE_GET` | `1<<7` | User interaction for import (required if not FILE and INPUT) |
| `AEIO_MFlag_INTERACTIVE_PUT` | `1<<8` | User interaction for export (required if not FILE and OUTPUT) |
| `AEIO_MFlag_CANT_CLIP` | `1<<9` | DrawFrame cannot handle worlds smaller than full dimensions |
| `AEIO_MFlag_MUST_INTERACT_PUT` | `1<<10` | Dialog cannot be skipped even with existing options |
| `AEIO_MFlag_CANT_SOUND_INTERLEAVE` | `1<<11` | All frames must be added before sound |
| `AEIO_MFlag_CAN_ADD_FRAMES_NON_LINEAR` | `1<<12` | AddFrame can handle non-sequential times |
| `AEIO_MFlag_HOST_DEPTH_DIALOG` | `1<<13` | AE provides depth selection dialog |
| `AEIO_MFlag_HOST_FRAME_START_DIALOG` | `1<<14` | AE provides starting frame dialog |
| `AEIO_MFlag_NO_OPTIONS` | `1<<16` | Module has no output options |
| `AEIO_MFlag_NO_PIXELS` | `1<<19` | Format stores no real pixels (geometry only) |
| `AEIO_MFlag_SEQUENCE_OPTIONS_OK` | `1<<20` | Inherits parent sequence options for folder selection |
| `AEIO_MFlag_INPUT_OPTIONS` | `1<<21` | Module has per-input user options (must be flat data) |
| `AEIO_MFlag_HSF_AWARE` | `1<<22` | Module sets HSF (horizontal scale factor) for imports |
| `AEIO_MFlag_HAS_LAYERS` | `1<<23` | Supports multiple layers per document |
| `AEIO_MFlag_SCRAP` | `1<<24` | Has clipboard parsing component |
| `AEIO_MFlag_NO_UI` | `1<<25` | No UI shown for this module |
| `AEIO_MFlag_SEQ_OPTIONS_DLG` | `1<<26` | Module has sequence options dialog |
| `AEIO_MFlag_HAS_AUX_DATA` | `1<<27` | Has auxiliary per-pixel data (depth, normals) |
| `AEIO_MFlag_HAS_META_DATA` | `1<<28` | Supports user-definable metadata |
| `AEIO_MFlag_CAN_DO_MARKERS` | `1<<29` | Supports markers (URL flips, chapters) |
| `AEIO_MFlag_CAN_DRAW_DEEP` | `1<<30` | Can draw into 16-bpc (64 bpp) worlds |

### Module Flags 2

Extended flags (`AEIO_ModuleFlags2`):

| Flag | Value | Description |
|------|-------|-------------|
| `AEIO_MFlag2_AUDIO_OPTIONS` | `1<<0` | Has audio output options |
| `AEIO_MFlag2_SEND_ADDMARKER_BEFORE_ADDFRAME` | `1<<2` | Send markers before the corresponding frame |
| `AEIO_MFlag2_CAN_DO_MARKERS_2` | `1<<3` | Supports combined markers |
| `AEIO_MFlag2_CAN_DRAW_FLOAT` | `1<<4` | Can draw into 32-bpc float worlds |
| `AEIO_MFlag2_CAN_DO_AUDIO_32` | `1<<6` | Supports 32-bit float audio output |
| `AEIO_MFlag2_SUPPORTS_ICC_PROFILES` | `1<<8` | Supports ICC color profiles |
| `AEIO_MFlag2_CAN_DO_MARKERS_3` | `1<<9` | Supports cue point markers |
| `AEIO_MFlag2_SEND_ADDMARKER_BEFORE_STARTADDING` | `1<<10` | Send markers before StartAdding |

## The AEIO Function Block

The `AEIO_FunctionBlock4` (frozen since AE 10) contains all callback function pointers. Every function receives an `AEIO_BasicData` pointer as its first argument:

```cpp
typedef struct AEIO_InData {
    AEIO_MessageFunc          msg_func;       // for displaying error messages
    const struct SPBasicSuite *pica_basicP;   // for acquiring PICA suites
    A_long                    aegp_plug_id;   // your registered plugin ID
    void                      *aegp_refconPV; // your refcon from registration
} AEIO_BasicData;
```

### Import Callbacks

#### AEIO_InitInSpecFromFile

Called when AE opens a file with a matching extension. This is where you read the file header and set all import properties.

```cpp
A_Err MyInitInSpecFromFile(
    AEIO_BasicData    *basic_dataP,
    const A_UTF16Char *file_pathZ,    // null-terminated UTF-16 path
    AEIO_InSpecH      inH)
{
    A_Err err = A_Err_NONE;
    AEGP_SuiteHandler suites(basic_dataP->pica_basicP);

    // Open and parse the file using standard C/C++ I/O
    // (The SDK does not provide file I/O functions)

    // Set dimensions
    ERR(suites.IOInSuite6()->AEGP_SetInSpecDimensions(inH, width, height));

    // Set bit depth (see AEIO_InputDepth enum)
    ERR(suites.IOInSuite6()->AEGP_SetInSpecDepth(inH, 32)); // 32 = 8bpc RGBA

    // Set frame rate as A_Fixed (16.16 fixed point)
    // 29.97 fps = (2997 << 16) / 100
    A_Fixed fps = (A_Fixed)(framerate * 65536.0);
    ERR(suites.IOInSuite6()->AEGP_SetInSpecNativeFPS(inH, fps));

    // Set duration
    A_Time duration;
    duration.value = num_frames;
    duration.scale = (A_u_long)(framerate + 0.5);
    ERR(suites.IOInSuite6()->AEGP_SetInSpecDuration(inH, &duration));

    // Set alpha interpretation
    AEIO_AlphaLabel alpha;
    alpha.version = AEIO_AlphaLabel_VERSION;
    alpha.alpha = AEIO_Alpha_PREMUL;  // or STRAIGHT, IGNORE, NONE
    alpha.flags = AEIO_AlphaPremul;
    alpha.red = alpha.green = alpha.blue = 0; // premul matte color
    ERR(suites.IOInSuite6()->AEGP_SetInSpecAlphaLabel(inH, &alpha));

    // Set file size for display
    ERR(suites.IOInSuite6()->AEGP_SetInSpecSize(inH, file_size_bytes));

    // Store custom options handle if needed
    // (must be flat data if AEIO_MFlag_INPUT_OPTIONS is set)

    return err;
}
```

#### AEIO_DrawSparseFrame

This is the core import callback. AE calls it when it needs a frame rendered into a provided `PF_EffectWorld`. See the dedicated [Frame Caching](aeio-frame-caching.md) document for details on when and how this is called.

```cpp
A_Err MyDrawSparseFrame(
    AEIO_BasicData                *basic_dataP,
    AEIO_InSpecH                  inH,
    const AEIO_DrawSparseFramePB  *sparse_framePPB,
    PF_EffectWorld                *worldP,
    AEIO_DrawingFlags             *draw_flagsP)
{
    A_Err err = A_Err_NONE;

    // sparse_framePPB contains:
    //   .qual          - AEIO_Qual_LOW or AEIO_Qual_HIGH
    //   .rs            - scale factor (rational x/y)
    //   .tr            - requested time
    //   .duration      - frame duration
    //   .required_region - sub-region needed (empty = entire frame)
    //   .inter         - abort/progress callbacks

    // Decode the frame at the requested time
    A_long frame_num = /* compute from sparse_framePPB->tr */;

    // Write pixels into worldP
    // IMPORTANT: respect worldP->rowbytes for stride
    for (A_long y = 0; y < worldP->height; y++) {
        PF_Pixel8 *rowP = (PF_Pixel8 *)((char *)worldP->data + y * worldP->rowbytes);
        for (A_long x = 0; x < worldP->width; x++) {
            rowP[x].alpha = 255;
            rowP[x].red   = /* decoded red */;
            rowP[x].green = /* decoded green */;
            rowP[x].blue  = /* decoded blue */;
        }
    }

    // Set drawing flags to indicate what processing you did
    *draw_flagsP = AEIO_DFlags_NONE;
    // *draw_flagsP |= AEIO_DFlags_DID_DEINT;      // if you deinterlaced
    // *draw_flagsP |= AEIO_DFlags_DID_ALPHA_CONV;  // if you handled alpha

    return err;
}
```

#### AEIO_GetSound

Called when AE needs audio data from an imported file:

```cpp
A_Err MyGetSound(
    AEIO_BasicData              *basic_dataP,
    AEIO_InSpecH                inH,
    AEIO_SndQuality             quality,        // APPROX, LO, or HI
    const AEIO_InterruptFuncs   *interrupt_funcsP0,
    const A_Time                *startPT,       // start time
    const A_Time                *durPT,         // duration requested
    A_u_long                    start_sampLu,   // starting sample
    A_u_long                    num_samplesLu,  // number of samples
    void                        *dataPV)        // output buffer
{
    // Fill dataPV with interleaved PCM audio samples
    // Format must match what you declared via AEGP_SetInSpecSoundEncoding,
    // AEGP_SetInSpecSoundSampleSize, and AEGP_SetInSpecSoundChannels
    return A_Err_NONE;
}
```

Audio properties are set during `InitInSpecFromFile`:

```cpp
// 48kHz, signed 16-bit PCM, stereo
ERR(suites.IOInSuite6()->AEGP_SetInSpecSoundRate(inH, 48000.0));
ERR(suites.IOInSuite6()->AEGP_SetInSpecSoundEncoding(inH, AEIO_E_SIGNED_PCM));
ERR(suites.IOInSuite6()->AEGP_SetInSpecSoundSampleSize(inH, AEIO_SS_2)); // 2 bytes
ERR(suites.IOInSuite6()->AEGP_SetInSpecSoundChannels(inH, AEIO_SndChannels_STEREO));
```

Sound encoding constants:

| Constant | Value | Description |
|----------|-------|-------------|
| `AEIO_E_UNSIGNED_PCM` | 1 | Unsigned integer PCM |
| `AEIO_E_SIGNED_PCM` | 2 | Signed integer PCM |
| `AEIO_E_SIGNED_FLOAT` | 3 | 32-bit float PCM |

Sample size constants:

| Constant | Value | Bytes per sample |
|----------|-------|-----------------|
| `AEIO_SS_1` | 1 | 8-bit |
| `AEIO_SS_2` | 2 | 16-bit |
| `AEIO_SS_4` | 4 | 32-bit |

#### Other Import Callbacks

| Callback | Purpose |
|----------|---------|
| `AEIO_InitInSpecInteractive` | Import via user interaction (no file path) |
| `AEIO_DisposeInSpec` | Clean up when import spec is disposed |
| `AEIO_FlattenOptions` | Flatten options handle for serialization |
| `AEIO_InflateOptions` | Restore options from flat data |
| `AEIO_SynchInSpec` | Re-synchronize spec when file changes |
| `AEIO_GetActiveExtent` | Return the active area at a given time |
| `AEIO_GetInSpecInfo` | Provide display strings (name, type, subtype) |
| `AEIO_GetDimensions` | Return dimensions at a given scale |
| `AEIO_GetDuration` | Return total duration |
| `AEIO_GetTime` | Return time base |
| `AEIO_InqNextFrameTime` | Find the next keyframe time (for variable frame rate) |
| `AEIO_VerifyFileImportable` | Quick check whether a file can be imported |
| `AEIO_SeqOptionsDlg` | Show sequence options dialog for import |
| `AEIO_CloseSourceFiles` | Close file handles (called when AE needs to free resources) |

### Export Callbacks

#### The Export Lifecycle

Export follows a strict sequence:

```
1. AEIO_InitOutputSpec     - Initialize output settings
2. AEIO_UserOptionsDialog  - (optional) Show codec/format settings
3. AEIO_SetOutputFile      - Receive the output file path
4. AEIO_StartAdding        - Begin the render session
5. AEIO_AddFrame           - Called once per frame (repeated)
   AEIO_AddSoundChunk      - Called for audio chunks (interleaved or after video)
   AEIO_AddMarker/2/3      - Called for markers
6. AEIO_EndAdding          - Finalize and close the file
```

For still image output, `AEIO_OutputFrame` is called instead of the StartAdding/AddFrame/EndAdding sequence.

#### AEIO_AddFrame

```cpp
A_Err MyAddFrame(
    AEIO_BasicData          *basic_dataP,
    AEIO_OutSpecH           outH,
    A_long                  frame_index,    // current frame number
    A_long                  frames,         // total frame count
    const PF_EffectWorld    *wP,            // rendered frame pixels
    const A_LPoint          *origin0,       // origin offset (can be NULL)
    A_Boolean               was_compressedB,
    AEIO_InterruptFuncs     *inter0)        // abort/progress callbacks
{
    // Read pixels from wP
    // wP->data points to pixel data
    // wP->rowbytes is the stride
    // wP->width, wP->height are dimensions
    // Pixel format depends on output depth setting

    // Encode and write the frame to your output file

    // Report progress if inter0 is not NULL
    if (inter0 && inter0->progress0) {
        inter0->progress0(inter0->refcon, frame_index, frames);
    }

    return A_Err_NONE;
}
```

#### AEIO_UserOptionsDialog

Provides a format settings dialog when the user clicks "Format Options" in the Output Module:

```cpp
A_Err MyUserOptionsDialog(
    AEIO_BasicData        *basic_dataP,
    AEIO_OutSpecH         outH,
    const PF_EffectWorld  *sample0,          // thumbnail preview (can be NULL)
    A_Boolean             *user_interacted0) // set to TRUE if user changed settings
{
    AEGP_SuiteHandler suites(basic_dataP->pica_basicP);

    // Retrieve current options
    void *optionsPV = NULL;
    ERR(suites.IOOutSuite5()->AEGP_GetOutSpecOptionsHandle(outH, &optionsPV));

    MyOptions *opts = reinterpret_cast<MyOptions*>(optionsPV);

    // Show your dialog (platform-specific or via AEGP_ExecuteScript for JS)
    // ...

    *user_interacted0 = TRUE;
    return A_Err_NONE;
}
```

> **Cross-platform tip**: For options dialogs that must work on both Windows and macOS, consider using `AEGP_ExecuteScript()` to display a dialog via JavaScript/ExtendScript. This avoids maintaining separate native UI code for each platform.

#### AEIO_GetDepths

Reports which output bit depths your module supports:

```cpp
A_Err MyGetDepths(
    AEIO_BasicData             *basic_dataP,
    AEIO_OutSpecH              outH,
    AEIO_SupportedDepthFlags   *which)
{
    *which = AEIO_SupportedDepthFlags_DEPTH_24 |     // RGB 8bpc
             AEIO_SupportedDepthFlags_DEPTH_32 |     // RGBA 8bpc
             AEIO_SupportedDepthFlags_DEPTH_48 |     // RGB 16bpc
             AEIO_SupportedDepthFlags_DEPTH_64 |     // RGBA 16bpc
             AEIO_SupportedDepthFlags_DEPTH_96 |     // RGB 32-bit float
             AEIO_SupportedDepthFlags_DEPTH_128;     // RGBA 32-bit float
    return A_Err_NONE;
}
```

## AEGP_IOInSuite (Version 6)

This suite is used within import callbacks to get and set properties on the `AEIO_InSpecH`:

| Function | Description |
|----------|-------------|
| `AEGP_GetInSpecOptionsHandle` / `SetInSpecOptionsHandle` | Get/set custom options data |
| `AEGP_GetInSpecFilePath` | Get the file path (returns `AEGP_MemHandle` of UTF-16, must be freed) |
| `AEGP_GetInSpecNativeFPS` / `SetInSpecNativeFPS` | Frame rate as `A_Fixed` (16.16 fixed point) |
| `AEGP_GetInSpecDepth` / `SetInSpecDepth` | Bit depth (`AEIO_InputDepth` values) |
| `AEGP_GetInSpecSize` / `SetInSpecSize` | File size in bytes |
| `AEGP_GetInSpecInterlaceLabel` / `SetInSpecInterlaceLabel` | Field order |
| `AEGP_GetInSpecAlphaLabel` / `SetInSpecAlphaLabel` | Alpha interpretation |
| `AEGP_GetInSpecDuration` / `SetInSpecDuration` | Total duration as `A_Time` |
| `AEGP_GetInSpecDimensions` / `SetInSpecDimensions` | Width and height |
| `AEGP_GetInSpecHSF` / `SetInSpecHSF` | Horizontal scale factor (pixel aspect ratio) |
| `AEGP_GetInSpecSoundRate` / `SetInSpecSoundRate` | Audio sample rate |
| `AEGP_GetInSpecSoundEncoding` / `SetInSpecSoundEncoding` | Audio encoding type |
| `AEGP_GetInSpecSoundSampleSize` / `SetInSpecSoundSampleSize` | Bytes per audio sample |
| `AEGP_GetInSpecSoundChannels` / `SetInSpecSoundChannels` | Mono or stereo |
| `AEGP_SetInSpecEmbeddedColorProfile` | Set embedded ICC profile or color space description |
| `AEGP_SetInSpecAssignedColorProfile` | Assign an RGB profile to footage |
| `AEGP_SetInSpecNativeStartTime` / `ClearInSpecNativeStartTime` | Native start timecode |
| `AEGP_SetInSpecNativeDisplayDropFrame` | Drop-frame timecode flag |
| `AEGP_SetInSpecStillSequenceNativeFPS` | FPS for still sequences |
| `AEGP_SetInSpecColorSpaceFromCICP` | Set color space from ITU-T H.273 CICP codes (AE 25+) |
| `AEGP_AddAuxExtMap` | Register auxiliary file extensions |

### Input Depth Values

The `AEIO_InputDepth` enum defines valid depth values for `SetInSpecDepth`:

| Constant | Value | Description |
|----------|-------|-------------|
| `AEIO_InputDepth_1` | 1 | 1-bit |
| `AEIO_InputDepth_8` | 8 | 8-bit grayscale or indexed |
| `AEIO_InputDepth_16` | 16 | 16-bit (RGB 5-5-5 + alpha) |
| `AEIO_InputDepth_24` | 24 | 24-bit RGB |
| `AEIO_InputDepth_32` | 32 | 32-bit RGBA (8 bpc) |
| `AEIO_InputDepth_48` | 48 | 48-bit RGB (16 bpc) |
| `AEIO_InputDepth_64` | 64 | 64-bit RGBA (16 bpc) |
| `AEIO_InputDepth_96` | 96 | RGB float (32 bpc) |
| `AEIO_InputDepth_128` | 128 | RGBA float (32 bpc) |
| `AEIO_InputDepth_GRAY_8` | 40 | 8-bit grayscale |
| `AEIO_InputDepth_GRAY_16` | -16 | 16-bit grayscale |
| `AEIO_InputDepth_GRAY_32` | -32 | 32-bit float grayscale |

## AEGP_IOOutSuite (Version 5)

Used within export callbacks to query and modify the `AEIO_OutSpecH`:

| Function | Description |
|----------|-------------|
| `AEGP_GetOutSpecOptionsHandle` / `SetOutSpecOptionsHandle` | Custom output options data |
| `AEGP_GetOutSpecFilePath` | Output file path (check `file_reservedPB` for render farm) |
| `AEGP_GetOutSpecFPS` / `SetOutSpecNativeFPS` | Frame rate |
| `AEGP_GetOutSpecDepth` / `SetOutSpecDepth` | Bit depth |
| `AEGP_GetOutSpecInterlaceLabel` / `SetOutSpecInterlaceLabel` | Field order |
| `AEGP_GetOutSpecAlphaLabel` / `SetOutSpecAlphaLabel` | Alpha handling |
| `AEGP_GetOutSpecDuration` / `SetOutSpecDuration` | Total duration |
| `AEGP_GetOutSpecDimensions` | Output dimensions |
| `AEGP_GetOutSpecHSF` / `SetOutSpecHSF` | Pixel aspect ratio |
| `AEGP_GetOutSpecSoundRate` / `SetOutSpecSoundRate` | Audio sample rate |
| `AEGP_GetOutSpecSoundEncoding` / `SetOutSpecSoundEncoding` | Audio format |
| `AEGP_GetOutSpecSoundSampleSize` / `SetOutSpecSoundSampleSize` | Audio sample size |
| `AEGP_GetOutSpecSoundChannels` / `SetOutSpecSoundChannels` | Audio channels |
| `AEGP_GetOutSpecIsStill` | Whether output is a single frame |
| `AEGP_GetOutSpecPosterTime` | Poster frame time |
| `AEGP_GetOutSpecStartFrame` | Starting frame number |
| `AEGP_GetOutSpecPullDown` | Pulldown phase |
| `AEGP_GetOutSpecIsMissing` | Whether output file is missing |
| `AEGP_GetOutSpecShouldEmbedICCProfile` | Whether to embed color profile |
| `AEGP_GetNewOutSpecColorProfile` | Get output color profile (must dispose) |
| `AEGP_GetOutSpecOutputModule` | Get render queue item and output module refs |
| `AEGP_GetOutSpecStartTime` | Output start time |
| `AEGP_GetOutSpecFrameTime` | Time per frame (relative to start) |
| `AEGP_GetOutSpecIsDropFrame` | Drop-frame timecode flag |

## InSpec and OutSpec Lifecycle

### Import (InSpec) Lifecycle

```
File dropped/imported
  -> AEIO_VerifyFileImportable    (quick sanity check)
  -> AEIO_InitInSpecFromFile      (parse file, set all properties)
  -> [Optional] AEIO_SeqOptionsDlg (user sequence options)
  -> AEIO_GetInSpecInfo            (display strings for Project panel)
  ...
  -> AEIO_DrawSparseFrame          (called each time a frame is needed)
  -> AEIO_GetSound                 (called when audio is needed)
  ...
  -> AEIO_SynchInSpec              (if file changes on disk)
  -> AEIO_CloseSourceFiles         (when AE needs file handles back)
  ...
  -> AEIO_DisposeInSpec            (footage removed from project)
```

### Export (OutSpec) Lifecycle

```
Output module selected
  -> AEIO_InitOutputSpec           (set defaults)
  -> AEIO_UserOptionsDialog        (format settings)
  -> AEIO_GetOutputInfo            (display strings for Output Module)
  ...
Render begins:
  -> AEIO_SetOutputFile            (receive file path)
  -> AEIO_StartAdding              (open file, write header)
  -> AEIO_AddFrame * N             (write each frame)
  -> AEIO_AddSoundChunk * N        (write audio)
  -> AEIO_EndAdding                (finalize, close file)
```

## The AEIO_Verbiage Structure

Both import and export info callbacks fill this structure for display in the UI:

```cpp
typedef struct {
    A_char name[AEIO_MAX_SEQ_NAME_LEN+1];       // "my_video.xyz"
    A_char type[AEIO_MAX_SEQ_NAME_LEN+1];       // "XYZ Movie"
    A_char sub_type[AEIO_MAX_MESSAGE_LEN+1];     // "H.265, 1920x1080, 24fps"
} AEIO_Verbiage;
```

## Color Management

For RGB source data with an embedded ICC profile:

```cpp
// Build a color profile from ICC data
AEGP_ColorProfileP profileP = NULL;
ERR(suites.ColorSettingsSuite2()->AEGP_GetNewColorProfileFromICCProfile(
    basic_dataP->aegp_plug_id, icc_size, icc_data, &profileP));

// Set the embedded profile (pass NULL for description)
ERR(suites.IOInSuite6()->AEGP_SetInSpecEmbeddedColorProfile(
    inH, profileP, NULL));

// Dispose the profile
ERR(suites.ColorSettingsSuite2()->AEGP_DisposeColorProfile(profileP));
```

For non-RGB source data (e.g., YUV), pass a description string instead:

```cpp
// Describe the color space (disables AE color management UI for this item)
A_UTF16Char desc[] = u"BT.709 YCbCr";
ERR(suites.IOInSuite6()->AEGP_SetInSpecEmbeddedColorProfile(
    inH, NULL, desc));
```

As of AE 25, you can also use CICP codes from ITU-T H.273:

```cpp
ERR(suites.IOInSuite6()->AEGP_SetInSpecColorSpaceFromCICP(
    inH,
    1,      // Color primaries (1 = BT.709)
    1,      // Transfer characteristics (1 = BT.709)
    1,      // Matrix coefficients (1 = BT.709)
    0,      // Full range flag (0 = limited, 1 = full)
    8,      // Bit depth
    FALSE   // Is RGB
));
```

## Common Pitfalls

**Cannot override built-in formats.** AE ignores custom AEIOs that register for file types it already handles natively. If you need to handle a format AE already supports, use a unique file extension.

**Options handles must be flat.** If you set `AEIO_MFlag_INPUT_OPTIONS`, the options data stored on `InSpecH` must be plain flat data (no pointers, no handles inside). AE serializes this data directly to disk.

**DrawSparseFrame world size may differ from source dimensions.** When AE requests a thumbnail or scaled preview, the provided `PF_EffectWorld` may be smaller than the native resolution. Always use `worldP->width` and `worldP->height`, not the source dimensions.

**Row bytes are not width * bytes_per_pixel.** Always use `worldP->rowbytes` for pixel stride. The buffer may have padding bytes at the end of each row.

**File I/O is your responsibility.** The SDK provides no file reading/writing functions. Use standard C/C++ I/O (`fopen`, `fread`, `std::ifstream`, etc.) or platform APIs. File paths arrive as `A_UTF16Char` (null-terminated UTF-16 strings).

**Coordinate order matters.** When iterating pixels manually (e.g., with `sampleIntegral32`), ensure coordinates are passed in the correct `(x, y)` order. Swapped coordinates cause crashes or garbage output.

**Marker ordering.** By default, `AEIO_AddMarker` is called just after the corresponding `AEIO_AddFrame`. Set `AEIO_MFlag2_SEND_ADDMARKER_BEFORE_ADDFRAME` if your format needs markers before the frame data.

**Render farm file reservation.** When `AEGP_GetOutSpecFilePath` returns `file_reservedPB == TRUE`, an empty placeholder file has already been created for multi-machine rendering. Do not create the file yourself in this case, or render farm coordination will fail.
