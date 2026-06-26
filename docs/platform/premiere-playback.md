# Premiere Pro Play Modules and Transmit Plugins

Premiere Pro supports two complementary plugin types for real-time output: **Play Modules** for integrated playback engines (controlling the monitor window, hardware output, and audio), and **Transmit Plugins** for streaming video and audio to external devices, monitors, or network destinations. This guide covers both architectures in detail.

**SDK Headers Covered:**
- `PrSDKPlayModule.h` -- Play module entry point, selectors, data structures, and callbacks
- `PrSDKPlayModuleAudioSuite.h` -- Audio playback (host-driven and plugin-driven modes)
- `PrSDKPlayModuleOverlaySuite.h` -- Overlay rendering during playback
- `PrSDKPlayModuleImmersiveVideoSuite.h` -- VR/360 video playback support
- `PrSDKTransmit.h` -- Transmit plugin architecture
- `PrSDKTransmitInvocationSuite.h` -- Programmatic transmit control

---

## Play Modules

### Overview

Play modules are Premiere Pro's extensible playback engine. They control how video and audio are rendered and displayed during timeline playback, scrubbing, and export-to-tape. A play module can:

- Drive the program monitor display
- Output video to external hardware (SDI cards, etc.)
- Control audio playback through the host or custom hardware
- Handle real-time segment analysis (what can play in real-time)
- Support VR/360 video viewing
- Draw overlays (safe margins, captions, etc.)

### Version History

```c
#define PLAYMOD_VERSION_70  700   // Premiere Pro 1.0
#define PLAYMOD_VERSION_75  750   // Premiere Pro 1.5
#define PLAYMOD_VERSION_80  800   // Premiere Pro 2.0
#define PLAYMOD_VERSION_90  900   // CS3
#define PLAYMOD_VERSION_100 1000  // CS4
#define PLAYMOD_VERSION_110 1100  // CS5
#define PLAYMOD_VERSION_115 1150  // CS5.5
#define PLAYMOD_VERSION_120 1200  // CS6
#define PLAYMOD_VERSION_130 1300  // CS7 (current)
```

### Entry Point

```c
typedef PREMPLUGENTRY (*PlayModEntryFunc)(
    int selector, pmStdParms* parms, void* param1, void* param2);
```

The `pmStdParms` structure provides version info, callback functions, and suite access:

```c
typedef struct {
    int              pmInterfaceVer;  // PLAYMOD_VERSION
    pmCallbackFuncs* funcs;           // Video, file, and class callbacks
    piSuitesPtr      piSuites;        // Standard suites
} pmStdParms;
```

### Selectors

Play modules respond to a selector-based dispatch model. The full selector list:

#### Lifecycle Selectors

| Selector | Description |
|----------|-------------|
| `playmod_Startup` (1) | Application startup; fill in `pmStartupRec` with GUID and display name |
| `playmod_Shutdown` (2) | Application shutdown |
| `playmod_Open` (3) | Open a single file for playback |
| `playmod_Close` (12) | Close playback and release resources |
| `playmod_NewList` (21) | Create a new cutlist (sequence playback) |
| `playmod_Activate` (62) | Activate/deactivate the player |
| `playmod_GetIndFormat` (14) | Report supported file formats (called iteratively) |
| `playmod_GetFilePrefs` (34) | Get/create preferences for a file type |
| `playmod_SetFilePrefs` (46) | Apply preferences |

#### Playback Control Selectors

| Selector | Description |
|----------|-------------|
| `playmod_Play` (55) | Start playback with `pmPlayParms` |
| `playmod_Stop` (10) | Stop playback |
| `playmod_SetPos` (56) | Seek to a position |
| `playmod_GetPos` (57) | Get current playback position and mode |
| `playmod_Step` (59) | Step forward/backward by `pmStepRec::stepDistance` |
| `playmod_Preroll` (60) | Prepare for playback (audio init happens here) |
| `playmod_PlayIdle` (11) | Called repeatedly during playback for servicing |
| `playmod_SetPlaybackSpeed` (54) | Change playback speed |
| `playmod_AllowSetPositionDuringPlayback` (78) | Query if seeking during playback is supported |

#### Display Selectors

| Selector | Description |
|----------|-------------|
| `playmod_SetDisp` (5) | Change display area |
| `playmod_Update` (6) | Redraw current frame (window was obscured) |
| `playmod_SetView` (65) | Set viewport/zoom |
| `playmod_SetVideoDisplayType` (69) | Set display mode (composite, alpha, channels) |
| `playmod_DisplayMoving` (70) | Display area is being moved |
| `playmod_DisplayChanged` (71) | Display area has finished changing |
| `playmod_SetDest` (86) | Set destination (CC 7.1+) |
| `playmod_SetBackgroundColor` (87) | Set background color (CC 7.1+) |

#### Quality and State Selectors

| Selector | Description |
|----------|-------------|
| `playmod_SetQuality` (53) | Set render quality |
| `playmod_SetDisplayStateProperties` (84) | Set display state (replaces quality + resolution) |
| `playmod_SetDisplayStateProperties2` (96) | With caption stream format (CC 13.0+) |
| `playmod_PushPlayerSettings` (76) | Receive player settings from host |
| `playmod_EnableDynamicPlayback` (77) | Enable/disable dynamic quality |
| `playmod_SetFractionalResolution` (82) | Set fractional playback resolution |

#### Frame Output Selectors

| Selector | Description |
|----------|-------------|
| `playmod_PutFrame` (49) | Display a rendered frame (VOut) |
| `playmod_PutFrameRequest` (67) | Specify preferred render format for PutFrame |
| `playmod_PutTemporaryTimeline` (83) | Display a temporary timeline (CS5+) |

#### Audio Selectors

| Selector | Description |
|----------|-------------|
| `playmod_GetAudioInfo` (74) | Report audio capabilities |
| `playmod_GetAudioChannelInfo` (75) | Report channel names |
| `playmod_AudioOutputMappingUpdate` (85) | Audio output mapping changed (CS7+) |

#### VR/360 Selectors (CC 10.0+)

| Selector | Description |
|----------|-------------|
| `playmod_GetVRSupported` (88) | Query VR support |
| `playmod_SetVRConfiguration` (89) | Configure VR projection and layout |
| `playmod_GetVRConfiguration` (90) | Get current VR config |
| `playmod_SetVREnabled` (91) | Enable/disable VR mode |
| `playmod_GetVREnabled` (92) | Query VR enabled state |
| `playmod_SetVRView` (93) | Set VR viewing direction |
| `playmod_GetVRView` (94) | Get current VR view |
| `playmod_CalculateVRDisplayDimensions` (95) | Calculate VR display dimensions |

#### Timeline Analysis Selectors

| Selector | Description |
|----------|-------------|
| `playmod_GetRTStatusForTime` (80) | Check real-time playability for a time range |
| `playmod_VideoSequenceHasChanged` (79) | Timeline has been edited |
| `playmod_EnterScrub` (63) | Entering scrub mode |
| `playmod_LeaveScrub` (64) | Leaving scrub mode |

### Startup and Format Registration

During `playmod_Startup`, fill in the `pmStartupRec`:

```c
typedef struct {
    prPluginID   outPlayerID;        // Unique GUID (required)
    prUTF16Char  outDisplayName[256];
} pmStartupRec;
```

Return `pmIsCacheable` (400) if the plugin can be lazy-initialized, or `playmod_ErrNone` (0) if it must load immediately.

During `playmod_GetIndFormat`, fill in `pmModuleInfoRec`:

```c
typedef struct {
    PrFourCC     filetype;
    PrFourCC     subtype;
    PrFourCC     classID;
    csSDK_int32  playflags;          // pmFlag_canPlayLists = 4
    int          hasSetup;            // Non-zero if has setup dialog
    int          capabilityFlags;     // PMCap* flags
    int          requestedAPIVersion;
    int          reserved[32];
} pmModuleInfoRec;
```

### Capability Flags

| Flag | Value | Description |
|------|-------|-------------|
| `PMCapPutFrame` | 0x00000080 | Supports PutFrame (video output) |
| `PMCapWillReportDroppedFrames` | 0x00000800 | Reports dropped frames during 1x playback |
| `PMCapCanLoopPlayback` | 0x00001000 | Supports looped playback |
| `PMCapCanShuttlePlayback` | 0x00002000 | Supports -4x to 4x playback speeds |
| `PMCapCanZoom` | 0x00004000 | Supports display scaling |
| `PMCapCanSafeMargin` | 0x00008000 | Can display safe area overlays |
| `PMCapCanExport` | 0x00010000 | Enables Export to Tape |
| `PMCapCanDoFractionalResolution` | 0x00080000 | Handles reduced resolution playback |
| `PMCapCanPutTemporaryTimeline` | 0x00100000 | Handles temporary timelines |
| `PMCapSupportsDisplayStateProperties` | 0x00200000 | Supports `playmod_SetDisplayStateProperties` |
| `PMCapSupportsOverlayDrawing` | 0x00400000 | Supports overlay suite |
| `PMCapSupportsImmersiveVideo` | 0x00800000 | Supports VR/immersive video |

### Real-Time Segment Status

When the host calls `playmod_GetRTStatusForTime`, the play module reports whether each time range can be played in real-time:

```c
typedef enum {
    PRT_PLAYCODE_REALTIME = 0,
    PRT_PLAYCODE_NON_REALTIME_UNSPECIFIED = 1,
    PRT_PLAYCODE_REALTIME_WITH_MISMATCH = 2,
    PRT_PLAYCODE_REALTIME_CACHED = 3
} prtPlaycode;

typedef struct {
    csSDK_int32 size;
    csSDK_int32 version;
    PrTime      inStartTime;    // Filled by host
    PrTime      outEndTime;     // Filled by plugin
    prtPlaycode playcode;       // Filled by plugin
} prtPlayableRangeRec;
```

The special value `kPrSDKEndOfTimeline` (91445760000000000LL) in `outEndTime` signals end-of-content.

### Display State Properties

Modern play modules should support `playmod_SetDisplayStateProperties`:

```c
typedef struct {
    pmPlayMode       inPlayMode;            // Stopped, Playing, Scrubbing
    PrRenderQuality  inRenderQuality;
    PrRenderQuality  inDeinterlaceQuality;
    pmFieldDisplay   inFieldDisplay;         // ShowFirstField, ShowSecondField, ShowBothFields
    csSDK_int32      inDownsampleFactor;
} pmDisplayStateProperties;
```

Version 2 adds caption stream format:

```c
typedef struct {
    // ... same as above, plus:
    PrRenderCaptionStreamFormat inCaptionStreamFormatToDisplay;
} pmDisplayStateProperties2;
```

### Player Settings

Received via `playmod_PushPlayerSettings`:

```c
typedef struct {
    int useMaximumRenderPrecision;    // Render at higher bit depth
    int mSuppressTransmit;            // Only desktop display
    int mPrimaryDisplayFullScreen;    // Full-screen display
    int mDrawTransparencyGrid;        // Show transparency grid
    int mUseEndOfSequenceIndicator;   // Draw last-frame indicator
    int mBypassEffects;               // Skip non-intrinsic effects
    int mEnableDisplayColorManagement; // Enable display color management
} pmPlayerSettings;
```

### Error Return Values

```c
typedef enum {
    playmod_ErrNone = 0,
    playmod_ErrBadFile = 1,
    playmod_ErrDriver = 2,
    playmod_ErrNotPreferred = 3,
    playmod_BadFormatIndex = 4,
    playmod_DeclinePlay = 5,           // Decline; find another module
    playmod_ListWrongType = 6,         // Cannot play; please render
    playmod_ListBadSpeed = 7,          // Cannot play at this speed
    playmod_CantAddSegment = 8,
    playmod_Unsupported = 9,
    playmod_AudioOverload = 10,
    playmod_OutOfRange = 11,
    playmod_CannotRender = 12,
    playmod_RebuildCutlist = 13,       // Not an error: rebuild needed
    playmod_CannotShiftLayer = 14,
    playmod_UnsupportedPlaybackSpeed = 16,
    playmod_BroadcastPrefs = 17,
    playmod_CannotRecord = 18,
    playmod_RenderAndPutFrame = 19,    // Premiere should render and PutFrame
} prPlaymodError;
```

---

## PrSDKPlayModuleAudioSuite

Suite name: `"Premiere Playmod Audio Suite"` (version 4)

Play modules have two audio modes: **host-based** (Premiere controls the audio hardware) and **plugin-based** (the plugin controls audio hardware and pulls audio buffers from Premiere).

### Host-Based Audio Flow

```
1. Plugin receives playmod_Preroll
2. Plugin calls InitHostAudio()
3. Plugin returns from preroll
4. Plugin receives playmod_Play
5. Plugin calls StartAudio()
6. During playback: host plays audio, plugin calls GetPosition() for timing
7. Plugin receives playmod_Stop
8. Plugin calls StopAudio()
```

```c
prSuiteError (*InitHostAudio)(
    csSDK_int32       inPlayID,
    AudioTimeCallback inTimerCallback,  // Optional high-priority timer
    void*             inInstanceData,
    int               inIsScrubbing,
    PrTime*           outClockInterval);

prSuiteError (*StartAudio)(
    csSDK_int32          inPlayID,
    const AudioPositions* inPlayPosition,
    float                inPlaybackSpeed,  // Must be non-zero
    PrTime               inMinimumPreroll);

prSuiteError (*GetPosition)(
    csSDK_int32 inPlayID,
    PrTime*     outPosition);

prSuiteError (*StopAudio)(csSDK_int32 inPlayID);
```

### Plugin-Based Audio Flow

```
1. Plugin receives playmod_Preroll
2. Plugin calls InitPluginAudio() with AudioPlaybackSettings
3. Plugin returns from preroll
4. Plugin receives playmod_Play
5. Plugin calls StartAudio()
6. During playback: plugin calls GetNextAudioBuffer() to pull audio
7. Plugin receives playmod_Stop
8. Plugin calls StopAudio()
```

```c
prSuiteError (*InitPluginAudio)(
    csSDK_int32                   inPlayID,
    int                           inIsScrubbing,
    const AudioPlaybackSettings*  inSettings);

prSuiteError (*GetNextAudioBuffer)(
    csSDK_int32   inPlayID,
    float**       inInBuffers,       // Input channel buffers
    float**       outOutBuffers,     // Output channel buffers
    unsigned int  inNumSampleFrames);
```

### Audio Playback Settings

Version 1 (basic):

```c
typedef struct {
    PrAudioChannelType outputChannelType;
    float              sampleRate;
    csSDK_uint32       maxBufferSize;     // Largest buffer in sample frames
    PrTime             inputLatency;
    PrTime             outputLatency;
} AudioPlaybackSettings;
```

Version 3 (latest, with delay compensation):

```c
typedef struct {
    csSDK_uint32 numInputChannels;
    csSDK_uint32 numOutputChannels;
    float        sampleRate;
    csSDK_uint32 maxBufferSize;
    PrTime       inputLatency;
    PrTime       outputLatency;
    PrTime       delayCompensation;  // How early to deliver audio
} AudioPlaybackSettings3;
```

Use `InitPluginAudio3` for version 3 settings.

### Playback Control

```c
prSuiteError (*SetPosition)(
    csSDK_int32  inPlayID,
    const PrTime* inRequestedPosition,
    PrTime*      outActualPosition);  // Can be NULL

prSuiteError (*SetRange)(
    csSDK_int32          inPlayID,
    const AudioPositions* inRequestedPosition,
    AudioPositions*       outActualPosition);  // Can be NULL

prSuiteError (*SetPlaybackSpeed)(
    csSDK_int32 inPlayID,
    float       inSpeed);  // Must be non-zero
```

The `AudioPositions` structure:

```c
typedef struct {
    PrTime inPosition;       // In point (must be < start and out)
    PrTime outPosition;      // Out point (must be > in and start)
    PrTime currentPosition;  // Current time (between in and out)
    prBool looping;          // Loop at boundaries
} AudioPositions;
```

### Audio Time Callback

For host-based audio, the optional timer callback runs on a high-priority audio thread:

```c
typedef void (*AudioTimeCallback)(void* inInstanceData, PrTime inCurrentTime);
```

**Warning:** This callback runs on a very high priority thread. Do not block, allocate memory, or perform lengthy operations.

---

## PrSDKPlayModuleOverlaySuite

Suite name: `"MediaCore Playmod Overlay Suite"` (version 4)

Enables play modules to render overlay graphics (safe margins, captions, guides) composited on top of the video frame. Set the `PMCapSupportsOverlayDrawing` capability flag to use this suite.

### RenderImage

```c
prSuiteError (*RenderImage)(
    PrPlayID    inPlayID,
    PrTime      inTime,
    const prRect* inLogicalRegion,      // Region of video being rendered
    int         inDisplayWidth,
    int         inDisplayHeight,
    float       inScaleFactor,          // HiDPI scaling
    prBool      inClearToTransparentBlack,
    PPixHand*   ioPPix);               // NULL = host allocates; or provide one
```

If `ioPPix` is NULL, the host allocates the PPix. If provided, it must be BGRA square pixel, sized to the display dimensions. When `inClearToTransparentBlack` is true, the frame is cleared before rendering, allowing the composite to use the whole frame rather than just visible regions.

Version 2 adds a transmit flag:

```c
prSuiteError (*RenderImage2)(
    /* ... same params ..., */
    prBool inIsTransmit,  // Drawing for transmit output
    PPixHand* ioPPix);
```

### Cache Identity

Overlays are cached. Return a unique identifier for the overlay that would be produced with given parameters:

```c
prSuiteError (*GetIdentifier)(
    PrPlayID inPlayID,
    PrTime inTime,
    const prRect* inLogicalRegion,
    int inDisplayWidth, int inDisplayHeight,
    float inScaleFactor,
    prBool inClearToTransparentBlack,
    prPluginID* outIdentifier);
```

### Visibility Queries

```c
prSuiteError (*HasVisibleRegions)(
    PrPlayID inPlayID, PrTime inTime,
    const prRect* inLogicalRegion,
    int inDisplayWidth, int inDisplayHeight,
    float inScaleFactor,
    prBool* outHasVisibleRegions);

prSuiteError (*VariesOverTime)(
    PrPlayID inPlayID,
    prBool* outVariesOverTime);
```

### Caption Detection (Version 4)

```c
prSuiteError (*IsCaptionsOverlay)(
    PrPlayID inPlayID,
    prBool* outHasCaptionsToDraw,
    PrRenderCaptionStreamFormat* outCaptionStreamFormatToDisplay);
```

---

## PrSDKPlayModuleImmersiveVideoSuite

Suite name: `"MediaCore Playmod Immersive Video Suite"` (version 2)

Supports VR/360/immersive video playback. Requires the `PMCapSupportsImmersiveVideo` capability flag.

### Player Configuration

```c
prSuiteError (*GetPlayerConfiguration)(
    PrPlayID inPlayID,
    PrIVProjectionType* outProjection,    // Equirectangular, etc.
    PrIVFrameLayout* outFrameLayout,      // Monoscopic, top/bottom, side-by-side
    csSDK_uint32* outHorizontalCapturedFOV,  // Degrees (up to 360)
    csSDK_uint32* outVerticalCapturedFOV);   // Degrees (up to 180)

// Version 2: adds ambisonics monitoring
prSuiteError (*GetPlayerConfiguration2)(
    PrPlayID inPlayID,
    PrIVProjectionType* outProjection,
    PrIVFrameLayout* outFrameLayout,
    csSDK_uint32* outHorizontalCapturedFOV,
    csSDK_uint32* outVerticalCapturedFOV,
    PrIVAmbisonicsMonitoringType* outAmbisonicsMonitoring);
```

When immersive video is not in use, returns `kPrIVProjection_None` and `kPrIVFrameLayout_Monoscopic` with 0-degree captured view.

### Head Tracking

```c
prSuiteError (*SupportsHeadTracking)(
    PrPlayID inPlayID,
    prBool   inTrackingSupported);

prSuiteError (*NotifyDirection)(
    PrPlayID inPlayID,
    float inPan,    // -180 to +180 (left/right)
    float inTilt,   // -90 to +90 (down/up)
    float inRoll);  // -180 to +180 (CCW/CW)
```

### Player Status (Version 2)

```c
prSuiteError (*GetPlayerStatus)(
    PrPlayID inPlayID,
    prBool* outViewingImmersiveVideo,
    prBool* outMonitoringAmbisonics,
    prBool* outTrackingHMD,
    float*  outPan,
    float*  outTilt,
    float*  outRoll);
```

---

## Transmit Plugins

### Overview

Transmit plugins receive rendered video and audio frames and send them to external destinations: hardware monitors, SDI outputs, streaming endpoints, or other applications. Unlike play modules which control the entire playback pipeline, transmit plugins are passive receivers that are activated/deactivated by the host.

### Entry Point

```c
#define tmEntryPointName "xTransmitEntry"

typedef tmResult (*tmEntryFunc)(
    csSDK_int32  inInterfaceVersion,  // tmInterfaceVersion (currently 4)
    prBool       inLoadModule,        // true = load, false = unload
    piSuitesPtr  piSuites,
    tmModule*    outModule);          // Fill in callback table
```

### tmModule Callback Table

```c
typedef struct {
    tmResult (*Startup)(tmStdParms*, tmPluginInfo*);
    tmResult (*Shutdown)(tmStdParms*);
    tmResult (*SetupDialog)(tmStdParms*, prParentWnd);
    tmResult (*NeedsReset)(const tmStdParms*, prBool* outResetModule);

    tmResult (*CreateInstance)(const tmStdParms*, tmInstance*);
    tmResult (*DisposeInstance)(const tmStdParms*, tmInstance*);

    tmResult (*QueryAudioMode)(const tmStdParms*, const tmInstance*, csSDK_int32 inQueryIterationIndex, tmAudioMode*);
    tmResult (*QueryVideoMode)(const tmStdParms*, const tmInstance*, csSDK_int32 inQueryIterationIndex, tmVideoMode*);

    tmResult (*ActivateDeactivate)(const tmStdParms*, const tmInstance*, PrActivationEvent, prBool inAudioActive, prBool inVideoActive);

    tmResult (*StartPlaybackClock)(const tmStdParms*, const tmInstance*, const tmPlaybackClock*);
    tmResult (*StopPlaybackClock)(const tmStdParms*, const tmInstance*);

    tmResult (*PushVideo)(const tmStdParms*, const tmInstance*, const tmPushVideo*);

    // Version 4 additions:
    tmResult (*StartPushAudio)(const tmStdParms*, const tmInstance*, PrTime, float, PrTime, PrTime, prBool, prBool, csSDK_uint32*);
    tmResult (*PushAudio)(const tmStdParms*, const tmInstance*, const tmPushAudio*);
    tmResult (*StopPushAudio)(const tmStdParms*, const tmInstance*);

    tmResult (*SetStreamingStateChangedCallback)(const tmStdParms*, void*, tmStreamingStateChangedCallback);
    tmResult (*EnableStreaming)(const tmStdParms*, prBool);
    tmResult (*IsStreamingEnabled)(const tmStdParms*, prBool*);
    tmResult (*IsStreamingActive)(const tmStdParms*, prBool*);
} tmModule;
```

### Plugin Info

Returned during `Startup`:

```c
typedef struct {
    prPluginID   outIdentifier;
    unsigned int outPriority;            // 0 = default, higher wins

    prBool       outAudioAvailable;
    prBool       outAudioDefaultEnabled;
    prBool       outClockAvailable;      // Can provide playback clock
    prBool       outVideoAvailable;
    prBool       outVideoDefaultEnabled;

    prUTF16Char  outDisplayName[256];
    prBool       outHideInUI;
    prBool       outHasSetup;            // Has a setup dialog
    csSDK_int32  outInterfaceVersion;

    prBool       outPushAudioAvailable;  // Version 4: push audio support
    prBool       outHasStreaming;         // Version 4: streaming support
} tmPluginInfo;
```

Return `tmResult_ContinueIterate` from `Startup` to register multiple transmit plugins from one module.

### Instances

Each transmit connection creates an instance:

```c
typedef struct {
    csSDK_int32      inInstanceID;
    PrTimelineID     inTimelineID;    // May be 0
    PrPlayID         inPlayID;        // May be 0

    prBool           inHasAudio;
    csSDK_uint32     inNumChannels;
    PrAudioChannelLabel inChannelLabels[16];
    PrAudioSampleType inAudioSampleType;
    float            inAudioSampleRate;

    prBool           inHasVideo;
    csSDK_int32      inVideoWidth;
    csSDK_int32      inVideoHeight;
    csSDK_int32      inVideoPARNum;
    csSDK_int32      inVideoPARDen;
    PrTime           inVideoFrameRate;
    prFieldType      inVideoFieldType;

    void*            ioPrivateInstanceData;
} tmInstance;
```

### Video and Audio Mode Negotiation

The host queries supported modes by calling `QueryVideoMode` and `QueryAudioMode` repeatedly with incrementing `inQueryIterationIndex`. Return `tmResult_Success` for each supported mode and an error when done.

```c
typedef struct {
    csSDK_int32  outWidth;          // 0 = any
    csSDK_int32  outHeight;         // 0 = any
    csSDK_int32  outPARNum;         // 0 = any
    csSDK_int32  outPARDen;         // 0 = any
    prFieldType  outFieldType;      // prFieldsAny for any
    PrPixelFormat outPixelFormat;   // PrPixelFormat_Any for any
    PrSDKString  outStreamLabel;    // {0} for normal
    PrTime       outLatency;        // Keep at or below 5 frames
    ColorSpaceRec outColorSpaceRec; // Default: BT.709 full range 32f
} tmVideoMode;

typedef struct {
    float        outAudioSampleRate;
    csSDK_uint32 outMaxBufferSize;
    csSDK_uint32 outNumChannels;
    PrAudioChannelLabel outChannelLabels[16];
    PrTime       outLatency;
    PrSDKString  outAudioOutputNames[16];  // Physical output names
} tmAudioMode;
```

**Important:** Audio output names (`PrSDKString` values in `tmAudioMode`) should be allocated by the plugin and NOT disposed by the plugin. The host takes ownership and will dispose them.

### Playback Clock

If your transmit plugin provides a clock (set `outClockAvailable = kPrTrue`), the host calls `StartPlaybackClock`:

```c
typedef struct {
    tmClockCallback inClockCallback;    // Call with time adjustments
    void*           inCallbackContext;
    PrTime          inStartTime;
    pmPlayMode      inPlayMode;
    float           inSpeed;            // 1.0 = normal, -2.0 = 2x reverse
    PrTime          inInTime;           // Informational
    PrTime          inOutTime;          // Informational
    prBool          inLoop;             // Informational
    tmDroppedFrameCallback inDroppedFrameCallback;
    PrTime          inAudioOffset;      // Host handles offset timing
    PrTime          inVideoOffset;
} tmPlaybackClock;
```

Call the clock callback with non-speed-adjusted relative time increments. `Start` may be called multiple times without `Stop` (e.g., when speed changes during playback). If video latency is specified, frames marked as `playmode_Playing` may be sent before `StartPlaybackClock`.

### Pushing Video

```c
typedef struct {
    PPixHand         inFrame;       // MUST be disposed by plugin
    PrSDKStreamLabel inStreamLabel;
} tmLabeledFrame;

typedef struct {
    PrTime               inTime;       // Negative for immediate
    pmPlayMode           inPlayMode;
    PrRenderQuality      inQuality;
    const tmLabeledFrame* inFrames;
    csSDK_size_t         inFrameCount;
} tmPushVideo;
```

**Critical:** The plugin is responsible for disposing all `PPixHand` handles passed via `PushVideo`. Failing to dispose them will leak video memory.

### Push Audio (Version 4)

For "secondary" devices that mirror audio from the primary clock device:

```c
typedef struct {
    PrTime       inTime;
    float**      inBuffers;
    csSDK_uint32 inNumSamples;
    csSDK_uint32 inNumChannels;
} tmPushAudio;
```

**Note:** `PushAudio` may be called concurrently with other API calls. This is the only transmit callback where concurrent access is allowed.

### Streaming Support (Version 4)

For network streaming plugins:

```c
tmResult (*SetStreamingStateChangedCallback)(
    const tmStdParms* inStdParms,
    void* inContext,
    tmStreamingStateChangedCallback inCallback);

tmResult (*EnableStreaming)(const tmStdParms*, prBool inEnabled);
tmResult (*IsStreamingEnabled)(const tmStdParms*, prBool* outEnabled);
tmResult (*IsStreamingActive)(const tmStdParms*, prBool* outActive);
```

Call the `tmStreamingStateChangedCallback` when the streaming state changes (connections gained/lost, enabled/disabled).

### Thread Safety

Transmit modules are single-threaded with one exception: `PushAudio` may be called concurrently with other callbacks. Design your audio buffer handling accordingly.

---

## PrSDKTransmitInvocationSuite

Suite name: `"MediaCore Transmit Invocation Suite"` (version 1)

Allows other plugins to push content to active transmit plugins programmatically.

### Known Adobe Transmit Plugin IDs

```c
#define PrTransmitDesktopAudioPluginID  "1A3A1D9F-772F-49EF-8850-402B885EF68C"
#define PrTransmitDVPluginID            "CA1A71F1-5D7F-454D-9594-5117F50E2CD3"
#define PrTransmitFullScreenPluginID    "D40EB215-EBE2-4E89-9D60-B7375F14A02D"
#define PrTransmitScopesPluginID        "6DE63FA2-47C8-56A1-8B46-E753FEA114F8"
#define PrTransmitSRTPluginID           "32163F0D-1B4C-4543-820E-545A98DEA9B5"
#define PrTransmitVRPluginID            "2E139984-4183-46FA-BC67-8CB7C47F9B14"
```

### Creating and Using Instances

```c
tmResult (*CreateInstance)(
    tmInstance* ioInstance,
    PrSDKTransmitChangedProc inTransmitChangedProc,  // Notification callback
    void* inTransmitChangedContext);

tmResult (*DisposeInstance)(tmInstance* ioInstance);

tmResult (*HasVideoDevice)(
    const tmInstance* inInstance,
    prBool* outHasVideo);

tmResult (*PushVideo)(
    const tmInstance* inInstance,
    const tmPushVideo* inPushVideo);
```

Fill in the video and audio properties of `tmInstance` before calling `CreateInstance`. The host will conform the media to the needs of the active transmit plugins.

---

## Pitfalls and Warnings

**Play module audio callback threading.** The `AudioTimeCallback` from `InitHostAudio` runs on a high-priority audio thread. Any blocking or heavy processing will cause audio dropouts.

**Transmit PPixHand ownership.** The plugin MUST dispose all `PPixHand` handles received through `PushVideo`. This is different from most other Premiere callbacks where the host manages lifetime.

**Transmit `NeedsReset` polling.** The host calls `NeedsReset` regularly on the first plugin of a module. If you set `outResetModule = true`, ALL plugins in the module will be shut down and restarted. Use this sparingly -- only for hardware state changes.

**Latency budgets.** The `tmVideoMode::outLatency` should be kept at or below 5 frames. Higher latencies cause visible A/V sync issues. The host automatically accounts for user-configured audio/video offsets by sending frames early.

**Play module GUID requirement.** A play module MUST fill in a unique GUID in `pmStartupRec::outPlayerID` during `playmod_Startup`, or it will not be loaded.

**Multiple instances.** Both play modules and transmit plugins may have multiple instances active simultaneously (e.g., source monitor and program monitor). Each instance has its own data pointer (`ioPrivatePluginData` or `ioPrivateInstanceData`).

**Version checking.** Always check `pmInterfaceVer` / `inInterfaceVersion` before accessing version-specific features. Handle graceful degradation for older host versions.
