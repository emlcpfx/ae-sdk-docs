# Premiere Pro Importers: File-Based, Async, and Synthetic

This document covers the Premiere Pro importer plugin architecture, referencing the SDK 26.0 headers (`PrSDKImport.h`, `PrSDKAsyncImporter.h`, `PrSDKImporterShared.h`, `PrSDKImporterFileManagerSuite.h`). If you already know the After Effects SDK, the key difference is that Premiere importers are standalone plugin types -- they are not effects. They have their own entry point, selector dispatch, and lifecycle.

## Entry Point

An importer's entry point must be named `xImportEntry`:

```cpp
// PrSDKImport.h
typedef PREMPLUGENTRY (*ImportEntryFunc)(
    csSDK_int32 selector,
    imStdParms* stdparms,
    void* param1,
    void* param2);
```

The `imStdParms` structure provides the interface version, callback functions, and the `piSuites` pointer for acquiring suites via `SPBasicSuite`:

```cpp
typedef struct {
    csSDK_int32     imInterfaceVer;   // IMPORTMOD_VERSION (currently 24)
    imCallbackFuncs *funcs;
    piSuitesPtr     piSuites;
} imStdParms;
```

The current interface version is `IMPORTMOD_VERSION_24` (Pr 23.2).

---

## Complete Selector List

Selectors are defined in `enum PrImporterSelector`. The importer receives these as the first argument to the entry point. Here is every selector, grouped by purpose:

### Lifecycle and Initialization

| Selector | Value | Version | Description |
|---|---|---|---|
| `imInit` | 0 | 5.0 | First call. Fill `imImportInfoRec` with capabilities. |
| `imShutdown` | 1 | 5.0 | Final call before unload. Release global resources. |
| `imGetSupports8` | 36 | Pro 2.0 | Must return `malSupports8`. Version gate. |
| `imGetIndFormat` | 14 | 5.0 | Enumerate supported file formats. Called with incrementing index; return `imBadFormatIndex` to stop. |

### File Operations

| Selector | Value | Version | Description |
|---|---|---|---|
| `imOpenFile8` | 39 | Pro 2.0 | Open a file. Receive `imFileOpenRec8`. Store file handle in `privatedata`. |
| `imQuietFile` | 8 | 5.0 | Release file handles but keep private data. Called when AE/PPro needs the file handle back. |
| `imCloseFile` | 9 | 5.0 | Close file and dispose of private data. |
| `imSaveFile8` | 40 | Pro 2.0 | Save/copy file to new location. |
| `imDeleteFile8` | 41 | Pro 2.0 | Delete file(s) on disk. |
| `imCopyFile` | 51 | Pro 2.0 | Copy media file with progress callback. |

### File Information

| Selector | Value | Version | Description |
|---|---|---|---|
| `imGetInfo8` | 37 | Pro 2.0 | Fill `imFileInfoRec8` with video/audio properties. |
| `imGetInfo9` | 81 | Pro 14.1 | Extended version with reserved padding. Fills `imFileInfoRec9`. |
| `imGetTimeInfo8` | 52 | Pro 2.0 | Return timecode and reel information. |
| `imSetTimeInfo8` | 53 | Pro 2.0 | Write timecode to file. |
| `imGetFileAttributes` | 55 | Pro 3.0 | Return file creation date stamp. |
| `imAnalysis` | 12 | 5.0 | Return text analysis of the file (codec info, etc.). |
| `imDataRateAnalysis` | 13 | 5.0 | Return per-frame data rate samples. |
| `imGetSubTypeNames` | 54 | Pro 3.0 | Return display names for codec subtypes. |

### Pixel Format and Color

| Selector | Value | Version | Description |
|---|---|---|---|
| `imGetIndPixelFormat` | 27 | 7.0 | Enumerate supported pixel formats. Called with incrementing index. |
| `imGetIndColorProfile` | 58 | Pro 5.5 | Enumerate color profiles (legacy). |
| `imGetIndColorSpace` | 73 | Pro 13.0 | Enumerate media color spaces (color-managed path). |
| `imGetEmbeddedLUT` | 82 | - | Report embedded LUT in media. |
| `imGetColorSpaceFromOpaqueData` | 84 | Pro 23.3 | Convert opaque color data to color space. |

### Video Frame Retrieval

| Selector | Value | Version | Description |
|---|---|---|---|
| `imImportImage` | 5 | 5.0 | Legacy frame import. Receive `imImportImageRec`. |
| `imGetSourceVideo` | 47 | Pro 2.0 | Modern frame delivery. Receive `imSourceVideoRec`. Return a `PPixHand`. |
| `imGetPreferredFrameSize` | 45 | Pro 2.0 | Report preferred decode sizes (for sub-resolution decode). |
| `imSelectClipFrameDescriptor` | 65 | Pro 8.0 | Negotiate optimal frame format with host. |
| `imSelectClipFrameDescriptor2` | 83 | Pro 23.2 | Extended version with max bit depth info. |

### Audio

| Selector | Value | Version | Description |
|---|---|---|---|
| `imImportAudio7` | 25 | 7.0 | Import audio sample frames as float32 buffers. |
| `imResetSequentialAudio` | 29 | 7.5 | Reset sequential audio read position. |
| `imGetSequentialAudio` | 30 | 7.5 | Read next chunk of sequential audio. |
| `imGetPeakAudio` | 49 | Pro 2.0 | Provide peak audio data for waveform display. |
| `imGetAudioChannelLayout` | 64 | Pro 7.0 | Report per-channel audio labels. |

### Preferences and Setup

| Selector | Value | Version | Description |
|---|---|---|---|
| `imGetPrefs8` | 38 | Pro 2.0 | Show setup dialog; allocate/populate prefs data. |
| `imGetInstancePrefs` | 72 | Pro 12.1 | Instance-based prefs (not static). |
| `imGetSupportsPerInstancePrefs` | 71 | Pro 12.1 | Report whether `imGetInstancePrefs` is supported. |

### Async Importer

| Selector | Value | Version | Description |
|---|---|---|---|
| `imCreateAsyncImporter` | 46 | Pro 2.0 | Create an async importer instance. Fill `imAsyncImporterCreationRec`. |

### Trimming and File Management

| Selector | Value | Version | Description |
|---|---|---|---|
| `imCalcSize8` | 42 | Pro 2.0 | Calculate trimmed/untrimmed file size. |
| `imCheckTrim8` | 43 | Pro 2.0 | Validate trim points; adjust to nearest keyframe. |
| `imTrimFile8` | 44 | Pro 2.0 | Perform the actual trim operation. |
| `imRetargetAccelerator` | 50 | Pro 2.0 | Update accelerator file paths after copy. |
| `imQueryDestinationPath` | 56 | Pro 5.0 | Override suggested copy destination. |
| `imQueryContentState` | 57 | Pro 5.0 | Report content state for cache invalidation. |
| `imQueryInputFileList` | 59 | Pro 6.0 | Report all files associated with this media. |
| `imQueryStreamLabel` | 60 | Pro 6.0 | Return label for a given stream index. |

### Closed Captions

| Selector | Value | Version | Description |
|---|---|---|---|
| `imInitiateAsyncClosedCaptionScan` | 61 | Pro 7.0 | Begin caption scanning. |
| `imGetNextClosedCaption` | 62 | Pro 7.0 | Retrieve next caption entry. |
| `imCompleteAsyncClosedCaptionScan` | 63 | Pro 7.0 | Finalize caption scan. |

### Data Streams (Pro 12.1+)

| Selector | Value | Version | Description |
|---|---|---|---|
| `imGetDataStreamsInfo` | 68 | Pro 12.1 | Report number of data streams. |
| `imGetDataStream` | 69 | Pro 12.1 | Get data stream metadata. |
| `imGetDataSample` | 70 | Pro 12.1 | Get a single data sample value. |
| `imGetDataStreamChunk` | 74 | Pro 13.0 | Get data stream in chunks (MGJSON). |
| `imGetDataFileInfo` | 75 | - | Get data file information. |
| `imGetDataHierarchy` | 76 | - | Get hierarchical data stream layout. |
| `imGetStreamInfo` | 77 | - | Get detailed stream information. |
| `imGetStreamTemporalSamples` | 78 | - | Get temporal samples from a dynamic stream. |
| `imGetDataStreamVideoSyncTime` | 79 | - | Get video sync time for data stream. |

### Miscellaneous

| Selector | Value | Version | Description |
|---|---|---|---|
| `imGetMetaData` | 23 | 6.0 | Read metadata chunk by fourCC. |
| `imSetMetaData` | 24 | 6.0 | Write metadata chunk. |
| `imDeferredProcessing` | 48 | Pro 2.0 | Perform background processing (e.g., indexing). |
| `imGetRollCrawlInfo` | 32 | 7.x | For title importers -- get roll/crawl parameters. |
| `imRollCrawlRenderPage` | 33 | 7.x | Render a single page of a roll/crawl. |
| `imPerformSourceSettingsCommand` | 66 | Pro 9.0 | Handle source settings effect commands. |
| `imGetExtendedFormatInfo` | 67 | Pro 12.0.1 | Provide extended format data for exporter passthrough. |
| `imGetCurrentSystemState` | 80 | Pro 14.0.2 | Report system state for cache invalidation. |

---

## imImportInfoRec: Capability Flags

During `imInit`, you fill in the `imImportInfoRec` to declare what your importer can do:

```cpp
typedef struct {
    csSDK_uint32  importerType;          // Unique fourCC identifier
    csSDK_int32   canSave;               // Can handle "save as"
    csSDK_int32   canDelete;             // Can delete its own files
    csSDK_int32   canResize;             // Can resize on decode
    csSDK_int32   canDoSubsize;          // Can rasterize subset at any size
    csSDK_int32   canDoContinuousTime;   // Can render arbitrary frame times
    csSDK_int32   noFile;                // Synthetic importer (no file on disk)
    csSDK_int32   addToMenu;             // imMenuNone or imMenuNew
    csSDK_int32   hasSetup;              // Has a setup dialog
    csSDK_int32   dontCache;             // Frames should not be cached
    csSDK_int32   setupOnDblClk;         // Setup dialog opens on double-click
    csSDK_int32   keepLoaded;            // Never unload plugin DLL
    csSDK_int32   priority;              // Override priority (>=100 to override shipping plugins)
    csSDK_int32   canCreate;             // Synthetic/custom importer that creates files
    csSDK_int32   canCalcSizes;          // Can calculate trimmed file sizes
    csSDK_int32   canTrim;               // Can trim media files
    csSDK_int32   avoidAudioConform;     // Fast audio retrieval supported
    prUTF16Char*  acceleratorFileExt;    // Accelerator file extension (256 chars)
    csSDK_int32   canCopy;               // Can copy media files
    csSDK_int32   canSupplyMetadataClipName;
    csSDK_int32   canProvidePeakAudio;
    csSDK_int32   canProvideFileList;
    csSDK_int32   canProvideClosedCaptions;
    // ... and more
} imImportInfoRec;
```

### Example: imInit handler

```cpp
case imInit:
{
    imImportInfoRec* importInfoP = reinterpret_cast<imImportInfoRec*>(param1);
    importInfoP->importerType       = 'SDK_';
    importInfoP->canSave            = kPrFalse;
    importInfoP->canDelete          = kPrFalse;
    importInfoP->canResize          = kPrTrue;
    importInfoP->canDoSubsize       = kPrTrue;
    importInfoP->canDoContinuousTime = kPrFalse;
    importInfoP->noFile             = kPrFalse;  // file-based
    importInfoP->addToMenu          = imMenuNone;
    importInfoP->hasSetup           = kPrFalse;
    importInfoP->dontCache          = kPrFalse;
    importInfoP->priority           = 0;
    importInfoP->avoidAudioConform  = kPrTrue;
    importInfoP->canProvidePeakAudio = kPrFalse;
    result = imNoErr;
    break;
}
```

---

## imGetIndFormat: Registering File Types

Called repeatedly with incrementing index. Fill `imIndFormatRec` and return `imNoErr` for valid entries, `imBadFormatIndex` to stop:

```cpp
case imGetIndFormat:
{
    csSDK_int32 index = reinterpret_cast<csSDK_int32>(param2);
    imIndFormatRec* formatRec = reinterpret_cast<imIndFormatRec*>(param1);

    if (index == 0)
    {
        formatRec->filetype = 'SDK_';
        formatRec->flags = xfIsMovie | xfCanOpen | xfCanImport | xfUsesFiles;
        formatRec->canWriteTimecode = 0;
        formatRec->canWriteMetaData = 0;
        strcpy(formatRec->FormatName, "SDK Sample Format");
        strcpy(formatRec->FormatShortName, "SDK");
        strcpy(formatRec->PlatformExtension, "sdk\0");
        result = imNoErr;
    }
    else
    {
        result = imBadFormatIndex;
    }
    break;
}
```

### xRef Flag Values

| Flag | Value | Meaning |
|---|---|---|
| `xfCanOpen` | 0x8000 | Importer can open files |
| `xfCanImport` | 0x4000 | Importer can import files |
| `xfCanReplace` | 0x2000 | Importer can replace files |
| `xfUsesFiles` | 0x1000 | Uses file references |
| `xfIsStill` | 0x0010 | Still image format |
| `xfIsMovie` | 0x0008 | Movie format |
| `xfIsSound` | 0x0004 | Audio-only format |

---

## imGetInfo8 / imGetInfo9: File Properties

This is where you report the video and audio properties of the opened file. The struct `imFileInfoRec8` contains:

```cpp
typedef struct {
    char             hasVideo;
    char             hasAudio;
    imImageInfoRec   vidInfo;        // Video properties
    csSDK_int32      vidScale;       // numerator of timebase
    csSDK_int32      vidSampleSize;  // denominator of timebase
    csSDK_int32      vidDuration;    // Duration in timebase units
    imAudioInfoRec7  audInfo;        // Audio properties
    PrAudioSample    audDuration;    // Audio duration in sample frames
    csSDK_int32      accessModes;    // Random or sequential access
    void*            privatedata;    // Your instance data
    void*            prefs;          // Persistent settings
    // ... many more fields
    csSDK_int64      vidDurationInFrames;  // Preferred over vidDuration
} imFileInfoRec8;
```

### imImageInfoRec: Video Properties

```cpp
typedef struct {
    csSDK_int32     imageWidth;
    csSDK_int32     imageHeight;
    csSDK_int16     depth;              // Informational only (buffers are 32bpp)
    csSDK_int32     subType;            // Codec fourCC
    char            fieldType;          // prFieldsNone, prFieldsUpperFirst, etc.
    char            alphaType;          // alphaStraight, alphaNone, alphaOpaque, etc.
    matteColRec     matteColor;         // Matte color for alphaArbitrary
    char            isStill;            // Single frame, only cached once
    char            noDuration;         // No intrinsic duration (synthetic)
    csSDK_uint32    pixelAspectNum;     // PAR numerator
    csSDK_uint32    pixelAspectDen;     // PAR denominator
    csSDK_int32     importerID;         // [in] Use with suite callbacks
    csSDK_int32     supportsAsyncIO;    // Set true for async importers
    csSDK_int32     supportsGetSourceVideo; // Set true for imGetSourceVideo
    csSDK_int32     colorSpaceSupport;  // imColorSpaceSupport_None or _Fixed
    PrTime          frameRate;          // If non-zero, supercedes vidScale/vidSampleSize
    csSDK_int32     bitDepth;           // Significant bits per color component
} imImageInfoRec;
```

### Example: Setting video properties

```cpp
case imGetInfo8:
{
    imFileInfoRec8* fileInfo = reinterpret_cast<imFileInfoRec8*>(param1);

    // Video
    fileInfo->hasVideo = kPrTrue;
    fileInfo->vidInfo.imageWidth = 1920;
    fileInfo->vidInfo.imageHeight = 1080;
    fileInfo->vidInfo.depth = 24;
    fileInfo->vidInfo.subType = 'SDK_';
    fileInfo->vidInfo.fieldType = prFieldsNone;
    fileInfo->vidInfo.alphaType = alphaNone;
    fileInfo->vidInfo.pixelAspectNum = 1;
    fileInfo->vidInfo.pixelAspectDen = 1;
    fileInfo->vidInfo.isStill = kPrFalse;
    fileInfo->vidInfo.supportsAsyncIO = kPrTrue;
    fileInfo->vidInfo.supportsGetSourceVideo = kPrTrue;

    // Use PrTime-based frame rate (preferred over vidScale/vidSampleSize)
    // Get ticks per second from PrSDKTimeSuite
    PrTime ticksPerSecond = 0;
    timeSuite->GetTicksPerSecond(&ticksPerSecond);
    fileInfo->vidInfo.frameRate = ticksPerSecond / 24;  // 24fps

    fileInfo->vidDurationInFrames = 240;  // 10 seconds at 24fps

    // Audio
    fileInfo->hasAudio = kPrTrue;
    fileInfo->audInfo.numChannels = 2;
    fileInfo->audInfo.sampleRate = 48000.0f;
    fileInfo->audInfo.sampleType = kPrAudioSampleType_32BitFloat;
    fileInfo->audDuration = 480000;  // 10 seconds at 48kHz

    fileInfo->accessModes = kRandomAccessImport;

    result = imNoErr;
    break;
}
```

---

## imGetSourceVideo: Frame Delivery

This is the modern (Pro 2.0+) path for delivering video frames. You receive an `imSourceVideoRec` and return a PPix:

```cpp
typedef struct {
    void*            inPrivateData;
    csSDK_int32      currentStreamIdx;
    PrTime           inFrameTime;            // Requested frame time in ticks
    imFrameFormat*   inFrameFormats;          // Preferred formats (array), NULL = any
    csSDK_int32      inNumFrameFormats;       // Count of preferred formats
    PPixHand*        outFrame;                // [out] Your returned PPix
    void*            prefs;
    csSDK_int32      prefsSize;
    PrSDKString      selectedColorProfileName;
    PrRenderQuality  inQuality;
    imRenderContext   inRenderContext;
    PrSDKColorSpaceID opaqueColorSpaceIdentifier;  // For color-managed importers
} imSourceVideoRec;
```

### Frame delivery workflow

```cpp
case imGetSourceVideo:
{
    imSourceVideoRec* srcVideoRec = reinterpret_cast<imSourceVideoRec*>(param1);
    MyPrivateData* myData = reinterpret_cast<MyPrivateData*>(srcVideoRec->inPrivateData);

    // 1. Check requested format preferences
    PrPixelFormat requestedFormat = PrPixelFormat_BGRA_4444_8u;
    if (srcVideoRec->inFrameFormats && srcVideoRec->inNumFrameFormats > 0)
    {
        requestedFormat = srcVideoRec->inFrameFormats[0].inPixelFormat;
    }

    // 2. Create a PPix
    PPixHand outFrame = NULL;
    prRect bounds = {0, 0, myData->height, myData->width};
    ppixCreatorSuite->CreatePPix(&outFrame,
                                  PrPPixBufferAccess_WriteOnly,
                                  requestedFormat,
                                  &bounds);

    // 3. Get the pixel buffer
    char* pixels = NULL;
    ppixSuite->GetPixels(outFrame, PrPPixBufferAccess_WriteOnly, &pixels);

    csSDK_int32 rowBytes = 0;
    ppixSuite->GetRowBytes(outFrame, &rowBytes);

    // 4. Decode/generate the frame into the pixel buffer
    DecodeFrame(myData, srcVideoRec->inFrameTime, pixels, rowBytes);

    // 5. Return the frame
    *srcVideoRec->outFrame = outFrame;

    result = imNoErr;
    break;
}
```

> **Pitfall**: Unlike AE's `PF_EffectWorld` where you can write anywhere in the buffer, PPix rowbytes may include padding. Always use `GetRowBytes()` to advance scanlines, never assume `width * bytesPerPixel`.

---

## imGetIndPixelFormat: Supported Pixel Formats

Report which pixel formats your importer can produce natively. Called with incrementing index:

```cpp
case imGetIndPixelFormat:
{
    csSDK_int32 index = reinterpret_cast<csSDK_int32>(param2);
    imIndPixelFormatRec* pixelFormatRec = reinterpret_cast<imIndPixelFormatRec*>(param1);

    switch (index)
    {
        case 0:
            pixelFormatRec->outPixelFormat = PrPixelFormat_BGRA_4444_8u;
            result = imNoErr;
            break;
        case 1:
            pixelFormatRec->outPixelFormat = PrPixelFormat_BGRA_4444_32f;
            result = imNoErr;
            break;
        default:
            result = imBadFormatIndex;
            break;
    }
    break;
}
```

---

## Alpha Interpretation

Set the alpha type in `imImageInfoRec.alphaType` using the constants from `PrSDKAlphaTypes.h`:

| Constant | Value | Meaning |
|---|---|---|
| `alphaStraight` | 1 | Straight (unmatted) alpha |
| `alphaNone` | 5 | No alpha channel |
| `alphaOpaque` | 6 | Alpha channel filled to opaque |
| `alphaIgnore` | 7 | Has alpha but ignore it |

For AE developers: Premiere does not support `alphaBlackMatte`, `alphaWhiteMatte`, or `alphaArbitrary` in the same way AE does. Stick to `alphaStraight` or `alphaNone` for predictable behavior.

Additionally, use `imImageInfoRec.interpretationUncertain` flags to hint that your interpretation may need user override:

```cpp
// Flag constants
#define imPixelAspectRatioUncertain    0x1
#define imFieldTypeUncertain           0x2
#define imAlphaInfoUncertain           0x4
#define imEmbeddedColorProfileUncertain 0x8
```

---

## File-Based Importers

A file-based importer follows this lifecycle:

1. **imInit** -- Register capabilities
2. **imGetIndFormat** -- Register file types and extensions
3. **imOpenFile8** -- Open the file, create private data
4. **imGetInfo8** / **imGetInfo9** -- Report video/audio properties
5. **imGetIndPixelFormat** -- Report supported pixel formats
6. **imGetSourceVideo** -- Deliver frames on demand
7. **imImportAudio7** -- Deliver audio on demand
8. **imQuietFile** -- Release file handles (keep private data)
9. **imCloseFile** -- Dispose everything

### imOpenFile8

```cpp
case imOpenFile8:
{
    imFileOpenRec8* openRec = reinterpret_cast<imFileOpenRec8*>(param1);

    // Allocate private data
    MyPrivateData* myData = new MyPrivateData();
    myData->importerID = openRec->inImporterID;

    // Open file using platform API
    myData->fileHandle = CreateFileW(
        openRec->fileinfo.filepath,
        GENERIC_READ,
        FILE_SHARE_READ,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL);

    if (myData->fileHandle == INVALID_HANDLE_VALUE)
    {
        delete myData;
        return imFileOpenFailed;
    }

    // Parse header, populate private data...
    ParseHeader(myData);

    openRec->fileinfo.privatedata = myData;
    result = imNoErr;
    break;
}
```

### imQuietFile vs imCloseFile

`imQuietFile` is called when Premiere needs to reclaim file handles (e.g., the user opened many clips). You must close your file handle but keep your private data intact. When the file is needed again, `imOpenFile8` will be called again with the same private data.

```cpp
case imQuietFile:
{
    imFileOpenRec8* quietRec = reinterpret_cast<imFileOpenRec8*>(param1);
    MyPrivateData* myData = reinterpret_cast<MyPrivateData*>(quietRec->fileinfo.privatedata);

    if (myData && myData->fileHandle != INVALID_HANDLE_VALUE)
    {
        CloseHandle(myData->fileHandle);
        myData->fileHandle = INVALID_HANDLE_VALUE;
    }
    // Do NOT delete myData
    result = imNoErr;
    break;
}
```

---

## Synthetic Importers

Synthetic importers generate content procedurally instead of reading from a file. Set `noFile = kPrTrue` in `imInit`:

```cpp
importInfoP->noFile    = kPrTrue;
importInfoP->addToMenu = imMenuNew;   // Appear in File > New menu
importInfoP->hasSetup  = kPrTrue;     // Has a setup dialog for parameters
```

Key differences from file-based importers:
- `imOpenFile8` is still called, but the file path will be empty. You only allocate private data.
- `imGetPrefs8` is called to show a setup dialog and allocate persistent preferences.
- `imGetSourceVideo` must generate frames procedurally based on the requested frame time.
- Set `imImageInfoRec.noDuration = imNoDurationNoDefault` or `imNoDurationStillDefault` if the content has no intrinsic duration.

### Synthetic Importer imGetPrefs8

```cpp
case imGetPrefs8:
{
    imGetPrefsRec* prefsRec = reinterpret_cast<imGetPrefsRec*>(param1);

    if (prefsRec->prefs == NULL)
    {
        // First call: tell host how much memory to allocate
        prefsRec->prefsLength = sizeof(MySynthPrefs);
    }
    else
    {
        MySynthPrefs* prefs = reinterpret_cast<MySynthPrefs*>(prefsRec->prefs);
        if (prefsRec->firstTime)
        {
            // Set defaults
            prefs->width = 1920;
            prefs->height = 1080;
            prefs->color = {255, 0, 0, 255};
        }
        // Optionally show a setup dialog here
    }
    result = imNoErr;
    break;
}
```

---

## Async Importers

Async importers allow Premiere to schedule non-blocking frame reads. The standard importer creates the async importer during `imCreateAsyncImporter`:

```cpp
case imCreateAsyncImporter:
{
    imAsyncImporterCreationRec* asyncRec =
        reinterpret_cast<imAsyncImporterCreationRec*>(param1);

    // Create async private data (MUST be independent of standard importer)
    MyAsyncData* asyncData = new MyAsyncData();
    asyncData->CopyFrom(asyncRec->inPrivateData);  // Deep copy!

    asyncRec->outAsyncEntry = MyAsyncImporterEntry;
    asyncRec->outAsyncPrivateData = asyncData;

    result = imNoErr;
    break;
}
```

### Async Entry Point

```cpp
PREMPLUGENTRY MyAsyncImporterEntry(int inSelector, void* inParam)
{
    switch (inSelector)
    {
        case aiInitiateAsyncRead:
        {
            aiAsyncRequest* req = reinterpret_cast<aiAsyncRequest*>(inParam);
            // Schedule disk I/O without blocking
            ScheduleRead(req->inPrivateData, req->inSourceRec.inFrameTime);
            return aiNoError;
        }
        case aiGetFrame:
        {
            imSourceVideoRec* srcRec = reinterpret_cast<imSourceVideoRec*>(inParam);
            // Return the frame if ready, or aiFrameNotFound if not
            return GetCachedFrame(srcRec);
        }
        case aiFlush:
        {
            // Cancel all pending reads, block until complete
            FlushAllReads(inParam);
            return aiNoError;
        }
        case aiClose:
        {
            MyAsyncData* data = reinterpret_cast<MyAsyncData*>(inParam);
            delete data;
            return aiNoError;
        }
        case aiCancelAsyncRead:
            // Hint to cancel a specific read
            return aiNoError;
        case aiSelectEfficientRenderTime:
            // Snap to nearest I-frame
            return aiUnsupported;
    }
    return aiUnsupported;
}
```

### Async Importer Selectors

| Selector | Required | Description |
|---|---|---|
| `aiInitiateAsyncRead` | Yes | Schedule a non-blocking read. Return immediately. |
| `aiCancelAsyncRead` | No | Hint to cancel a pending read. |
| `aiFlush` | Yes | Cancel all pending operations. Must block until complete. |
| `aiGetFrame` | Yes | Return a previously scheduled frame (check cache). |
| `aiClose` | Yes | Dispose of async importer. Called after `aiFlush`. |
| `aiContinueToWaitForAsyncIO` | No | Query if async operation needs more time. |
| `aiSelectEfficientRenderTime` | No | Snap to efficient decode times (I-frames). |
| `aiGetMinPlaybackSpeedForSelectingEfficientTime` | No | Playback speed threshold for efficient time selection. |

> **Critical rule**: The async importer's private data MUST be fully independent of the standard importer. Their lifetimes are decoupled. The async importer must contain deep copies of all data it needs.

---

## PrSDKImporterFileManagerSuite

This suite lets importers manage file handles and trigger async file refreshes:

```cpp
#define kPrSDKImporterFileManagerSuite  "Importer File Manager Suite"
// Version 4

prSuiteError (*SetImporterStreamFileCount)(
    csSDK_uint32 inPluginID,
    csSDK_int32  inStreamIndex,
    csSDK_int32  inFileCount);

prSuiteError (*RefreshFileAsync)(const prUTF16Char* inFilePath);

prSuiteError (*GetGrowingFileRefreshInterval)(csSDK_int32* outIntervalInSeconds);

prSuiteError (*SetImporterInstanceStreamFileCount)(
    void*       inImportID,
    csSDK_int32 inFileCount);
```

Use `SetImporterStreamFileCount` to tell Premiere how many file handles you hold open between `imOpenFile8` and `imQuietFile`. This helps Premiere manage system-wide file handle limits.

---

## Trimming Support

If your importer sets `canTrim = kPrTrue` and `canCalcSizes = kPrTrue`, it will receive:

1. `imCalcSize8` -- Calculate the size of a trimmed file
2. `imCheckTrim8` -- Validate trim points (snap to keyframes)
3. `imTrimFile8` -- Actually trim the file

```cpp
case imCheckTrim8:
{
    imCheckTrimRec* trimRec = reinterpret_cast<imCheckTrimRec*>(param1);

    // Snap to nearest keyframe
    csSDK_int32 nearestKeyframe = FindNearestKeyframe(
        trimRec->privatedata, trimRec->trimIn);

    trimRec->newTrimIn = nearestKeyframe;
    trimRec->newDuration = trimRec->duration - (nearestKeyframe - trimRec->trimIn);

    result = imNoErr;
    break;
}
```

The `imTrimFile8` selector receives an `imTrimFileRec8` with a progress callback:

```cpp
typedef csSDK_int32 (*importProgressFunc)(
    csSDK_int32 partDone,
    csSDK_int32 totalToDo,
    void*       trimCallbackID);
```

Call this periodically during trim operations. It returns `imProgressAbort` (0) if the user cancelled, or `imProgressContinue` (1) to keep going.

---

## Return Values

Importers return values from `enum PrImporterReturnValue`:

| Value | Code | Meaning |
|---|---|---|
| `imNoErr` | 0 | Success |
| `imBadFile` | 2 | Not a valid file for this importer |
| `imUnsupported` | 3 | Selector not handled |
| `imMemErr` | 4 | Memory allocation failure |
| `imBadFormatIndex` | 1 | No more formats/pixel formats to enumerate |
| `imIterateStreams` | 18 | File has multiple streams (from imGetInfo) |
| `imBadStreamIndex` | 19 | No more streams |
| `imFrameNotFound` | 28 | Frame not available (async importers) |
| `imFileOpenFailed` | 31 | Could not open the file |
| `imNoCaptions` | 40 | No captions available |

---

## Pitfalls and AE Developer Notes

1. **PPix vs PF_EffectWorld**: Premiere uses `PPixHand` (handle-based) rather than AE's `PF_EffectWorld` (direct pointer). You access pixels through `PrSDKPPixSuite::GetPixels()`, never by dereferencing the handle directly.

2. **Channel order**: Premiere's native format is BGRA, not AE's ARGB. If your codec decodes to ARGB, you must either swizzle channels or declare `PrPixelFormat_ARGB_4444_8u` as your output format and let Premiere convert.

3. **Rowbytes can be negative**: The `PrSDKPPixSuite::GetRowBytes()` documentation states rowbytes "may be negative." This means the image may be stored bottom-up. Always handle this.

4. **imQuietFile is not imCloseFile**: A common mistake is freeing private data in `imQuietFile`. Only release file handles; keep your instance data alive.

5. **imGetSourceVideo vs imImportImage**: Always prefer `imGetSourceVideo` (set `supportsGetSourceVideo = kPrTrue`). The `imImportImage` path is legacy and less flexible.

6. **Thread safety**: `imGetSourceVideo` and async importer selectors can be called from multiple threads simultaneously. Your decoding pipeline must be thread-safe.

7. **Frame time is in ticks**: The `inFrameTime` in `imSourceVideoRec` is in Premiere ticks (obtained via `PrSDKTimeSuite::GetTicksPerSecond()`). Convert to frame numbers by dividing by the ticks-per-frame value.

8. **Growing files**: Set `imFileInfoRec8.mayBeGrowing = kPrTrue` for files that may still be written to (e.g., camera ingest). Premiere will periodically call `imOpenFile8`/`imGetInfo8` to refresh.
