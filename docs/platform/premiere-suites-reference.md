# Premiere Pro Suite Reference: Memory, Time, Errors, and Utilities

This document covers the utility suites that nearly every Premiere Pro plugin needs, regardless of plugin type (importer, exporter, effect, play module, device control, etc.). These suites handle memory allocation, time representation, error reporting, string management, host identification, application settings, sequence properties, audio rendering, window access, and color management.

## Suite Acquisition

All Premiere suites are acquired through the `SPBasicSuite` (SweetPea2). The pattern is consistent:

```cpp
SomeSDKSuite* someSuite = nullptr;
SPErr err = spBasicSuite->AcquireSuite(
    kPrSDKSomeSuiteName,           // Suite name string constant
    kPrSDKSomeSuiteVersion,        // Version number
    reinterpret_cast<const void**>(&someSuite));

if (err == kSPNoError && someSuite != nullptr)
{
    // Use the suite
}

// When done (typically at plugin shutdown):
spBasicSuite->ReleaseSuite(kPrSDKSomeSuiteName, kPrSDKSomeSuiteVersion);
```

For plugins that receive a `piSuitesPtr` in their standard parameters, suites can also be acquired through `piSuites->utilFuncs->getSPBasicSuite()`.

---

## PrSDKTimeSuite -- Time Representation

**Suite ID:** `"Premiere Time Suite"` / Version 1

Premiere Pro uses **tick-based time**, not frame numbers. All time values are stored as `PrTime`, which is a signed 64-bit integer (`prInt64`). The tick rate is constant for the duration of the application's runtime but must be queried -- never hardcode it.

### Core Type

```cpp
typedef prInt64 PrTime;
```

### RatioTime

For values not representable by integer ticks:

```cpp
struct PrRatioTime
{
    prInt64 mValue;
    prInt64 mScale;
};
```

### Suite Functions

```cpp
typedef struct
{
    prSuiteError (*GetTicksPerSecond)(PrTime* outTicksPerSec);

    prSuiteError (*GetTicksPerVideoFrame)(
        PrVideoFrameRates inVideoFrameRate,
        PrTime*           outTicksPerFrame);

    prSuiteError (*GetTicksPerAudioSample)(
        float   inSampleRate,
        PrTime* outTicksPerSample);
} PrSDKTimeSuite;
```

### Video Frame Rates

The `PrVideoFrameRates` enum provides standard broadcast rates:

| Enum | Value | Frame Rate |
|------|-------|------------|
| `kVideoFrameRate_24Drop` | 1 | 24000/1001 (23.976) |
| `kVideoFrameRate_24` | 2 | 24 |
| `kVideoFrameRate_PAL` | 3 | 25 |
| `kVideoFrameRate_NTSC` | 4 | 30000/1001 (29.97) |
| `kVideoFrameRate_30` | 5 | 30 |
| `kVideoFrameRate_PAL_HD` | 6 | 50 |
| `kVideoFrameRate_NTSC_HD` | 7 | 60000/1001 (59.94) |
| `kVideoFrameRate_60` | 8 | 60 |

### Playback Timebase

For play modules, there is also a timebase struct:

```cpp
typedef struct {
    csSDK_uint32 scale;         // rate of the timebase
    csSDK_int32  sampleSize;    // size of one sample
    csSDK_int32  fileDuration;  // number of samples in file
} pmPlayTimebase;
// Example: scale=2997, sampleSize=100, fileDuration=numframes*100
```

### Time Conversion Examples

```cpp
PrTime ticksPerSecond;
timeSuite->GetTicksPerSecond(&ticksPerSecond);

// Convert a frame number to PrTime at 29.97fps
PrTime ticksPerFrame;
timeSuite->GetTicksPerVideoFrame(kVideoFrameRate_NTSC, &ticksPerFrame);
PrTime frameTime = frameNumber * ticksPerFrame;

// Convert PrTime to seconds (floating point)
double seconds = static_cast<double>(someTime) / static_cast<double>(ticksPerSecond);

// Convert PrTime to an audio sample offset at 48kHz
PrTime ticksPerSample;
timeSuite->GetTicksPerAudioSample(48000.0f, &ticksPerSample);
int64_t sampleOffset = someTime / ticksPerSample;
```

### Pitfall: Audio Sample Rate Rounding

`GetTicksPerAudioSample` returns `kPrTimeSuite_RoundedAudioRate` if the requested sample rate is not an even divisor of the base tick count. Times at that rate will not be exact. Standard rates (44100, 48000, 96000) are typically exact. Non-standard rates may introduce drift over long durations.

---

## PrSDKMemoryManagerSuite -- Memory Allocation

**Suite ID:** `"Premiere Memory Manager Suite"` / Version 4

Premiere provides its own memory management with integrated cache management. There have been four versions, with the current (V4 / `PrSDKMemoryManagerSuite`) being the most capable.

### Version Progression

| Version | SDK Era | Key Addition |
|---------|---------|-------------|
| V1 | Original | `ReserveMemory` only |
| V2 | Later | Added `NewPtr`, `NewHandle`, and friends |
| V3/V4 | CS4+ | Added managed cache blocks with purge callbacks |

### Current Suite (Version 4)

```cpp
typedef struct
{
    // Reserve memory for your plugin
    prSuiteError (*ReserveMemory)(csSDK_uint32 inPluginID, csSDK_uint32 inSize);

    // Query the current media cache size
    prSuiteError (*GetMemoryManagerSize)(csSDK_uint64* outMemoryManagerSize);

    // --- Managed Cache Block API ---
    prSuiteError (*AddBlock)(
        csSDK_size_t inSize,
        PrSDKMemoryManagerSuite_PurgeMemoryFunction inPurgeFunction,
        void* inPurgeMemoryData,
        csSDK_uint32* outID);

    prSuiteError (*TouchBlock)(csSDK_uint32 inID);
    prSuiteError (*RemoveBlock)(csSDK_uint32 inID);

    // --- Pointer-based allocation ---
    PrMemoryPtr (*NewPtrClear)(csSDK_uint32 byteCount);
    PrMemoryPtr (*NewPtr)(csSDK_uint32 byteCount);
    csSDK_uint32 (*GetPtrSize)(PrMemoryPtr p);
    void (*SetPtrSize)(PrMemoryPtr* p, csSDK_uint32 newSize);
    void (*PrDisposePtr)(PrMemoryPtr p);

    // --- Handle-based allocation ---
    PrMemoryHandle (*NewHandle)(csSDK_uint32 byteCount);
    PrMemoryHandle (*NewHandleClear)(csSDK_uint32 byteCount);
    void (*DisposeHandle)(PrMemoryHandle h);
    short (*SetHandleSize)(PrMemoryHandle h, csSDK_uint32 newSize);
    csSDK_uint32 (*GetHandleSize)(PrMemoryHandle h);

    // Adjust reserved memory (signed, can release)
    prSuiteError (*AdjustReservedMemorySize)(
        csSDK_uint32 inPluginID, csSDK_int64 inSize);
} PrSDKMemoryManagerSuite;
```

### Managed Cache Blocks

The managed cache API lets you register memory blocks with the host's LRU cache. When Premiere needs memory, it calls your purge callback:

```cpp
typedef void (*PrSDKMemoryManagerSuite_PurgeMemoryFunction)(
    void* inPurgeMemoryData,
    csSDK_uint32 inMemoryID);
```

**Important:** The purge callback may be called on **any thread**. Ensure your purge handler is thread-safe.

```cpp
// Register a cached decode buffer
csSDK_uint32 blockID;
memorySuite->AddBlock(
    bufferSize,
    MyPurgeCallback,
    myContextData,
    &blockID);

// Touch the block each time you use it (promotes in LRU)
memorySuite->TouchBlock(blockID);

// Manually remove when no longer needed (purge callback will NOT be called)
memorySuite->RemoveBlock(blockID);
```

### ReserveMemory vs. AdjustReservedMemorySize

- `ReserveMemory` sets an absolute reservation size for your plugin.
- `AdjustReservedMemorySize` takes a signed 64-bit value, letting you incrementally increase or decrease your reservation. Use this for dynamic memory needs.

### Pitfall: 32-bit Size Limits

The `NewPtr` and `NewHandle` functions take `csSDK_uint32` for the byte count, limiting individual allocations to ~4GB. For larger allocations, use standard system allocators and register them via `AddBlock` for cache management.

---

## PrSDKErrorSuite -- Error Reporting

**Suite ID:** `"Premiere Error Suite"` / Version 3

Plugins report errors, warnings, and informational messages to Premiere's Events panel.

### Version History

| Version | Era | Feature |
|---------|-----|---------|
| V1 | Pro 1.0 | Separate `SetErrorString`, `SetWarningString`, `SetInfoString` (ASCII) |
| V2 | Pro 1.5 | Unified `SetEventString` with title + description (ASCII) |
| V3 | CS4+ | `SetEventStringUnicode` with UTF-16 support and event flags |

### Current Suite (Version 3)

```cpp
typedef struct
{
    enum {
        kEventTypeInformational = 1,
        kEventTypeWarning,
        kEventTypeError,

        kEventType_Mask = 0xFF,

        // Flags (bitwise OR with event type):
        kEventFlag_DecodeError       = 0x100,
        kEventFlag_SubstitutedFrame  = 0x200,
        kEventFlag_ImportOperation   = 0x400,
    };

    prSuiteError (*SetEventStringUnicode)(
        csSDK_uint32  eventType,
        prUTF16Char*  eventTitle,
        prUTF16Char*  eventDescription);
} PrSDKErrorSuite3;
```

### Event Type Flags

The `eventType` parameter combines a base type with optional flags:

```cpp
// Simple error
errorSuite->SetEventStringUnicode(
    PrSDKErrorSuite3::kEventTypeError,
    L"Export Failed",
    L"The output file could not be written. Check disk space.");

// Decode error with substituted frame
errorSuite->SetEventStringUnicode(
    PrSDKErrorSuite3::kEventTypeWarning
        | PrSDKErrorSuite3::kEventFlag_DecodeError
        | PrSDKErrorSuite3::kEventFlag_SubstitutedFrame,
    L"Frame Decode Warning",
    L"Frame at 00:01:23:15 could not be decoded. A nearby frame was substituted.");

// Import operation with filename
errorSuite->SetEventStringUnicode(
    PrSDKErrorSuite3::kEventTypeWarning
        | PrSDKErrorSuite3::kEventFlag_ImportOperation,
    L"Import Warning",
    L"Unsupported metadata in \"myfile.mxf\"");
```

When `kEventFlag_ImportOperation` is set, Premiere expects the description to end with a filename in quotes.

### Version 1 (Legacy, ASCII)

Still available for backward compatibility:

```cpp
prSuiteError (*SetErrorString)(const char* inErrorString, csSDK_uint32 inContextID);
prSuiteError (*SetWarningString)(const char* inWarningString, csSDK_uint32 inContextID);
prSuiteError (*SetInfoString)(const char* inInfoString, csSDK_uint32 inContextID);
```

The `inContextID` is the context ID passed to the plugin (e.g., a `CompilerID` for exporters). Premiere concatenates all pending strings of each type until they can be displayed.

---

## PrSDKAppInfoSuite -- Host Version Detection

**Suite ID:** `"MediaCore App Info Suite"` / Version 3

Identifies which Adobe application is hosting your plugin and its version.

```cpp
typedef struct
{
    enum {
        kAppInfo_AppFourCC,     // csSDK_uint32
        kAppInfo_Version,       // VersionInfo struct
        kAppInfo_Build,         // csSDK_uint32 (added in version 2)
        kAppInfo_Language,      // LanguageInfo struct (added in version 3)
    };

    prSuiteError (*GetAppInfo)(int settingsSelector, void* appInfo);
} PrSDKAppInfoSuite;
```

### Host Application FourCC Codes

| Constant | FourCC | Application |
|----------|--------|-------------|
| `kAppPremierePro` | `'PPro'` | Adobe Premiere Pro |
| `kAppPremiereElements` | `'PrEl'` | Adobe Premiere Elements |
| `kAppAfterEffects` | `'FXTC'` | Adobe After Effects |
| `kAppMediaEncoder` | `'AME '` | Adobe Media Encoder |
| `kAppPrelude` | `'PRLD'` | Adobe Prelude |
| `kAppAudition` | `'AdAu'` | Adobe Audition |
| `kAppCharacterAnimator` | `'Anml'` | Adobe Character Animator |

### Usage

```cpp
// Detect host application
csSDK_uint32 appFourCC = 0;
appInfoSuite->GetAppInfo(PrSDKAppInfoSuite::kAppInfo_AppFourCC, &appFourCC);

if (appFourCC == kAppPremierePro)
{
    // Premiere Pro specific behavior
}
else if (appFourCC == kAppMediaEncoder)
{
    // AME specific behavior (e.g., no UI)
}

// Get version
VersionInfo version;
appInfoSuite->GetAppInfo(PrSDKAppInfoSuite::kAppInfo_Version, &version);
// version.major, version.minor, version.patch

// Get language
LanguageInfo language;
appInfoSuite->GetAppInfo(PrSDKAppInfoSuite::kAppInfo_Language, &language);
// language.languageID is e.g. "en_US", "ja_JP", "zh_CN"
```

### Version and Language Structs

```cpp
typedef struct
{
    unsigned int major;
    unsigned int minor;
    unsigned int patch;
} VersionInfo;

typedef struct
{
    char languageID[6];  // e.g., "en_US"
} LanguageInfo;
```

### Pitfall: MediaCore Shared Suites

This suite is part of the "MediaCore" layer shared across Adobe applications. The same importer or exporter plugin can load in Premiere Pro, After Effects, Media Encoder, and Audition. Always check the host FourCC before using host-specific features.

---

## PrSDKApplicationSettingsSuite -- Application Preferences

**Suite ID:** `"MediaCore Application Settings Suite"` / Version 1

Provides access to project-level and application-level settings.

```cpp
typedef struct
{
    prSuiteError (*GetScratchDiskFolder)(
        unsigned int  inScratchDiskType,
        PrSDKString*  outScratchDiskFolder);

    prSuiteError (*GetCurrentProjectPath)(
        PrSDKString* outCurrentProjectPath);
} PrSDKApplicationSettingsSuite;
```

### Scratch Disk Types

```cpp
enum ScratchDiskType
{
    kScratchDiskType_CaptureVideo = 0,
    kScratchDiskType_CaptureAudio,
    kScratchDiskType_PreviewVideo,
    kScratchDiskType_PreviewAudio,
};
```

### Usage with PrSDKStringSuite

The returned `PrSDKString` must be converted using the String Suite:

```cpp
PrSDKString projectPath;
prSuiteError err = appSettingsSuite->GetCurrentProjectPath(&projectPath);
if (err == suiteError_NoError)
{
    // Get required buffer size
    prUTF16Char* utf16Path = nullptr;
    csSDK_uint32 bufferElements = 0;
    stringSuite->CopyToUTF16String(&projectPath, nullptr, &bufferElements);

    // Allocate and copy
    utf16Path = new prUTF16Char[bufferElements];
    stringSuite->CopyToUTF16String(&projectPath, utf16Path, &bufferElements);

    // Use utf16Path...

    delete[] utf16Path;
    stringSuite->DisposeString(&projectPath);
}
```

### Pitfall: Context Requirements

Both functions return `suiteError_InvalidCall` if:
- No project is currently loaded
- Called from a helper application context (e.g., ImporterProcessServer)

Always handle this return code gracefully.

---

## PrSDKStringSuite -- String Handling

**Suite ID:** `"MediaCore StringSuite"` / Version 1

Premiere uses `PrSDKString` as an opaque string type. This suite converts between `PrSDKString` and UTF-8 or UTF-16 C strings.

```cpp
typedef struct
{
    prSuiteError (*DisposeString)(const PrSDKString* inSDKString);

    prSuiteError (*AllocateFromUTF8)(
        const prUTF8Char* inUTF8String,
        PrSDKString*      outSDKString);

    prSuiteError (*CopyToUTF8String)(
        const PrSDKString* inSDKString,
        prUTF8Char*        outUTF8StringBuffer,
        csSDK_uint32*      ioUTF8StringBufferSizeInElements);

    prSuiteError (*AllocateFromUTF16)(
        const prUTF16Char* inUTF16String,
        PrSDKString*       outSDKString);

    prSuiteError (*CopyToUTF16String)(
        const PrSDKString* inSDKString,
        prUTF16Char*       outUTF16StringBuffer,
        csSDK_uint32*      ioUTF16StringBufferSizeInElements);
} PrSDKStringSuite;
```

### Two-Pass Copy Pattern

Both `CopyToUTF8String` and `CopyToUTF16String` use a two-pass pattern:

```cpp
// Pass 1: Get required size
csSDK_uint32 bufferSize = 0;
stringSuite->CopyToUTF8String(&sdkString, nullptr, &bufferSize);

// Pass 2: Copy the data
prUTF8Char* buffer = new prUTF8Char[bufferSize];
stringSuite->CopyToUTF8String(&sdkString, buffer, &bufferSize);

// ... use buffer ...
delete[] buffer;
```

### Return Codes

| Code | Meaning |
|------|---------|
| `suiteError_NoError` | Success |
| `suiteError_InvalidParms` | A parameter is null or invalid |
| `suiteError_StringBufferTooSmall` | Buffer too small; `ioBufferSizeInElements` updated with required size |
| `suiteError_StringNotFound` | String was not allocated or already disposed |

### Pitfall: Always Dispose

Every `PrSDKString` you receive from a suite call must be disposed with `DisposeString`. Failing to do so leaks memory. It is safe to dispose an empty string.

---

## PrSDKSequenceInfoSuite -- Sequence Properties

**Suite ID:** `"MediaCore Sequence Info Suite"` / Version 9

Query properties of the active sequence. All functions take a `PrTimelineID`.

```cpp
typedef struct
{
    prSuiteError (*GetFrameRect)(PrTimelineID id, prRect* outFrameRect);

    prSuiteError (*GetPixelAspectRatio)(
        PrTimelineID id,
        csSDK_uint32* outNumerator,
        csSDK_uint32* outDenominator);

    prSuiteError (*GetFrameRate)(PrTimelineID id, PrTime* outTicksPerFrame);

    prSuiteError (*GetFieldType)(PrTimelineID id, prFieldType* outFieldType);

    prSuiteError (*GetZeroPoint)(PrTimelineID id, PrTime* outTime);

    prSuiteError (*GetTimecodeDropFrame)(PrTimelineID id, prBool* outDropFrame);

    prSuiteError (*GetProxyFlag)(PrTimelineID id, prBool* outProxyFlag);

    prSuiteError (*GetImmersiveVideoVRConfiguration)(
        PrTimelineID id,
        PrIVProjectionType* outProjectionType,
        PrIVFrameLayout*    outFrameLayout,
        csSDK_uint32*       outHorizontalCapturedView,
        csSDK_uint32*       outVerticalCapturedView);

    prSuiteError (*GetWorkingColorSpace)(
        PrTimelineID id,
        PrSDKColorSpaceID* outPrWorkingColorSpaceID);

    prSuiteError (*GetGraphicsWhiteLuminance)(
        PrTimelineID id,
        csSDK_uint32* outGraphicsWhiteLuminance);

    prSuiteError (*GetLUTInterpolationMethod)(
        PrTimelineID id,
        csSDK_uint32* outLUTInterpolationMethod);

    prSuiteError (*GetAutoToneMapEnabled)(
        PrTimelineID id,
        prBool* outAutoToneMapEnabled);

    prSuiteError (*GetSDRGamma)(
        PrTimelineID id,
        csSDK_uint32* outSDRGamma);
} PrSDKSequenceInfoSuite;
```

### HDR Graphics White Luminance Values

| Constant | Value (nits) | Use |
|----------|-------------|-----|
| `kSDRReference` | 100 | 1.0 mapped to 100 nits (Rec.709) |
| `kHDRReferenceForHLG` | 203 | 1.0 mapped to 203 nits (HLG) |
| `kHDRReferenceForPQ` | 300 | 1.0 mapped to 300 nits (PQ) |

### LUT Interpolation Methods

| Value | Method |
|-------|--------|
| 0 | Trilinear |
| 1 | Tetrahedral |

### SDR Gamma Values

The value is the gamma multiplied by 100:

| Value | Gamma | Use |
|-------|-------|-----|
| 240 | 2.4 | Broadcast |
| 196 | 1.96 | QuickTime |
| 220 | 2.2 | Web delivery |

### Zero Point

`GetZeroPoint` returns the sequence start time. Premiere sequences do not necessarily start at timecode 00:00:00:00. The zero point offset must be applied when converting between timeline time and display timecode.

---

## PrSDKSequenceAudioSuite -- Audio Rendering from Timeline

**Suite ID:** `"MediaCore Sequence Audio Suite"` / Version 3

Creates an audio renderer that returns mixed-down audio from the timeline. Used by exporters and play modules.

### Current Suite (Version 3)

```cpp
typedef struct
{
    prSuiteError (*MakeAudioRenderer)(
        csSDK_uint32        inPluginID,
        PrTime              inStartTime,
        csSDK_uint32        inNumChannels,
        PrAudioChannelLabel* inChannelLabels,
        PrAudioSampleType   inSampleType,
        float               inSampleRate,
        csSDK_uint32*       outAudioRenderID);

    prSuiteError (*ReleaseAudioRenderer)(
        csSDK_uint32 inPluginID,
        csSDK_uint32 inAudioRenderID);

    prSuiteError (*GetAudio)(
        csSDK_uint32 inAudioRenderID,
        csSDK_uint32 inFrameCount,
        float**      inBuffer,
        char         inClipAudio);

    prSuiteError (*ResetAudioToBeginning)(csSDK_uint32 inAudioRenderID);

    prSuiteError (*GetMaxBlip)(
        csSDK_uint32 inAudioRenderID,
        PrTime       inTicksPerFrame,
        csSDK_int32* maxBlipSize);

    // Version 3 addition:
    prSuiteError (*MakeAudioRendererWithQuality)(
        csSDK_uint32        inPluginID,
        PrTime              inStartTime,
        csSDK_uint32        inNumChannels,
        PrAudioChannelLabel* inChannelLabels,
        PrAudioSampleType   inSampleType,
        float               inSampleRate,
        PrRenderQuality     inRenderQuality,
        csSDK_uint32*       outAudioRenderID);
} PrSDKSequenceAudioSuite3;
```

### Audio Quality Levels

When using `MakeAudioRendererWithQuality`:

| Quality | Resampling Algorithm |
|---------|---------------------|
| `kPrRenderQuality_Draft` | Linear (high aliasing, fast) |
| `kPrRenderQuality_Low` / `kPrRenderQuality_Medium` | Cubic (moderate aliasing, moderate speed) |
| `kPrRenderQuality_High` / `kPrRenderQuality_Max` | Sinc (no measurable aliasing, slow) |

### Usage Pattern

```cpp
// Create a stereo 48kHz audio renderer
PrAudioChannelLabel channelLabels[] = {
    kPrAudioChannelLabel_FrontLeft,
    kPrAudioChannelLabel_FrontRight
};

csSDK_uint32 audioRenderID = 0;
audioSuite->MakeAudioRendererWithQuality(
    pluginID,
    startTime,
    2,                          // stereo
    channelLabels,
    kPrAudioSampleType_32BitFloat,
    48000.0f,
    kPrRenderQuality_High,
    &audioRenderID);

// Allocate uninterleaved buffers
const csSDK_uint32 framesPerChunk = 4096;
float* buffers[2];
buffers[0] = new float[framesPerChunk];
buffers[1] = new float[framesPerChunk];

// Read audio sequentially
audioSuite->GetAudio(audioRenderID, framesPerChunk, buffers, true);
// buffers[0] = left channel, buffers[1] = right channel

// For multi-pass encoding, reset to start
audioSuite->ResetAudioToBeginning(audioRenderID);

// Cleanup
audioSuite->ReleaseAudioRenderer(pluginID, audioRenderID);
delete[] buffers[0];
delete[] buffers[1];
```

### GetMaxBlip

Returns the maximum "blip" size -- the largest number of audio sample frames that correspond to a single video frame at the given frame rate. This is useful for aligning audio chunks to video frame boundaries:

```cpp
csSDK_int32 maxBlip = 0;
audioSuite->GetMaxBlip(audioRenderID, ticksPerFrame, &maxBlip);
```

### Version History

| Version | Era | Change |
|---------|-----|--------|
| V1 | CS4 | Used `PrAudioChannelType` enum |
| V2 | 9.0 | Switched to `PrAudioChannelLabel` array for flexible layouts |
| V3 | 22.1 | Added `MakeAudioRendererWithQuality` |

### Error Codes

| Code | Meaning |
|------|---------|
| `sequenceAudioSuite_ErrIncompatibleChannelType` | `'chan'` -- Incompatible channel type/layout |
| `sequenceAudioSuite_ErrUnknown` | 255 -- Unknown error |

### Pitfall: Audio Data Format

`GetAudio` always returns **uninterleaved float arrays**. The `inBuffer` parameter must point to an array of `float*` pointers, one per channel. Each pointer must have room for `inFrameCount` floats. The function returns the next contiguous chunk of audio -- you cannot seek to arbitrary positions (use `ResetAudioToBeginning` for multi-pass).

---

## PrSDKWindowSuite -- UI Integration

**Suite ID:** `"Premiere Window Suite"` / Version 1

A minimal suite for accessing the host's main window and triggering UI refreshes.

```cpp
typedef struct
{
    prWnd (*GetMainWindow)();
    void  (*UpdateAllWindows)();
} PrSDKWindowSuite;
```

- `GetMainWindow()` returns a platform window handle (`HWND` on Windows, `NSWindow*` on macOS via the `prWnd` typedef). Use this as a parent for modal dialogs.
- `UpdateAllWindows()` forces a repaint of all Premiere panels. Call after making changes that affect the UI (e.g., after programmatically modifying sequence properties).

---

## PrSDKColorManagementSuite -- Color Profiles

**Suite ID:** `"Color Management Suite"` / Version 1

Allows plugins to interpret the opaque `PrSDKColorSpaceID` that Premiere passes for source clips and sequence working spaces.

```cpp
typedef struct
{
    prSuiteError (*GetColorSpaceTypeForColorSpace)(
        const PrSDKColorSpaceID* inSDKColorSpaceID,
        PrSDKColorSpaceType*     outSDKColorSpaceType);

    prSuiteError (*HasICCEquivalentColorSpaceForColorSpace)(
        const PrSDKColorSpaceID* inSDKColorSpaceID,
        prBool*                  outHasICCEquivalentColorSpace);

    prSuiteError (*GetICCEquivalentColorSpaceForColorSpace)(
        const PrSDKColorSpaceID* inSDKColorSpaceID,
        PrSDKColorSpaceID*       outSDKICCColorSpaceID);

    prSuiteError (*GetSEIColorCodesForColorSpace)(
        const PrSDKColorSpaceID* inSDKColorSpaceID,
        prSEIColorCodesRec*      outSEIColorCodesRec);

    prSuiteError (*GetICCProfileSizeForColorSpace)(
        const PrSDKColorSpaceID* inSDKColorSpaceID,
        csSDK_int32*             outICCProfileSize);

    prSuiteError (*GetICCProfileForColorSpace)(
        const PrSDKColorSpaceID* inSDKColorSpaceID,
        PrMemoryPtr              ioICCProfileData);

    prSuiteError (*GetDisplayNameForPredefinedColorSpace)(
        const char* inSDKPredefinedColorSpace,
        prUTF16Char outDisplayName[256]);
} PrSDKColorManagementSuite;
```

### Color Space Types

A `PrSDKColorSpaceID` can represent different kinds of color spaces:

- **Predefined** (`kPrSDKColorSpaceType_Predefined`) -- One of the standard spaces from `PrSDKColorSpaces.h`
- **SEI Tags** (`kPrSDKColorSpaceType_SEITags`) -- Described by H.264/H.265 SEI color primaries/transfer/matrix codes
- **ICC Profile** (`kPrSDKColorSpaceType_ICC`) -- Described by an ICC profile blob
- **Undefined** (`kPrSDKColorSpaceType_Undefined`) -- Unknown color space; handle gracefully

### Predefined Color Spaces (from PrSDKColorSpaces.h)

A selection of the most common predefined color space tokens:

| Token | Description |
|-------|-------------|
| `kPrSRGBColorSpace` | sRGB (full range, RGB, 8-bit, Display) |
| `kPrRec709` | BT.709 (narrow range, YCC, 8-bit, Display) |
| `kPrRec709Scene` | BT.709 Scene-referred |
| `kPrOverranged709` | BT.709 RGB Full (32-bit float, over-range connection space) |
| `kPrRec2020` | BT.2020 (narrow range, YCC NCL, 10-bit, Display) |
| `kPrRec601525ColorSpace` | BT.601 NTSC |
| `kPrRec601625ColorSpace` | BT.601 PAL |

The over-ranged 709 space (`kPrOverranged709`) serves as the **connection space** between the host and plugins when the host does not understand a plugin's native color space. All importers and exporters should support conversion between their native space and this connection space.

### Getting an ICC Profile

```cpp
PrSDKColorSpaceID colorSpaceID = /* from sequence or clip */;

// Check if ICC is available
PrSDKColorSpaceType csType;
colorMgmtSuite->GetColorSpaceTypeForColorSpace(&colorSpaceID, &csType);

if (csType == kPrSDKColorSpaceType_ICC)
{
    csSDK_int32 profileSize = 0;
    colorMgmtSuite->GetICCProfileSizeForColorSpace(&colorSpaceID, &profileSize);

    PrMemoryPtr profileData = memorySuite->NewPtr(profileSize);
    colorMgmtSuite->GetICCProfileForColorSpace(&colorSpaceID, profileData);

    // Use ICC profile data...
    memorySuite->PrDisposePtr(profileData);
}
```

### Getting SEI Color Codes

For H.264/H.265 export, you may need the standard video usability information codes:

```cpp
prSEIColorCodesRec seiCodes;
colorMgmtSuite->GetSEIColorCodesForColorSpace(&colorSpaceID, &seiCodes);
// seiCodes contains colour_primaries, transfer_characteristics, matrix_coefficients
```

### Pitfall: Undefined Color Spaces

`GetColorSpaceTypeForColorSpace` may return `kPrSDKColorSpaceType_Undefined`. Your plugin must handle this case -- typically by falling back to BT.709 or the over-ranged 709 connection space.

---

## Quick Reference Table

| Suite | String ID | Version | Primary Use |
|-------|-----------|---------|-------------|
| TimeSuite | `"Premiere Time Suite"` | 1 | Tick-based time conversion |
| MemoryManagerSuite | `"Premiere Memory Manager Suite"` | 4 | Memory allocation and caching |
| ErrorSuite | `"Premiere Error Suite"` | 3 | User-facing error/warning/info messages |
| AppInfoSuite | `"MediaCore App Info Suite"` | 3 | Host app and version detection |
| ApplicationSettingsSuite | `"MediaCore Application Settings Suite"` | 1 | Scratch disks, project path |
| SequenceInfoSuite | `"MediaCore Sequence Info Suite"` | 9 | Sequence frame size, rate, color, HDR |
| SequenceAudioSuite | `"MediaCore Sequence Audio Suite"` | 3 | Timeline audio rendering |
| StringSuite | `"MediaCore StringSuite"` | 1 | PrSDKString to/from UTF-8/UTF-16 |
| WindowSuite | `"Premiere Window Suite"` | 1 | Main window handle, UI refresh |
| ColorManagementSuite | `"Color Management Suite"` | 1 | Color space identification and ICC profiles |

## Related Headers

- `PrSDKTimeSuite.h` -- Time types and suite
- `PrSDKMemoryManagerSuite.h` -- Memory manager
- `PrSDKErrorSuite.h` -- Error reporting
- `PrSDKAppInfoSuite.h` -- Host application info
- `PrSDKApplicationSettingsSuite.h` -- Application settings
- `PrSDKSequenceInfoSuite.h` -- Sequence properties
- `PrSDKSequenceAudioSuite.h` -- Audio rendering
- `PrSDKStringSuite.h` -- String conversion
- `PrSDKWindowSuite.h` -- Window access
- `PrSDKColorManagementSuite.h` -- Color management
- `PrSDKColorSpaces.h` -- Predefined color space tokens
- `PrSDKColorProfile.h` -- Color profile types
- `PrSDKMALErrors.h` -- Error code definitions
- `PrSDKTypes.h` -- Fundamental SDK types
