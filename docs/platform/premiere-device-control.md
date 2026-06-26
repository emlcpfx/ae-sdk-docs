# Premiere Pro Device Control and Record Modules

Device control plugins drive tape decks, VTRs, and other hardware transport devices. Record modules handle video/audio capture from hardware sources. Together, they form Premiere Pro's hardware I/O layer for ingest workflows. A third component, the Play Module Device Control Suite, bridges playback (transmit) plugins with device control during Export to Tape operations.

## Device Control Plugins

### Overview

Device control plugins manage communication with hardware devices that have transport controls (play, stop, rewind, fast-forward, record, shuttle, jog) and timecode capability. The classic use case is RS-422 serial control of professional VTRs (tape decks), but the API is general enough for any device with transport and timecode.

**Header:** `PrSDKDevice.h`

### API Version History

| Constant | Value | Era |
|----------|-------|-----|
| `kDeviceControlAPIVersion75` | 750 | Premiere Pro 1.5 |
| `kDeviceControlAPIVersion8` | 800 | Pro 2.0 |
| `kDeviceControlAPIVersion9` | 900 | Pro 3.0 / CS3 |
| `kDeviceControlAPIVersion10` | 1000 | CS5 |
| `kDeviceControlAPIVersion105` | 1050 | CS5.5 |
| `kDeviceControlAPIVersion11` | 1100 | CS6.0.1 |
| `kDeviceControlAPIVersion12` | 1200 | CC 7.0 |
| `kDeviceControlAPIVersion13` | 1300 | CC 7.0.1 |
| `kDeviceControlAPIVersion14` | 1400 | CC 7.1 |

### Entry Point

```cpp
typedef PRDEVICEENTRY (*DeviceModEntryFunc)(short selector, DeviceHand theData);
```

The plugin exports this function. Premiere calls it with a selector and a handle to the `DeviceRec` structure.

### Selectors

| Selector | Value | Purpose |
|----------|-------|---------|
| `dsInit` | 0 | Create structures, pick operating mode. No dialogs. |
| `dsSetup` | 1 | Show user configuration dialog |
| `dsExecute` | 2 | Execute a command (see Commands below) |
| `dsCleanup` | 3 | Dispose all allocated structures |
| `dsRestart` | 4 | Re-connect device at program startup |
| `dsQuiet` | 5 | Disconnect from device, keep structures |
| `dsHasOptions` | 6 | Return `dmHasNoOptions` if no settings dialog, else `dmNoError` |

**dsRestart behavior:** Called at application startup to reconnect using saved `deviceData`. If the plugin rejects the data (e.g., incompatible version), it should return an error, and Premiere will immediately call `dsInit` instead.

### The DeviceRec Structure

The `DeviceRec` is the central data exchange structure between Premiere and the device control plugin:

```cpp
typedef struct {
    PrMemoryHandle          deviceData;            // Plugin's private data
    short                   command;               // Command to perform
    short                   mode;                  // New mode (in) or current mode (out)
    csSDK_int32             timecode;              // Timecode (in or out); -1 = N/A, -2 = blank
    short                   timeformat;            // 0 = non-drop, 1 = drop-frame
    short                   timerate;              // FPS for the timecode
    csSDK_int32             features;              // Feature bits (out for cmdGetFeatures)
    short                   error;                 // Error code (out)
    short                   preroll;               // Pre-roll frames for cmdLocate
    CallBackPtr             callback;              // Callback for cmdLocate (returns non-zero to stop)
    PauseProcPtr            PauseProc;             // Callback to pause operations
    ResumeProcPtr           ResumeProc;            // Callback to restart operations
    long                    xtimecode;             // Outpoint or duration for editing
    long                    keycode;               // Film keycode from userbits
    short                   editmode;              // Editing operation mode
    short                   exteditmode;           // Extended editing mode
    Print2TapeProcPtr       PrintProc;             // Callback when driver performs the edit
    prWnd                   parentWindow;          // Parent HWND for setup dialog (dsSetup only)
    piSuitesPtr             piSuites;              // Pointer to SDK suites
    char*                   displayName;           // Device name for Capture Window (max 255)
    TimecodeUpdatePtr       TimecodeUpdateProc;    // Reports timecode during cmdLocate
    void*                   classID;               // ClassID for timecode reporting
    long                    version;               // API version
    short                   videoStreamIsDrop;     // Drop/non-drop from video stream
    short                   autoDetectDropness;    // Let recorder determine dropness
    char*                   currentDeviceIDStr;    // Unique device identifier string
    long                    preferredScale;        // Preferred timebase scale (3.0+)
    unsigned long           preferredSampleSize;   // Preferred timebase sample size (3.0+)
    DroppedFrameCountPtr    DroppedFrameProc;      // Dropped frame count during insert edit (CS6.0.1+)
    csSDK_uint32            exportAudioChannels;   // Audio channel bitmask for export (CC 7.0+)
    csSDK_uint16            exportFlags;           // Export flags (CC 7.0+)
    csSDK_int32             delayFrames;           // Movie start delay in frames (7.0.1+)
    char                    reserved[18];
} DeviceRec;
```

### Commands (dsExecute)

When `dsExecute` is called, `DeviceRec.command` contains one of:

| Command | Value | Description |
|---------|-------|-------------|
| `cmdGetFeatures` | 0 | Return feature bits in `features` field |
| `cmdStatus` | 1 | Fill in current `mode` and `timecode`. Called repeatedly. |
| `cmdNewMode` | 2 | Change to mode specified in `mode` field |
| `cmdGoto` | 3 | Seek to `timecode` |
| `cmdLocate` | 4 | Find `timecode`, leave deck in play |
| `cmdShuttle` | 5 | Shuttle at rate in `mode` (-100 to +100) |
| `cmdEject` | 8 | Eject media |
| `cmdInsertEdit` | 10 | Driver-controlled insert edit |
| `cmdGetDeviceDisplayName` | 12 | Fill in `displayName` |
| `cmdSetDropness` | 13 | Premiere tells plugin dropness from DV stream |
| `cmdGetCurrentDeviceIdentifier` | 14 | Return unique non-localized device string |
| `cmdSetDeviceHandler` | 15 | Current handler changed (capture or edit-to-tape) |

### Feature Bits

Returned via `cmdGetFeatures` in `DeviceRec.features`:

| Flag | Value | Capability |
|------|-------|------------|
| `fBasic` | 0x00000100 | Stop, Play, Pause, FastFwd, Rewind |
| `fPositionInfo` | 0x00001000 | Returns position/timecode info |
| `fGoto` | 0x00000800 | Can seek to specific frame (requires `fPositionInfo`) |
| `fCanLocate` | 0x00000020 | Can locate a specific timecode |
| `fCanShuttle` | 0x00000008 | Supports shuttle command |
| `fReversePlay` | 0x00000040 | Supports reverse play |
| `fRecord` | 0x00002000 | Can record |
| `fStepFwd` | 0x00008000 | Can step forward one frame |
| `fStepBack` | 0x00004000 | Can step backward one frame |
| `fCanEject` | 0x00010000 | Can eject media |
| `fHasJogMode` | 0x00020000 | Device has jog capabilities |
| `fDrvrQuiet` | 0x00040000 | Driver supports quiet mode |
| `fExportDialog` | 0x00100000 | Driver has own export dialog |
| `fCanInsertEdit` | 0x00080000 | Supports insert edit command |
| `fNoTransport` | 0x00800000 | No transport modes at all |
| `fRecordImmediate` | 0x01000000 | Record immediately after seek |
| `fCanAssembleEdit` | 0x02000000 | Supports assemble record mode |
| `fCanPreviewEdit` | 0x04000000 | Supports preview |
| `fCanInsertNoUI` | 0x08000000 | Supports CC 7.0 record modes |
| `fCanUseCC` | 0x10000000 | Handles closed caption data |
| `fCanPrintToTape` | 0x20000000 | Supports print-to-tape |
| `fCanDelayMovieStart` | 0x40000000 | Supports delayed movie start |

### Device Modes

The device reports its current mode via `DeviceRec.mode`:

| Mode | Value | State |
|------|-------|-------|
| `modeStop` | 0 | Stopped |
| `modePlay` | 1 | Playing at normal speed |
| `modePlay1_5` | 2 | Playing at 1/5 speed |
| `modePlay1_10` | 3 | Playing at 1/10 speed |
| `modePause` | 4 | Paused |
| `modeFastFwd` | 5 | Fast forward |
| `modeRewind` | 6 | Rewind |
| `modeRecord` | 7 | Recording |
| `modeGoto` | 8 | Seeking |
| `modeStepFwd` | 9 | Step forward |
| `modeStepBack` | 10 | Step backward |
| `modePlayRev` | 11 | Reverse play |
| `modeTapeOut` | 14 | No media in device |
| `modeLocal` | 15 | VTR in local mode (not remote) |
| `modeRecordPause` | 16 | Record pause |
| `modePlayFastFwd` | 17 | Fast-forward playback |
| `modePlayRewind` | 18 | Rewind playback |
| `modeRecordAssemble` | 19 | Assemble record |
| `modeRecordInsert` | 20 | Insert record |

### Error Codes

| Error | Value | Meaning |
|-------|-------|---------|
| `dmNoError` | 0 | Success |
| `dmDeviceNotFound` | 1 | Device not available |
| `dmTimecodeNotFound` | 2 | Cannot read timecode |
| `dmBadTimecode` | 3 | Timecode unreliable |
| `dmCantRecord` | 4 | Unable to record |
| `dmUserAborted` | 5 | User cancelled |
| `dmLastErrorSet` | 6 | Error string set via ErrorSuite |
| `dmExportToTapeFinished` | 7 | Export to tape completed |
| `dmTapeWriteProtected` | 8 | Tape write-protected |
| `dmNoTape` | 9 | No tape in deck |
| `dmLastInfoSet` | 10 | Info string set via ErrorSuite |
| `dmLastWarningSet` | 11 | Warning string set via ErrorSuite |
| `dmHasNoOptions` | 12 | No settings dialog |
| `dmUnsupported` | -100 | Unsupported selector |
| `dmGeneralError` | -1 | Unspecified error |

### Timecode

Timecodes are stored as 32-bit integers in the `DeviceRec`. Special values:

| Value | Meaning |
|-------|---------|
| `kInvalidTimecode` (-1) | Invalid or unavailable timecode |
| -2 | Blank (no signal on tape) |

The `timeformat` field indicates 0 for non-drop frame and 1 for drop-frame. The `timerate` field indicates the frame rate (24, 25, 30).

### Export Flags

The `exportFlags` field (CC 7.0+) is a bitmask:

| Flag | Value | Meaning |
|------|-------|---------|
| `exportVideo` | 0x01 | Record video to tape (insert mode) |
| `processCCdata` | 0x02 | Read and record closed caption data |
| `previewEdit` | 0x04 | Preview mode |

### Audio Channel Selection

The `exportAudioChannels` field (CC 7.0+) uses a bitmask where bit 0 = A1, bit 1 = A2, etc. On `cmdGetFeatures`, set the bits corresponding to audio channels available on the device. On record, the bits indicate which channels to export.

### Handler Types

When receiving `cmdSetDeviceHandler`, the `mode` field contains:

| Handler | Value | Context |
|---------|-------|---------|
| `handlerCapture` | 1 | Capture/ingest |
| `handlerEditToTape` | 2 | Export to tape |

---

## Record Modules

### Overview

Record modules implement video and audio capture from hardware. They handle the capture pipeline from device initialization through frame acquisition to file writing.

**Header:** `PrSDKRecordModule.h`

### Version History

| Constant | Value | Era |
|----------|-------|-----|
| `RECMOD_VERSION_1` | 1 | 5.0 |
| `RECMOD_VERSION_4` | 4 | 7.0 / Pro 1.0 |
| `RECMOD_VERSION_5` | 5 | Pro 1.5 |
| `RECMOD_VERSION_6` | 6 | Pro 2.0 |
| `RECMOD_VERSION_9` | 9 | CS4 |
| `RECMOD_VERSION_10` | 10 | CS5 |
| `RECMOD_VERSION_12` | 12 | CS6 (current) |

### Entry Point

```cpp
typedef PREMPLUGENTRY (*RecordEntryFunc)(
    csSDK_int32  selector,
    rmStdParms*  stdparms,
    void*        param1,
    void*        param2);
```

### Standard Parameters

```cpp
typedef struct
{
    int              rmInterfaceVer;  // RECMOD_VERSION
    recCallbackFuncs *funcs;          // Class data and memory functions
    piSuitesPtr      piSuites;        // SDK suites
} rmStdParms;
```

### Selectors

| Selector | Description | param1 | param2 |
|----------|-------------|--------|--------|
| `recmod_Startup8` | Startup (Unicode). Fill in capabilities. | `recInfoRec8*` | -- |
| `recmod_Startup` | Startup (ASCII, deprecated). | `recInfoRec*` | -- |
| `recmod_Shutdown` | Final cleanup. | -- | -- |
| `recmod_Open` | Open the record module. | `char** (private storage)` | `recOpenParms*` |
| `recmod_Close` | Close the record module. | private storage | -- |
| `recmod_SetDisp` | Update display position. | private storage | `recDisplayPos*` |
| `recmod_ShowOptions` | Show settings dialog. | private storage | `recSetupParms*` |
| `recmod_PrepRecord8` | Prepare for recording. | private storage | `recCapParmsRec8*` |
| `recmod_StartRecord` | Begin capture. | private storage | `recCapturedFileInfo*` |
| `recmod_ServiceRecord` | Give time to capture. | private storage | -- |
| `recmod_StopRecord` | Stop capture. | private storage | -- |
| `recmod_CloseRecord` | Finalize capture file. | private storage | -- |
| `recmod_StillRecord` | Capture a single still frame. | private storage | `recStillCapParmsRec*` |
| `recmod_Idle` | Idle processing, return timecode. | private storage | `recGetTimecodeRec*` |
| `recmod_QueryInfo` | Respond to settings changes. | private storage | `recCapInfoRec*` |
| `recmod_GetSetupInfo8` | Get setup item info (Unicode). | private storage | `recCapSetups8*` |
| `recmod_GetAudioIndFormat` | Query supported audio formats. | `recAudioInfoRec*` | index |
| `recmod_StartSceneSearch` | Begin scene detection search. | private storage | `recSceneDetectionParmsRec*` |
| `recmod_StopSceneSearch` | Stop scene detection. | private storage | -- |
| `recmod_ServiceSceneSearch` | Service scene search. | private storage | -- |
| `recmod_DeviceStatusChanged` | Device connect/disconnect. | private storage | -- |
| `recmod_GetConnectedDeviceNames` | List connected devices. | private storage | `recConnectedDeviceListRec*` |
| `recmod_SelectCaptureDevice` | Select a capture device by index. | private storage | index |
| `recmod_GetConnectedAudioDeviceNames` | List audio devices. | private storage | `recConnectedDeviceListRec*` |
| `recmod_SelectAudioCaptureDevice` | Select audio device by index. | private storage | index |
| `recmod_GetSelectedDeviceVideoStandard` | Get video standard. | private storage | `recVideoStandard*` |

### Capabilities Record (recInfoRec8)

Filled in during `recmod_Startup8`:

```cpp
typedef struct
{
    csSDK_int32     recmodID;              // Runtime ID (do not change)
    csSDK_int32     fileType;              // AVI, MOOV, etc.
    csSDK_int32     classID;               // Legacy class ID
    int             canVideoCap;           // Can capture video
    int             canAudioCap;           // Can capture audio
    int             canStepCap;            // Can capture async frames
    int             canStillCap;           // Can capture stills
    int             canRecordLimit;        // Accepts time limits
    int             acceptsTimebase;       // Accepts arbitrary timebase
    int             acceptsBounds;         // Accepts arbitrary frame size
    int             multipleFiles;         // May capture to multiple files
    int             canSeparateVidAud;     // Separate video/audio drives
    int             canPreview;            // Can display preview
    int             wantsEvents;           // Wants OS events
    int             wantsMenuInactivate;   // Wants menu inactivation
    int             acceptsAudioSettings;  // Accepts host audio settings
    int             canCountFrames;        // Can count and stop at N frames
    int             canAbortDropped;       // Can abort on dropped frames
    int             requestedAPIVersion;   // Expected API version
    int             canGetTimecode;        // Can read timecode from stream
    int             reserved[16];
    int             activeDuringSetup;     // Stay active during setup dialog
    csSDK_int32     prefTimescale;         // Preferred timebase
    csSDK_int32     prefSamplesize;        // Preferred sample size
    csSDK_int32     minWidth, minHeight;   // Min capture dimensions
    csSDK_int32     maxWidth, maxHeight;   // Max capture dimensions
    int             prefAspect;            // 16.16 pixel aspect ratio
    csSDK_int32     prefPreviewWidth;      // Preferred preview width
    csSDK_int32     prefPreviewHeight;     // Preferred preview height
    prUTF16Char     recmodName[256];       // Display name (Unicode)
    csSDK_int32     audioOnlyFileType;     // File type for audio-only capture
    int             canSearchScenes;       // Can detect scenes for searching
    int             canCaptureScenes;      // Can capture scene boundaries
    prPluginID      outRecorderID;         // GUID identifier (required since Pro 2.0)
} recInfoRec8;
```

Return `rmIsCacheable` (value 400) instead of `rmNoErr` from `recmod_Startup8` if the plugin can be lazily initialized. This improves application startup time.

### Two Recording Models

Record modules can use one of two capture models:

**Model 1: Blocking capture in recmod_StartRecord**
```
recmod_PrepRecord8 -> recmod_StartRecord (blocks until done) -> recmod_StopRecord -> recmod_CloseRecord
```
Return `rmStatusCaptureDone` when capture completes normally, or `rmUserAbort` if cancelled.

**Model 2: Polled capture via recmod_ServiceRecord**
```
recmod_PrepRecord8 -> recmod_StartRecord (returns immediately) -> recmod_ServiceRecord (called repeatedly) -> recmod_StopRecord -> recmod_CloseRecord
```
Return `malNoError` from `recmod_StartRecord` to use this model. Premiere will repeatedly call `recmod_ServiceRecord` until capture ends.

### Capture Parameters (recCapParmsRec8)

Passed with `recmod_PrepRecord8`:

```cpp
typedef struct
{
    void*               callbackID;           // For status/preroll callbacks
    int                 stepcapture;          // 1 = step capture, 0 = streaming
    int                 capVideo;             // Capture video?
    int                 capAudio;             // Capture audio?
    int                 width, height;        // Capture dimensions
    csSDK_int32         timescale;            // Timebase numerator
    csSDK_int32         samplesize;           // Timebase denominator
    csSDK_int32         audSubtype;           // Audio compression format
    csSDK_uint32        audrate;              // Audio sample rate (Hz)
    int                 audsamplesize;        // 0 = 8-bit, 1 = 16-bit
    int                 stereo;               // 0 = mono, 1 = stereo
    char*               setup;                // Saved plugin settings
    int                 abortondrops;         // Abort if frames drop
    int                 recordlimit;          // Time limit in seconds
    recFileSpec8        thefile;              // Output file path (Unicode)
    StatusDispFunc      statFunc;             // Status callback
    PrerollFunc         prerollFunc;          // Device control preroll
    csSDK_int32         frameCount;           // V2: Frame count limit
    char                reportDrops;          // V2: Report dropped frames
    short               currate;              // V2: Deck FPS (24/25/30)
    short               timeFormat;           // V3: 0=NDF, 1=DF
    csSDK_int32         timeCode;             // V3: In-point timecode (-1=ignore)
    csSDK_int32         inHandleAmount;       // V3: Handle frames before in-point
    ReportSceneFunc     reportSceneFunc;      // Scene detection callback
    int                 captureScenes;        // User wants scene capture
    SceneCapturedFunc8  sceneCapturedFunc;    // Scene captured notification
    bool                recordImmediate;      // Record immediately after seek
    GetDeviceTimecodeFunc getDeviceTimecodeFunc; // Get device timecode
} recCapParmsRec8;
```

### Status Callback

During capture, call the status function to report progress and check for abort:

```cpp
typedef int (*StatusDispFunc)(void* callbackID, char* stattext, int framenum);
```

Returns non-zero if capture should be halted (user pressed Stop).

### Preroll Callback

Must be called just before starting actual capture to synchronize with device control:

```cpp
typedef csSDK_int32 (*PrerollFunc)(void* callbackID);
```

Returns a `prDevicemodError` to indicate why preroll failed (if it did).

### Audio Peak Metering

Record modules can send real-time audio metering data to the host (CS5+):

```cpp
typedef struct {
    float shortAmplitude;     // Current instantaneous level
    float longAmplitude;      // Averaged peak since last call
    bool  hasClipped;         // Clipping detected since last call
} AudioPeakChannelData;

typedef struct {
    csSDK_uint32         numOfUsedChannels;
    AudioPeakChannelData data[16];  // Up to 16 channels
} recAudioPeakData;

typedef void (*AudioPeakDataFunc)(void* callbackID, recAudioPeakData* inAudioPeakData);
```

The `AudioPeakDataFunc` is provided in the `recOpenParms` struct during `recmod_Open`.

### Scene Detection

Scene detection operates in two passes:

1. **Fast Scan** (`sceneSearch_FastScan`): Device plays at high speed. Plugin reports approximate scene boundary range.
2. **Slow Scan** (`sceneSearch_SlowScan`): Device plays in reverse through the range. Plugin reports exact scene edge timecodes.

```cpp
typedef struct
{
    void*           callbackID;
    ReportSceneFunc reportSceneFunc;
    int             searchingForward;
    int             searchMode;              // sceneSearch_FastScan or sceneSearch_SlowScan
    short           isDropFrame;
    csSDK_int32     earliestTimecode;        // SlowScan only: range start
    csSDK_int32     greatestTimecode;        // SlowScan only: range end
} recSceneDetectionParmsRec;
```

### Error Codes

| Error | Value | Meaning |
|-------|-------|---------|
| `rmNoErr` | 0 | Success |
| `rmUnsupported` | 1 | Unsupported selector |
| `rmAudioRecordError` | 2 | Audio recording error |
| `rmVideoRecordError` | 3 | Video recording error |
| `rmVideoDataError` | 4 | Data rate too high |
| `rmDriverError` | 5 | General driver error |
| `rmMemoryError` | 6 | Memory error |
| `rmDiskFullError` | 7 | Disk full |
| `rmDriverNotFound` | 8 | Can't connect to driver |
| `rmStatusCaptureDone` | 9 | Capture completed successfully |
| `rmCaptureLimitReached` | 10 | Module-imposed time limit reached |
| `rmPrerollAbort` | 14 | Preroll function aborted |
| `rmUserAbort` | 15 | User cancelled capture |
| `rmFileSizeLimit` | 16 | File size limit reached |
| `rmFramesDropped` | 17 | Frames dropped |
| `rmDeviceRemoved` | 18 | Device removed during capture |
| `rmDeviceNotFound` | 19 | Device not available |
| `rmCapturedNoFrames` | 20 | Captured zero frames |
| `rmEndOfScene` | 21 | End of scene detected |
| `rmNoFrameTimeout` | 22 | No frames for extended period |
| `rmRequiresCustomPrefsError` | 29 | Need custom prefs (CS3+) |
| `rmIsCacheable` | 400 | Plugin can be lazy-initialized |

---

## PrSDKPlayModuleDeviceControlSuite -- Export to Tape

**Suite ID:** `"Premiere Playmod Device Control Suite"` / Version 1

This suite enables play modules (transmit plugins) to control hardware devices during Export to Tape operations. The suite functions must be called **in order**.

```cpp
typedef struct
{
    prSuiteError (*Seek)(PlayModuleDeviceID inDeviceID);
    prSuiteError (*Arm)(PlayModuleDeviceID inDeviceID);
    prSuiteError (*Record)(PlayModuleDeviceID inDeviceID);
    prSuiteError (*Stop)(PlayModuleDeviceID inDeviceID);
} PrSDKPlayModuleDeviceControlSuite;
```

### Required Call Sequence

```
1. Seek()   -- Device seeks to the target timecode
2. Arm()    -- Device prepares to record (pre-roll)
3. Record() -- Device starts recording
4. Stop()   -- Device stops recording (also cleans up; device ID is invalid after this)
```

**Important:** `Stop()` invalidates the `PlayModuleDeviceID`. Do not make any further calls with that ID.

### Error Codes

| Code | Value | Meaning |
|------|-------|---------|
| `kPrDeviceControlResult_Success` | 0 | Success |
| `kPrDeviceControlResult_GeneralError` | -1 | General failure |
| `kPrDeviceControlResult_IllegalCallSequence` | -2 | Functions called out of order |

### Integration with Play Modules

The `PlayModuleDeviceID` is provided to the play module when Premiere initiates an Export to Tape operation. The play module is responsible for:

1. Calling `Seek` to position the tape
2. Calling `Arm` to put the deck in record-ready mode
3. Starting video/audio output
4. Calling `Record` when output is flowing
5. Calling `Stop` when output is complete

## Timecode Handling Best Practices

### Converting Timecode Integers

Device control timecodes are packed into 32-bit integers. The format depends on the `timeformat` and `timerate` fields:

```cpp
// Unpack a timecode integer (BCD-style packing varies by device)
// This is a simplified example -- actual packing depends on the device protocol
int hours   = (timecode >> 24) & 0xFF;
int minutes = (timecode >> 16) & 0xFF;
int seconds = (timecode >> 8)  & 0xFF;
int frames  = timecode & 0xFF;
```

### Drop Frame Considerations

The `autoDetectDropness` field in `DeviceRec` lets the recorder determine the drop-frame flag from the actual video stream rather than relying on the device control plugin's report. When `cmdSetDropness` is sent, `videoStreamIsDrop` contains the stream-detected value.

### Converting Between Device Timecode and PrTime

To convert a device timecode to `PrTime` for use with Premiere's time suites, you need to:

1. Determine the frame rate from `DeviceRec.timerate`
2. Convert the timecode to a frame number (accounting for drop-frame)
3. Multiply by ticks-per-frame from the Time Suite

```cpp
PrTime ticksPerFrame;
timeSuite->GetTicksPerVideoFrame(kVideoFrameRate_NTSC, &ticksPerFrame);

// Convert timecode fields to frame count
int totalFrames = hours * 3600 * 30 + minutes * 60 * 30 + seconds * 30 + frames;
// Adjust for drop-frame if timeformat == 1
if (isDropFrame)
{
    int totalMinutes = hours * 60 + minutes;
    totalFrames -= 2 * (totalMinutes - totalMinutes / 10);
}

PrTime prTime = totalFrames * ticksPerFrame;
```

## Common Pitfalls

**cmdStatus is called repeatedly.** The `cmdStatus` command fires continuously while the device control is active. Keep this handler fast. Do not block on hardware I/O -- use non-blocking reads and cache the last known state.

**Feature bits must be accurate.** Premiere uses feature bits to determine which UI elements to show. If you claim `fCanInsertEdit` but do not handle it, the UI will offer insert edit and the operation will fail.

**Preroll is mandatory.** During capture with device control, you must call the `PrerollFunc` callback just before starting the actual capture. Skipping this will cause device control synchronization to fail.

**Record module lazy initialization.** Return `rmIsCacheable` from `recmod_Startup8` whenever possible. This allows Premiere to defer loading your plugin until the user actually opens the capture panel, improving startup time.

**Device removal during capture.** Always handle `rmDeviceRemoved` gracefully. Save whatever has been captured successfully up to the point of disconnection rather than discarding it.

**Export audio channels bitmask.** The `exportAudioChannels` field is bidirectional. On `cmdGetFeatures`, set the bits for available device channels. On record commands, read the bits to know which channels the user selected for export.

## Related Headers

- `PrSDKDevice.h` -- Device control plugin definitions
- `PrSDKRecordModule.h` -- Record module definitions
- `PrSDKPlayModuleDeviceControlSuite.h` -- Play module device control
- `PrSDKPlayModule.h` -- Play module definitions
- `PrSDKTimeSuite.h` -- Time conversion
- `PrSDKErrorSuite.h` -- Error reporting to Events panel
- `PrSDKMALErrors.h` -- Error code definitions
- `PrSDKAudioSuite.h` -- Audio format types
- `PrSDKCaptionStreamFormat.h` -- Caption stream formats (for `fCanUseCC` devices)
