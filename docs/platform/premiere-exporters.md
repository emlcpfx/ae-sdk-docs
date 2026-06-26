# Premiere Pro Exporters: Video/Audio Export Plugins

This document covers the Premiere Pro exporter plugin architecture, referencing the SDK 26.0 headers (`PrSDKExport.h`, `PrSDKExportParamSuite.h`, `PrSDKExportInfoSuite.h`, `PrSDKExportFileSuite.h`, `PrSDKExportProgressSuite.h`, `PrSDKExportStdParamSuite.h`). Exporters are standalone plugin types that render timeline content to output files. Unlike AE's `AEIO` plugins, Premiere exporters are a distinct module type with their own entry point, parameter UI system, and rendering pipeline.

## Entry Point

An exporter's entry point must be named `xSDKExport`:

```cpp
// PrSDKExport.h
typedef PREMPLUGENTRY (*ExportEntryFunc)(
    csSDK_int32     selector,
    exportStdParms* stdparms,
    void*           param1,
    void*           param2);

#define SDKExportEntryPointName "xSDKExport"
```

The `exportStdParms` provides the interface version and access to the suite mechanism:

```cpp
typedef struct {
    csSDK_int32                 interfaceVer;      // EXPORTMOD_VERSION
    plugGetSPBasicSuiteFunc     getSPBasicSuite;   // Returns SPBasicSuite*
} exportStdParms;
```

The current interface version is `EXPORTMOD_VERSION` = `prExportVersion800` (value 10).

---

## Complete Selector List

Selectors are defined in `enum PrExportSelector`:

| Selector | Description |
|---|---|
| `exSelStartup` | Register exporter. Fill `exExporterInfoRec` with file type, name, and capabilities. Return `exportReturn_IterateExporter` for multi-format exporters. |
| `exSelShutdown` | Release global resources before unload. |
| `exSelBeginInstance` | Create instance-specific private data. Receives `exExporterInstanceRec`. |
| `exSelEndInstance` | Dispose of instance private data. |
| `exSelGenerateDefaultParams` | Create the default parameter tree. Receives `exGenerateDefaultParamRec`. |
| `exSelPostProcessParams` | Validate loaded params, fill localized names, populate constrained value lists. |
| `exSelValidateParamChanged` | Respond to a single parameter change. Hide/show/enable/disable related params. |
| `exSelGetParamSummary` | Generate human-readable summary text (video, audio, bitrate lines). |
| `exSelParamButton` | Handle a button click in the parameter UI. |
| `exSelExport` | Perform the actual export. Receives `exDoExportRec`. |
| `exSelExport2` | Extended export selector. Receives `exDoExportRec2` (adds LUT support). |
| `exSelQueryExportFileExtension` | Override the file extension dynamically. |
| `exSelQueryOutputFileList` | Report all output files (for multi-file exports). |
| `exSelQueryStillSequence` | For still-only exporters: export as image sequence? |
| `exSelQueryOutputSettings` | Report current output settings (resolution, fps, bitrate, etc.). Required. |
| `exSelValidateOutputSettings` | Validate current settings. Return error string if invalid. |
| `exSelStartupLicense` | Secondary startup call for license verification. |
| `exSelQueryExportColorSpace` | Report the color space for rendered frames. |

---

## exExporterInfoRec: Startup Registration

During `exSelStartup`, fill `exExporterInfoRec` to declare your exporter:

```cpp
typedef struct {
    csSDK_uint32    fileType;                       // Unique fourCC identifier
    prUTF16Char     fileTypeName[256];              // Display name
    prUTF16Char     fileTypeDefaultExtension[256];  // Default extension
    csSDK_uint32    classID;                        // Differentiates exporters with same fileType
    csSDK_int32     exportReqIndex;                 // For multi-format exporter iteration
    csSDK_int32     wantsNoProgressBar;             // Show your own progress UI
    csSDK_int32     hideInUI;                       // Preview-only exporter
    csSDK_int32     doesNotSupportAudioOnly;
    csSDK_int32     canExportVideo;
    csSDK_int32     canExportAudio;
    csSDK_int32     singleFrameOnly;                // Stills exporter
    csSDK_int32     maxAudiences;
    csSDK_int32     interfaceVersion;               // EXPORTMOD_VERSION
    csSDK_uint32    isCacheable;
    csSDK_uint32    canEmbedCaptions;
    csSDK_uint64    flags;                          // exInfoRecFlags
    csSDK_uint32    canEmbedColorProfile;
    csSDK_int32     supportsColorManagement;
} exExporterInfoRec;
```

### Example: exSelStartup

```cpp
case exSelStartup:
{
    exExporterInfoRec* infoRec = reinterpret_cast<exExporterInfoRec*>(param1);

    infoRec->fileType = 'SDK_';
    copyConvertStringLiteralIntoUTF16(L"SDK Sample Export", infoRec->fileTypeName);
    copyConvertStringLiteralIntoUTF16(L"sdk", infoRec->fileTypeDefaultExtension);
    infoRec->classID = 'SDK_';
    infoRec->interfaceVersion = EXPORTMOD_VERSION;
    infoRec->canExportVideo = kPrTrue;
    infoRec->canExportAudio = kPrTrue;
    infoRec->singleFrameOnly = kPrFalse;
    infoRec->maxAudiences = 1;
    infoRec->isCacheable = kPrFalse;
    infoRec->supportsColorManagement = kPrTrue;

    return exportReturn_ErrNone;
}
```

### Multi-Format Exporters

If a single plugin provides multiple export formats, return `exportReturn_IterateExporter` from `exSelStartup` for each format except the last. Return `exportReturn_IterateExporterDone` after the final format. The `exportReqIndex` field increments on each call.

---

## Instance Lifecycle

```
exSelStartup         --> Register file type(s)
exSelBeginInstance   --> Create private data for this export session
exSelGenerateDefaultParams --> Build parameter tree
exSelPostProcessParams --> Localize names, validate loaded presets
  [User edits params]
exSelValidateParamChanged  --> React to changes (repeated)
exSelGetParamSummary       --> Update summary text (repeated)
exSelExport / exSelExport2 --> Perform the export
exSelEndInstance     --> Clean up instance data
exSelShutdown        --> Global cleanup
```

### exSelBeginInstance / exSelEndInstance

```cpp
case exSelBeginInstance:
{
    exExporterInstanceRec* instRec = reinterpret_cast<exExporterInstanceRec*>(param1);
    MyExportData* myData = new MyExportData();
    myData->exporterPluginID = instRec->exporterPluginID;
    instRec->privateData = myData;
    return exportReturn_ErrNone;
}

case exSelEndInstance:
{
    exExporterInstanceRec* instRec = reinterpret_cast<exExporterInstanceRec*>(param1);
    MyExportData* myData = reinterpret_cast<MyExportData*>(instRec->privateData);
    delete myData;
    return exportReturn_ErrNone;
}
```

---

## Export Parameters

Premiere provides a rich parameter UI system through `PrSDKExportParamSuite`. Parameters are organized in a hierarchy:

```
MultiGroup (top-level container, at least one required)
  Group (visible tab, e.g., "Video", "Audio")
    Group (section within a tab)
      Param (the actual control)
```

### Parameter Types

From `exParamType`:

| Type | Description |
|---|---|
| `exParamType_bool` | Boolean checkbox |
| `exParamType_int` | Integer value |
| `exParamType_float` | Double-precision float |
| `exParamType_ticksFrameRate` | PrTime ticks for frame rate |
| `exParamType_ticksTime` | PrTime ticks for timecode display |
| `exParamType_string` | UTF-16 string |
| `exParamType_button` | Button with opaque data |
| `exParamType_arbitrary` | Opaque binary blob (no UI) |
| `exParamType_group` | Group container |
| `exParamType_multiGroup` | Top-level audience container |
| `exParamType_ratio` | Two int32 values (aspect ratio, 2D point) |
| `exParamType_thumbnail` | Thumbnail display |

### Parameter Flags

From `exParamFlags`:

| Flag | Value | Description |
|---|---|---|
| `exParamFlag_slider` | 0x01 | Show slider for int/float |
| `exParamFlag_independant` | 0x02 | No validation needed on change |
| `exParamFlag_shared` | 0x04 | Shared across multigroups |
| `exParamFlag_multiLine` | 0x08 | Multi-line text (string only) |
| `exParamFlag_password` | 0x10 | Hidden text (string only) |
| `exParamFlag_filePath` | 0x20 | File path with Browse button |
| `exParamFlag_optional` | 0x40 | Attached checkbox, disabled by default |
| `exParamFlag_verticalAlignment` | 0x80 | Label above control instead of beside |
| `exParamFlag_nonlinear` | 0x100 | Non-linear slider mapping |
| `exParamFlag_actionButton` | 0x200 | Action-styled button |

### Standard Parameter Identifiers

The SDK defines standard identifier strings that Premiere recognizes:

```cpp
// Video
#define ADBEVideoWidth        "ADBEVideoWidth"
#define ADBEVideoHeight       "ADBEVideoHeight"
#define ADBEVideoFPS          "ADBEVideoFPS"
#define ADBEVideoFieldType    "ADBEVideoFieldType"
#define ADBEVideoCodec        "ADBEVideoCodec"
#define ADBEVideoAspect       "ADBEVideoAspect"
#define ADBEVideoQuality      "ADBEVideoQuality"

// Audio
#define ADBEAudioRatePerSecond   "ADBEAudioRatePerSecond"
#define ADBEAudioSampleType      "ADBEAudioSampleType"
#define ADBEAudioCodec           "ADBEAudioCodec"
#define ADBEAudioChannelConfiguration "ADBEAudioChannelConfiguration"

// Bitrate
#define ADBEVideoTargetBitrate   "ADBEVideoTargetBitrate"
#define ADBEVideoMaxBitrate      "ADBEVideoMaxBitrate"
#define ADBEVideoBitrateEncoding "ADBEVideoBitrateEncoding"

// Groups
#define ADBEVideoTabGroup     "ADBEVideoTabGroup"
#define ADBEAudioTabGroup     "ADBEAudioTabGroup"
#define ADBETopParamGroup     "ADBETopParamGroup"
```

Using these identifiers allows Premiere to automatically match source settings and provide intelligent defaults.

### exSelGenerateDefaultParams Example

```cpp
case exSelGenerateDefaultParams:
{
    exGenerateDefaultParamRec* paramRec =
        reinterpret_cast<exGenerateDefaultParamRec*>(param1);
    csSDK_uint32 pluginID = paramRec->exporterPluginID;

    // Acquire the param suite
    PrSDKExportParamSuite* paramSuite = NULL;
    spBasic->AcquireSuite(kPrSDKExportParamSuite, kPrSDKExportParamSuiteVersion,
                          (const void**)&paramSuite);

    // Create a multigroup (at least one required)
    csSDK_int32 mgIndex = 0;
    paramSuite->AddMultiGroup(pluginID, &mgIndex);

    // Add video tab group
    paramSuite->AddParamGroup(pluginID, mgIndex,
        ADBETopParamGroup, ADBEVideoTabGroup,
        L"Video", kPrFalse, kPrFalse, kPrFalse);

    // Add basic video settings group
    paramSuite->AddParamGroup(pluginID, mgIndex,
        ADBEVideoTabGroup, ADBEBasicVideoGroup,
        L"Basic Video Settings", kPrFalse, kPrFalse, kPrFalse);

    // Width param
    exNewParamInfo widthParam = {};
    widthParam.structVersion = 1;
    strncpy(widthParam.identifier, ADBEVideoWidth, sizeof(exParamIdentifier));
    widthParam.paramType = exParamType_int;
    widthParam.flags = exParamFlag_none;
    widthParam.paramValues.structVersion = 1;
    widthParam.paramValues.value.intValue = 1920;
    widthParam.paramValues.rangeMin.intValue = 16;
    widthParam.paramValues.rangeMax.intValue = 8192;
    widthParam.paramValues.disabled = kPrFalse;
    widthParam.paramValues.hidden = kPrFalse;
    paramSuite->AddParam(pluginID, mgIndex, ADBEBasicVideoGroup, &widthParam);

    // Height param
    exNewParamInfo heightParam = {};
    heightParam.structVersion = 1;
    strncpy(heightParam.identifier, ADBEVideoHeight, sizeof(exParamIdentifier));
    heightParam.paramType = exParamType_int;
    heightParam.flags = exParamFlag_none;
    heightParam.paramValues.structVersion = 1;
    heightParam.paramValues.value.intValue = 1080;
    heightParam.paramValues.rangeMin.intValue = 16;
    heightParam.paramValues.rangeMax.intValue = 8192;
    paramSuite->AddParam(pluginID, mgIndex, ADBEBasicVideoGroup, &heightParam);

    // Frame rate param (uses ticks)
    PrTime ticksPerSecond = 0;
    PrSDKTimeSuite* timeSuite = NULL;
    spBasic->AcquireSuite("MediaCore Time Suite", 1, (const void**)&timeSuite);
    timeSuite->GetTicksPerSecond(&ticksPerSecond);

    exNewParamInfo fpsParam = {};
    fpsParam.structVersion = 1;
    strncpy(fpsParam.identifier, ADBEVideoFPS, sizeof(exParamIdentifier));
    fpsParam.paramType = exParamType_ticksFrameRate;
    fpsParam.flags = exParamFlag_none;
    fpsParam.paramValues.structVersion = 1;
    fpsParam.paramValues.value.timeValue = ticksPerSecond / 24;  // 24fps

    paramSuite->AddParam(pluginID, mgIndex, ADBEBasicVideoGroup, &fpsParam);

    return exportReturn_ErrNone;
}
```

### Constrained Value Lists (Dropdowns)

For enum-style parameters, add constrained value pairs:

```cpp
// In exSelPostProcessParams:
exOneParamValueRec cbrValue = {};
cbrValue.intValue = prVideoBitrateEncoding_CBR1Pass;
paramSuite->AddConstrainedValuePair(pluginID, mgIndex,
    ADBEVideoBitrateEncoding, &cbrValue, L"CBR");

exOneParamValueRec vbrValue = {};
vbrValue.intValue = prVideoBitrateEncoding_VBR1Pass;
paramSuite->AddConstrainedValuePair(pluginID, mgIndex,
    ADBEVideoBitrateEncoding, &vbrValue, L"VBR, 1 Pass");
```

### Reading Parameter Values

```cpp
exParamValues widthValue;
paramSuite->GetParamValue(pluginID, mgIndex, ADBEVideoWidth, &widthValue);
int width = widthValue.value.intValue;

exParamValues fpsValue;
paramSuite->GetParamValue(pluginID, mgIndex, ADBEVideoFPS, &fpsValue);
PrTime ticksPerFrame = fpsValue.value.timeValue;
```

---

## PrSDKExportStdParamSuite: Standard Parameters

The standard param suite provides pre-built parameter groups so you do not have to manually create common settings:

```cpp
#define kPrExportStdParamSuite  "Premiere Export Standard Param Suite"

typedef struct {
    void (*AddStandardParams)(
        csSDK_uint32      exporterPluginID,
        PrSDKStdParamType inSDKStdParamType);

    void (*PostProcessParamNames)(
        csSDK_uint32         exporterPluginID,
        PrAudioChannelType   inSourceAudioChannelType);

    void (*QueryOutputSettings)(
        csSDK_uint32              exporterPluginID,
        exQueryOutputSettingsRec* outOutputSettings);

    void (*MakeParamSummary)(
        csSDK_uint32  exporterPluginID,
        prBool        inDoVideo,
        prBool        inDoAudio,
        prBool        inIsDiscreteAudio,
        prUTF16Char*  outVideoDescription,   // 256 chars
        prUTF16Char*  outAudioDescription);   // 256 chars
} PrSDKExportStdParamSuite;
```

### Standard Param Types

| Enum | What it adds |
|---|---|
| `SDKStdParams_Video` | Width, height, FPS, field type, aspect ratio |
| `SDKStdParams_Audio` | Sample rate, sample type, channels |
| `SDKStdParams_Still` | Still image settings |
| `SDKStdParams_VideoBitrateGroup` | Target/max bitrate, encoding mode |
| `SDKStdParams_VideoWithSizePopup` | Video params with size preset popup |
| `SDKStdParams_AudioTabOnly` | Audio tab only |
| `SDKStdParams_AudioBitrateGroup` | Audio bitrate group |
| `SDKStdParams_AudioChannelConfigurationGroup` | Channel configuration |

### Usage

```cpp
case exSelGenerateDefaultParams:
{
    // Use standard params for video and audio
    stdParamSuite->AddStandardParams(pluginID, SDKStdParams_Video);
    stdParamSuite->AddStandardParams(pluginID, SDKStdParams_Audio);

    // Add custom codec-specific params on top
    // ...
    return exportReturn_ErrNone;
}

case exSelPostProcessParams:
{
    stdParamSuite->PostProcessParamNames(pluginID, sourceAudioChannelType);
    // Localize any custom param names here too
    return exportReturn_ErrNone;
}
```

---

## PrSDKExportInfoSuite: Querying Source Properties

Use `PrSDKExportInfoSuite` to query properties of the source being exported:

```cpp
#define kPrSDKExportInfoSuite  "MediaCore Export Info Suite"

prSuiteError (*GetExportSourceInfo)(
    csSDK_uint32                 inExporterPluginID,
    PrExportSourceInfoSelector   inSelector,
    PrParam*                     outSourceInfo);
```

### Available Selectors

| Selector | Returns | Type |
|---|---|---|
| `kExportInfo_VideoWidth` | Source width | Int32 |
| `kExportInfo_VideoHeight` | Source height | Int32 |
| `kExportInfo_VideoFrameRate` | Frame rate | PrTime |
| `kExportInfo_VideoFieldType` | Field type | Int32 (prFieldType) |
| `kExportInfo_VideoDuration` | Video duration | Int64 (PrTime) |
| `kExportInfo_PixelAspectNumerator` | PAR num | Int32 |
| `kExportInfo_PixelAspectDenominator` | PAR den | Int32 |
| `kExportInfo_AudioDuration` | Audio duration | Int64 (PrTime) |
| `kExportInfo_AudioChannelsType` | Channel type | Int32 |
| `kExportInfo_AudioSampleRate` | Sample rate | Float64 |
| `kExportInfo_SourceHasAudio` | Has audio | Bool |
| `kExportInfo_SourceHasVideo` | Has video | Bool |
| `kExportInfo_RenderAsPreview` | Is preview render | Bool |
| `kExportInfo_SequenceGUID` | Sequence identifier | GUID |
| `kExportInfo_UsePreviewFiles` | Use cached previews | Bool |
| `kExportInfo_SourceBitrate` | Source bitrate (kbps) | Int64 |

### Example: Matching source settings

```cpp
PrParam sourceWidth;
infoSuite->GetExportSourceInfo(pluginID, kExportInfo_VideoWidth, &sourceWidth);

PrParam sourceHeight;
infoSuite->GetExportSourceInfo(pluginID, kExportInfo_VideoHeight, &sourceHeight);

PrParam sourceFrameRate;
infoSuite->GetExportSourceInfo(pluginID, kExportInfo_VideoFrameRate, &sourceFrameRate);

// Use these to set "match source" defaults
```

---

## PrSDKExportFileSuite: File Output

The file suite handles writing to the output file:

```cpp
#define kPrSDKExportFileSuite  "MediaCore Export File Suite"

prSuiteError (*Open)(csSDK_uint32 inFileObject);

prSuiteError (*Write)(
    csSDK_uint32  inFileObject,
    void*         inBytes,
    csSDK_int32   inNumBytes);

prSuiteError (*Seek)(
    csSDK_uint32            inFileObject,
    prInt64                 inPosition,
    prInt64&                outNewPosition,
    ExFileSuite_SeekMode    inSeekMode);

prSuiteError (*Close)(csSDK_uint32 inFileObject);

prSuiteError (*GetPlatformPath)(
    csSDK_uint32   inFileObject,
    csSDK_int32*   outPathLength,
    prUTF16Char*   outPlatformPath);
```

The `inFileObject` comes from `exDoExportRec.fileObject`. Seek modes are:

| Mode | Description |
|---|---|
| `fileSeekMode_Begin` | Absolute position from start |
| `fileSeekMode_End` | Offset from end |
| `fileSeekMode_Current` | Offset from current position |

> **Warning**: In version 1 of the suite, `fileSeekMode_End` and `fileSeekMode_Current` were swapped due to a bug. Version 2 fixes this. Always acquire version 2.

---

## PrSDKExportProgressSuite: Progress Reporting

```cpp
#define kPrSDKExportProgressSuite  "MediaCore Export Progress Suite"

prSuiteError (*SetProgressString)(
    csSDK_uint32   inExportID,
    prUTF16Char*   inProgressString);    // NULL to reset to default

prSuiteError (*UpdateProgressPercent)(
    csSDK_uint32   inExportID,
    float          inPercent);           // 0.0 to 1.0

prSuiteError (*UpdateProgressPercentWithFrame)(
    csSDK_uint32   inExportID,
    float          inPercent,
    PPixHand       inPPix);              // Frame to show in preview

prSuiteError (*WaitForResume)(
    csSDK_uint32   inExportID);
```

`UpdateProgressPercent` returns one of:
- `exportReturn_ErrNone` -- Continue normally
- `exportReturn_Abort` -- User cancelled; stop exporting
- `suiteError_ExporterSuspended` -- Minimize memory, then call `WaitForResume()`

### Progress reporting pattern

```cpp
float totalFrames = (float)(endTime - startTime) / ticksPerFrame;
float currentFrame = 0;

for (PrTime time = startTime; time < endTime; time += ticksPerFrame)
{
    // Render frame...

    // Report progress
    prSuiteError progressResult = progressSuite->UpdateProgressPercent(
        exporterPluginID, currentFrame / totalFrames);

    if (progressResult == suiteError_ExporterSuspended)
    {
        progressSuite->WaitForResume(exporterPluginID);
    }
    else if (progressResult == exportReturn_Abort)
    {
        // Clean up and return exportReturn_Abort
        return exportReturn_Abort;
    }

    currentFrame += 1.0f;
}
```

---

## exSelExport: The Export Loop

The `exDoExportRec` provides everything needed for the export:

```cpp
typedef struct {
    csSDK_uint32     exporterPluginID;
    void*            privateData;
    csSDK_uint32     fileType;
    csSDK_int32      exportAudio;           // Always check these!
    csSDK_int32      exportVideo;
    PrTime           startTime;             // Export range start
    PrTime           endTime;               // Export range end
    csSDK_uint32     fileObject;            // For ExportFileSuite
    PrTimelineID     timelineData;          // For timeline render suites
    csSDK_int32      maximumRenderQuality;  // Non-zero = max quality
    csSDK_int32      embedCaptions;
    ColorProfileRec  colorProfile;
    PrSDKColorSpaceID exportColorSpaceID;
    csSDK_int32      maximumFileSize;       // Non-zero = size limit
} exDoExportRec;
```

### Complete Export Example

```cpp
case exSelExport:
{
    exDoExportRec* exportRec = reinterpret_cast<exDoExportRec*>(param1);
    MyExportData* myData = reinterpret_cast<MyExportData*>(exportRec->privateData);

    // 1. Read parameter values
    exParamValues widthVal, heightVal, fpsVal;
    paramSuite->GetParamValue(exportRec->exporterPluginID, 0, ADBEVideoWidth, &widthVal);
    paramSuite->GetParamValue(exportRec->exporterPluginID, 0, ADBEVideoHeight, &heightVal);
    paramSuite->GetParamValue(exportRec->exporterPluginID, 0, ADBEVideoFPS, &fpsVal);

    int width = widthVal.value.intValue;
    int height = heightVal.value.intValue;
    PrTime ticksPerFrame = fpsVal.value.timeValue;

    // 2. Open the output file
    fileSuite->Open(exportRec->fileObject);

    // 3. Set up the sequence render suite for fetching frames
    // (acquire PrSDKSequenceRenderSuite via SPBasic)
    csSDK_uint32 videoRendererID = 0;
    PrPixelFormat requestedFormat = PrPixelFormat_BGRA_4444_8u;

    sequenceRenderSuite->MakeVideoRenderer(
        exportRec->exporterPluginID,
        &videoRendererID,
        exportRec->timelineData);

    // 4. Export loop
    PrTime currentTime = exportRec->startTime;
    float totalFrames = (float)(exportRec->endTime - exportRec->startTime)
                        / ticksPerFrame;
    float frameIndex = 0;

    while (currentTime < exportRec->endTime)
    {
        if (exportRec->exportVideo)
        {
            // Render a frame from the timeline
            SequenceRender_ParamsRec renderParams = {};
            renderParams.inRequestedPixelFormatArray = &requestedFormat;
            renderParams.inRequestedPixelFormatArrayCount = 1;
            renderParams.inWidth = width;
            renderParams.inHeight = height;
            renderParams.inRenderQuality =
                exportRec->maximumRenderQuality
                    ? kPrRenderQuality_Max
                    : kPrRenderQuality_High;

            SequenceRender_GetFrameReturnRec frameReturn = {};
            sequenceRenderSuite->RenderVideoFrame(
                videoRendererID,
                currentTime,
                &renderParams,
                kRenderCacheType_None,
                &frameReturn);

            // Access pixels from the returned PPix
            char* pixels = NULL;
            ppixSuite->GetPixels(frameReturn.outFrame,
                                  PrPPixBufferAccess_ReadOnly, &pixels);
            csSDK_int32 rowBytes = 0;
            ppixSuite->GetRowBytes(frameReturn.outFrame, &rowBytes);

            // Encode and write the frame
            EncodeAndWriteFrame(pixels, width, height, rowBytes, exportRec->fileObject);

            // Dispose the frame
            ppixSuite->Dispose(frameReturn.outFrame);
        }

        // Update progress
        prSuiteError progErr = progressSuite->UpdateProgressPercent(
            exportRec->exporterPluginID, frameIndex / totalFrames);

        if (progErr == exportReturn_Abort)
        {
            sequenceRenderSuite->ReleaseVideoRenderer(
                exportRec->exporterPluginID, videoRendererID);
            fileSuite->Close(exportRec->fileObject);
            return exportReturn_Abort;
        }

        currentTime += ticksPerFrame;
        frameIndex += 1.0f;
    }

    // 5. Export audio (if applicable)
    if (exportRec->exportAudio)
    {
        ExportAudio(exportRec, fileSuite);
    }

    // 6. Finalize and close
    sequenceRenderSuite->ReleaseVideoRenderer(
        exportRec->exporterPluginID, videoRendererID);
    fileSuite->Close(exportRec->fileObject);

    return exportReturn_ErrNone;
}
```

---

## Audio Export

Audio is exported through the sequence audio render suite. Audio is always delivered as 32-bit float samples:

```cpp
void ExportAudio(exDoExportRec* exportRec, PrSDKExportFileSuite* fileSuite)
{
    csSDK_uint32 audioRendererID = 0;
    sequenceAudioSuite->MakeAudioRenderer(
        exportRec->exporterPluginID,
        exportRec->timelineData,
        kPrAudioSampleType_32BitFloat,
        kPrAudioSampleRate_48000,
        &audioRendererID);

    const csSDK_int32 samplesPerChunk = 8192;
    float* audioBuffer[6] = {};  // Up to 5.1

    // Allocate per-channel buffers
    for (int ch = 0; ch < numChannels; ch++)
    {
        audioBuffer[ch] = new float[samplesPerChunk];
    }

    PrAudioSample totalSamples = GetTotalAudioSamples(exportRec);
    PrAudioSample samplesRemaining = totalSamples;

    while (samplesRemaining > 0)
    {
        csSDK_int32 samplesToGet = (csSDK_int32)min(
            (PrAudioSample)samplesPerChunk, samplesRemaining);

        sequenceAudioSuite->GetAudio(
            audioRendererID,
            samplesToGet,
            audioBuffer,
            kPrFalse);  // not clipping

        // Encode and write audio chunk
        WriteAudioChunk(audioBuffer, numChannels, samplesToGet, fileSuite);

        samplesRemaining -= samplesToGet;
    }

    // Cleanup
    sequenceAudioSuite->ReleaseAudioRenderer(
        exportRec->exporterPluginID, audioRendererID);
    for (int ch = 0; ch < numChannels; ch++)
        delete[] audioBuffer[ch];
}
```

---

## Multi-Pass Export

For VBR 2-pass encoding, perform a first analysis pass without writing the final file:

```cpp
// Pass 1: Analysis
for (PrTime time = startTime; time < endTime; time += ticksPerFrame)
{
    // Render frame, analyze complexity
    // Do NOT write to output file
    AnalyzeFrameComplexity(frame);
    progressSuite->UpdateProgressPercent(pluginID, progress * 0.5f);
}

// Pass 2: Encode with bitrate allocation
for (PrTime time = startTime; time < endTime; time += ticksPerFrame)
{
    // Render frame, encode with optimal bitrate
    EncodeFrameWithAllocation(frame);
    progressSuite->UpdateProgressPercent(pluginID, 0.5f + progress * 0.5f);
}
```

Scale progress to 0.0-0.5 for pass 1 and 0.5-1.0 for pass 2.

---

## exSelQueryOutputSettings

This selector is required. Fill `exQueryOutputSettingsRec` with the current settings:

```cpp
case exSelQueryOutputSettings:
{
    exQueryOutputSettingsRec* settingsRec =
        reinterpret_cast<exQueryOutputSettingsRec*>(param1);

    settingsRec->outVideoWidth = 1920;
    settingsRec->outVideoHeight = 1080;
    settingsRec->outVideoFrameRate = ticksPerFrame;
    settingsRec->outVideoAspectNum = 1;
    settingsRec->outVideoAspectDen = 1;
    settingsRec->outVideoFieldType = prFieldsNone;
    settingsRec->outAudioSampleRate = 48000.0;
    settingsRec->outAudioSampleType = kPrAudioSampleType_32BitFloat;
    settingsRec->outAudioChannelType = kPrAudioChannelType_Stereo;
    settingsRec->outBitratePerSecond = 20000000;

    // Or use the standard param suite:
    // stdParamSuite->QueryOutputSettings(pluginID, settingsRec);

    return exportReturn_ErrNone;
}
```

---

## exSelGetParamSummary

Generate 3 lines of summary text:

```cpp
case exSelGetParamSummary:
{
    exParamSummaryRec* summaryRec =
        reinterpret_cast<exParamSummaryRec*>(param1);

    swprintf(summaryRec->videoSummary, 256,
        L"1920x1080, 23.976 fps, SDK Codec");
    swprintf(summaryRec->audioSummary, 256,
        L"48 kHz, 16-bit, Stereo");
    swprintf(summaryRec->bitrateSummary, 256,
        L"Target: 20 Mbps, VBR 1 Pass");

    return exportReturn_ErrNone;
}
```

---

## Return Values

From `enum PrExportReturnValue`:

| Value | Meaning |
|---|---|
| `exportReturn_ErrNone` | Success |
| `exportReturn_Abort` | User cancelled |
| `exportReturn_Done` | Export finished normally |
| `exportReturn_InternalError` | Internal error |
| `exportReturn_OutOfDiskSpace` | Disk full |
| `exportReturn_ErrMemory` | Out of memory |
| `exportReturn_IterateExporter` | More exporters to register (from exSelStartup) |
| `exportReturn_IterateExporterDone` | Last exporter registered |
| `exportReturn_InternalErrorSilent` | Error, but plugin shows its own message |
| `exportReturn_ErrLastErrorSet` | Error string set via PrSDKErrorSuite |
| `exportReturn_ErrLastWarningSet` | Warning string set via PrSDKErrorSuite |
| `exportReturn_Unsupported` | Selector not handled |
| `exportReturn_ParamButtonCancel` | User cancelled settings dialog |

---

## Pitfalls and AE Developer Notes

1. **Always check exportAudio/exportVideo**: These flags in `exDoExportRec` may differ from your initial capabilities. The user can disable video or audio export independently.

2. **Ticks, not frames**: All time values in the export system use PrTime ticks. Get the ticks-per-second value from `PrSDKTimeSuite` and compute frame times from there.

3. **exSelPostProcessParams is not optional**: Even if you have no localization, you must handle this selector to populate constrained value lists. Presets loaded from disk will not have constrained value names until you add them here.

4. **exSelValidateParamChanged can have empty changedParamIdentifier**: The `changedParamIdentifier` may be empty when `exportAudio`, `exportVideo`, or the multigroup index changed. Always check for this case.

5. **The file suite Write is not buffered**: Each `Write()` call hits the OS. For performance, buffer your own data and write in large chunks.

6. **Progress reporting is mandatory**: If you do not call `UpdateProgressPercent()`, the user cannot cancel the export. Even if you set `wantsNoProgressBar`, you should still check for abort conditions.

7. **Color management**: If `supportsColorManagement` is true, use the `exportColorSpaceID` from `exDoExportRec` when requesting rendered frames, so the host can convert to the correct output color space.

8. **Suspend/resume**: When `UpdateProgressPercent` returns `suiteError_ExporterSuspended`, the system is under memory pressure. Call `WaitForResume()` and the system will return control when memory is available. This is not an error condition.
