# Premiere Pro Captioning and Subtitles

Premiere Pro provides SDK-level access to closed captioning data through two key headers: `PrSDKCaptioningSuite.h` for reading caption data at specific times, and `PrSDKCaptionStreamFormat.h` for identifying caption stream formats. These APIs are primarily used by transmit/play modules and exporters that need to embed caption data into output streams such as SDI, HD-SDI, DV, or file-based containers.

## Supported Caption Standards

Premiere Pro supports a comprehensive set of international captioning standards:

### CEA-608 (Line 21 Captions)

The legacy North American closed captioning standard. CEA-608 data is carried in the vertical blanking interval (VBI) of analog NTSC signals, or embedded in the VANC of digital streams. Premiere supports all CEA-608 services:

| Format Enum | Value | Description |
|-------------|-------|-------------|
| `kPrRenderCaptionStreamFormat_608_CC1` | 20 | Primary Synchronous Caption Service |
| `kPrRenderCaptionStreamFormat_608_CC2` | 21 | Special Non-Synchronous Use Captions |
| `kPrRenderCaptionStreamFormat_608_CC3` | 22 | Secondary Synchronous Caption Service |
| `kPrRenderCaptionStreamFormat_608_CC4` | 23 | Special Non-Synchronous Use Captions |
| `kPrRenderCaptionStreamFormat_608_Text1` | 24 | First Text Service |
| `kPrRenderCaptionStreamFormat_608_Text2` | 25 | Second Text Service |
| `kPrRenderCaptionStreamFormat_608_Text3` | 26 | Third Text Service |
| `kPrRenderCaptionStreamFormat_608_Text4` | 27 | Fourth Text Service |
| `kPrRenderCaptionStreamFormat_608_XDS` | 28 | Extended Data Service |

### CEA-708 (Digital TV Captions)

The modern North American digital captioning standard. CEA-708 data is carried as ANC packets (SMPTE 291M) containing CDP packets (SMPTE 334-2). Premiere supports services 1-6:

| Format Enum | Value | Description |
|-------------|-------|-------------|
| `kPrRenderCaptionStreamFormat_708_Service1` | 40 | Primary Caption Service |
| `kPrRenderCaptionStreamFormat_708_Service2` | 41 | Secondary Caption Service |
| `kPrRenderCaptionStreamFormat_708_Service3` | 42 | Service 3 |
| `kPrRenderCaptionStreamFormat_708_Service4` | 43 | Service 4 |
| `kPrRenderCaptionStreamFormat_708_Service5` | 44 | Service 5 |
| `kPrRenderCaptionStreamFormat_708_Service6` | 45 | Service 6 |

### Australian Captions (OP-42 / OP-47)

| Format Enum | Value | Description |
|-------------|-------|-------------|
| `kPrRenderCaptionStreamFormat_OP_42` | 10 | OP-42 (Australian Captions) |
| `kPrRenderCaptionStreamFormat_OP_47` | 11 | OP-47 (Australian Captions, SMPTE RDD-8) |

### European Teletext (EBU N-19)

| Format Enum | Value | Description |
|-------------|-------|-------------|
| `kPrRenderCaptionStreamFormat_Teletext_Level1` | 50 | Teletext Level 1 |
| `kPrRenderCaptionStreamFormat_Teletext_Level2` | 51 | Teletext Level 2 |

### Open Captions / Subtitles

| Format Enum | Value | Description |
|-------------|-------|-------------|
| `kPrRenderCaptionStreamFormat_Open` | 1 | Open Captions (burned-in, free format) |
| `kPrRenderCaptionStreamFormat_Open_Subtitling` | 52 | Open Subtitling |
| `kPrRenderCaptionStreamFormat_Undefined` | 0 | No captions rendered |

## Caption Data Sources

Users attach caption data to timelines through two file formats:

- **SCC files** (Scenarist Caption / DVD Caption, `*.scc`) -- Each line contains a start timecode followed by CEA-608 data.
- **MCC files** (MacCaption VANC, `*.mcc`) -- Each line starts with a timecode, followed by an ANC (SMPTE 291M) packet that has been LZW-compressed as described in the file header.

## PrSDKCaptioningSuite

**Suite ID:** `"Captioning Suite"` (Version 3)

```cpp
#define kPrSDKCaptioningSuite       "Captioning Suite"
#define kPrSDKCaptioningSuiteVersion 3
```

This suite is available to play modules, transmit plugins, and exporters. All functions take a `PrTimelineID` which is passed to the plugin via the `pmNewListParms` struct during player initialization.

### Checking for Caption Data

```cpp
prSuiteError (*HasCaptionData)(
    PrTimelineID inTimelineID,
    bool*        outHasCaptionData);
```

Always check before attempting to read caption data. The return value can change at any time because the user may attach or detach caption data while the plugin is running.

### CEA-608 Caption Data

The CEA-608 functions return raw caption bytes suitable for injection into line 21 of an analog signal or the VAUX Closed Caption pack (0x65) of a DV frame.

```cpp
prSuiteError (*Get608CaptionDataMaxSize)(
    PrTimelineID inTimelineID,
    size_t*      outCaptionDataSize);

prSuiteError (*GetNext608CaptionData)(
    PrTimelineID inTimelineID,
    PrTime*      inOutCaptionTime,
    char*        outCaptionData,
    size_t*      outCaptionDataSize);

prSuiteError (*GetPrevious608CaptionData)(
    PrTimelineID inTimelineID,
    PrTime*      inOutCaptionTime,
    char*        outCaptionData,
    size_t*      outCaptionDataSize);
```

**Usage pattern:**

```cpp
// 1. Query maximum buffer size once
size_t maxSize = 0;
captioningSuite->Get608CaptionDataMaxSize(timelineID, &maxSize);

// 2. Allocate buffer
char* captionData = new char[maxSize];

// 3. Retrieve caption data at or after a given time
PrTime captionTime = currentPlaybackTime;
size_t actualSize = 0;
prSuiteError err = captioningSuite->GetNext608CaptionData(
    timelineID, &captionTime, captionData, &actualSize);

if (err == suiteError_NoError)
{
    // captionTime now holds the exact time of this caption data
    // actualSize is the actual byte count (0, 2, or 4 bytes for CEA-608)
    // Inject captionData into your output stream
}
else if (err == suiteError_NoMoreData)
{
    // No more caption data available beyond this time
}
```

**Data format:** The returned data conforms to CEA-608-E and is typically 0, 2, or 4 bytes. A 0-byte result means there is no caption event at or after the requested time but data may exist earlier.

**Bidirectional seeking:** Use `GetNext608CaptionData` during forward playback and `GetPrevious608CaptionData` during reverse. The `GetPrevious` variant returns data at or before the specified time.

### CEA-708 Caption Data

The CEA-708 functions return complete ANC packets (SMPTE 291M) that typically contain CDP packets (SMPTE 334-2). These packets can contain both CEA-608 and CEA-708 data simultaneously, plus other ancillary data such as AFD (Active Format Description).

```cpp
prSuiteError (*Get708CaptionDataMaxSize)(
    PrTimelineID inTimelineID,
    size_t*      outCaptionDataSize);

prSuiteError (*GetNext708CaptionData)(
    PrTimelineID inTimelineID,
    PrTime*      inOutCaptionTime,
    char*        outCaptionData,
    size_t*      outCaptionDataSize);

prSuiteError (*GetPrevious708CaptionData)(
    PrTimelineID inTimelineID,
    PrTime*      inOutCaptionTime,
    char*        outCaptionData,
    size_t*      outCaptionDataSize);
```

The usage pattern is identical to CEA-608. The data is written to the VANC section of SDI or HD-SDI.

### Automatic Format Conversion

Premiere Pro performs automatic conversion between CEA-608 and CEA-708:

- If the user imports an **SCC file** (CEA-608), both the 608 and 708 APIs will function. Premiere converts CEA-608 to CEA-708 on the fly.
- If the user imports an **MCC file** (CEA-708), the 608 API will first attempt to extract CEA-608 data from within the ANC packet. If extraction fails, Premiere converts CEA-708 down to CEA-608.

This means you can safely call both the 608 and 708 APIs regardless of the source format. However, some information loss is inevitable when converting 708 to 608 (708 supports more services, styling, and windowing features).

### OP-47 Caption Data (Australian Captions)

OP-47 captions use a different retrieval pattern because each frame requires **two** ANC packets (typically placed on lines 19 and 582 of the HD-SDI signal):

```cpp
prSuiteError (*GetOP47MaxPayloadSize)(
    size_t* outCaptionDataSize);

prSuiteError (*GetNextOP47CaptionData)(
    PrTimelineID inTimelineID,
    PrTime       inStartEncodeTime,
    PrTime*      inOutCaptionTime,
    bool         inSecondPacket,
    char*        outCaptionData,
    size_t*      outCaptionDataSize);
```

**Critical:** You must call `GetNextOP47CaptionData` **twice** for each frame:

```cpp
size_t maxOP47Size = 0;
captioningSuite->GetOP47MaxPayloadSize(&maxOP47Size);

char* packetData = new char[maxOP47Size];

// First packet (even running ID)
PrTime captionTime = currentFrameTime;
size_t actualSize = 0;
captioningSuite->GetNextOP47CaptionData(
    timelineID, encodeStartTime, &captionTime,
    false,  // first packet
    packetData, &actualSize);
// Write packetData to VANC line 19

// Second packet (odd running ID)
captionTime = currentFrameTime;
captioningSuite->GetNextOP47CaptionData(
    timelineID, encodeStartTime, &captionTime,
    true,   // second packet
    packetData, &actualSize);
// Write packetData to VANC line 582
```

The `inStartEncodeTime` parameter is used internally to determine the running index of the OP-47 data packets. Pass the start time of the encode operation.

### Checking for XDCAM-Embeddable Captions

For XDCAM HD workflows, you may need to know which caption formats are present before starting an export:

```cpp
prSuiteError (*HasCaptionXDCAMEmbedableData)(
    PrTimelineID inTimelineID,
    bool*        outHas708Data,
    bool*        outHasOP47Data);
```

The range-limited variant respects encode in/out points:

```cpp
prSuiteError (*HasCaptionXDCAMEmbedableDataInRange)(
    PrTimelineID inTimelineID,
    PrTime       inStartEncodeTime,
    PrTime       inEndEncodeTime,
    bool*        outHas708Data,
    bool*        outHasOP47Data);
```

`outHas708Data` will be true if any CEA-708 compatible data exists (including CEA-608 that can be encapsulated in CEA-708). `outHasOP47Data` will be true if OP-47 captions are present.

### Deprecated: GetCaptionInfo

```cpp
prSuiteError (*GetCaptionInfo)(
    PrTimelineID inTimelineID,
    PrSDKString* outFileFormat,
    PrSDKString* outCaptionUUID,
    PrSDKString* outCreationProgram,
    PrSDKString* outCreationDate,
    PrSDKString* outCreationTime);
```

This function was deprecated after CS6. Do not use it in new development.

## PrRenderCaptionStreamFormat

Defined in `PrSDKCaptionStreamFormat.h`, this enum is used by play modules and transmit plugins to declare which caption stream format they are rendering:

```cpp
typedef enum
{
    kPrRenderCaptionStreamFormat_Undefined = 0,
    kPrRenderCaptionStreamFormat_Open,
    // Australian
    kPrRenderCaptionStreamFormat_OP_42 = 10,
    kPrRenderCaptionStreamFormat_OP_47,
    // CEA-608
    kPrRenderCaptionStreamFormat_608_CC1 = 20,
    kPrRenderCaptionStreamFormat_608_CC2,
    // ... through CC4, Text1-Text4, XDS
    // CEA-708
    kPrRenderCaptionStreamFormat_708_Service1 = 40,
    // ... through Service6
    // Teletext
    kPrRenderCaptionStreamFormat_Teletext_Level1 = 50,
    kPrRenderCaptionStreamFormat_Teletext_Level2,
    kPrRenderCaptionStreamFormat_Open_Subtitling
} PrRenderCaptionStreamFormat;
```

This enum is included by `PrSDKPlayModule.h` and is available in the play module's render parameters.

## Integration with Play Modules

Play modules receive caption format information as part of their render setup. The `PrSDKPlayModule.h` header includes `PrSDKCaptionStreamFormat.h` directly. When a play module is initialized for transmit output, it can:

1. Check `HasCaptionData` to see if captions are attached to the active timeline.
2. Determine which format to use based on the output stream type (SDI, DV, file, etc.).
3. Call the appropriate `Get*CaptionData` functions during each frame render.
4. Embed the caption bytes into the output stream at the correct location.

## Integration with Exporters

Exporters that write to caption-capable containers (MXF, MOV with closed caption tracks, etc.) follow a similar pattern but typically process captions in bulk during the export loop rather than frame-by-frame during playback.

```cpp
// Typical exporter caption embedding loop
PrTime currentTime = exportStartTime;
PrTime ticksPerFrame;
timeSuite->GetTicksPerVideoFrame(videoFrameRate, &ticksPerFrame);

while (currentTime < exportEndTime)
{
    // Get 708 caption data for this frame
    PrTime captionTime = currentTime;
    size_t dataSize = 0;
    prSuiteError err = captioningSuite->GetNext708CaptionData(
        timelineID, &captionTime, captionBuffer, &dataSize);

    if (err == suiteError_NoError && captionTime == currentTime)
    {
        // There is caption data exactly at this frame time
        WriteCaptionDataToContainer(captionBuffer, dataSize, currentTime);
    }

    currentTime += ticksPerFrame;
}
```

## Playback Speed Considerations

The caption data returned by the Captioning Suite is mapped 1:1 with timecodes and is designed for **1x speed playback**. No adjustment is made for other playback speeds (slow motion, fast forward, etc.).

If you need to display captions during paused playback, you must cache the most recent caption data yourself and continue presenting it. The suite does not automatically repeat data for paused states.

## Common Pitfalls

**Buffer allocation is your responsibility.** Call `Get608CaptionDataMaxSize` or `Get708CaptionDataMaxSize` first, allocate a buffer of that size, and pass it to the retrieval functions. Allocate once and reuse.

**Time is bidirectional.** `GetNext*` returns data at or after the requested time. `GetPrevious*` returns data at or before. The actual time is written back to the `inOutCaptionTime` parameter. Always check this output time to know exactly when the caption data applies.

**OP-47 requires two calls per frame.** Forgetting the second packet call will result in corrupt running IDs and potentially garbled subtitle display on OP-47 decoders.

**Caption data can appear at any time.** Because users can attach SCC/MCC files while the plugin is running, always recheck `HasCaptionData` periodically rather than caching the result from startup.

**CEA-608 data size varies.** The returned data is 0, 2, or 4 bytes per frame. Zero bytes means no caption event at that time -- this is normal and does not indicate an error.

**suiteError_NoMoreData is not an error.** It simply means there is no more caption data in the requested direction. Handle it gracefully as a boundary condition.

## Related Headers

- `PrSDKCaptioningSuite.h` -- The captioning suite definition
- `PrSDKCaptionStreamFormat.h` -- Caption format enumeration
- `PrSDKTimeSuite.h` -- Time representation (PrTime)
- `PrSDKPlayModule.h` -- Play module integration (includes caption stream format)
- `PrSDKMALErrors.h` -- Error codes including `suiteError_NoMoreData`
