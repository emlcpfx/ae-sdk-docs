# Premiere Pro Control Surface Plugins

Control surface plugins allow hardware mixing consoles, fader banks, transport controllers, and color grading panels to communicate bidirectionally with Premiere Pro (and Adobe Audition). The architecture uses a suite-based model where the **host** provides suites the plugin calls to read/write application state, and the **plugin** provides suites the host calls to push notifications about changes.

## Architecture Overview

Control surface plugins are built on Adobe's SweetPea2 (SPBasic) suite architecture. They are compiled as shared libraries (`.dll` on Windows, `.bundle` on macOS) and loaded at application startup.

The communication model is symmetric:

```
+---------------------+                  +---------------------+
|   Premiere Pro      |                  |   Plugin (.dll)     |
|   (Host)            |                  |   (Control Surface) |
|                     |                  |                     |
|  HostSuite -------->|  plugin calls    |                     |
|  HostMixerSuite --->|  to read/write   |                     |
|  HostTransportSuite>|  app state       |                     |
|  HostCommandSuite ->|                  |                     |
|  HostMarkerSuite -->|                  |                     |
|  HostLumetriSuite ->|                  |                     |
|                     |                  |                     |
|                     |  host calls to   |<-- PluginSuite      |
|                     |  push changes    |<-- MixerSuite       |
|                     |  to hardware     |<-- TransportSuite   |
|                     |                  |<-- CommandSuite     |
|                     |                  |<-- MarkerSuite      |
|                     |                  |<-- LumetriSuite     |
+---------------------+                  +---------------------+
```

## Opaque Reference Types

All interactions use opaque reference handles defined in `ControlSurfaceTypes.h`:

| Type | Purpose |
|------|---------|
| `ADOBESDK_ControlSurfaceHostRef` | Root handle to the host; used to obtain subsystem refs |
| `ADOBESDK_ControlSurfacePluginRef` | Root handle to the plugin instance |
| `ADOBESDK_ControlSurfaceRef` | Handle to the connected control surface |
| `ADOBESDK_ControlSurfaceHostCommandRef` | Host-side command interface |
| `ADOBESDK_ControlSurfaceHostMarkerRef` | Host-side marker interface |
| `ADOBESDK_ControlSurfaceHostMixerRef` | Host-side mixer interface |
| `ADOBESDK_ControlSurfaceHostTransportRef` | Host-side transport interface |
| `ADOBESDK_ControlSurfaceHostLumetriRef` | Host-side Lumetri color interface |
| `ADOBESDK_ControlSurfaceTransportRef` | Plugin-side transport interface |
| `ADOBESDK_ControlSurfaceMarkerRef` | Plugin-side marker interface |
| `ADOBESDK_ControlSurfaceCommandRef` | Plugin-side command interface |
| `ADOBESDK_ControlSurfaceMixerRef` | Plugin-side mixer interface |
| `ADOBESDK_ControlSurfaceLumetriRef` | Plugin-side Lumetri interface |

## Host Identification

The plugin receives a host identifier at entry, allowing it to adapt behavior:

| Constant | Value | Host |
|----------|-------|------|
| `kADOBESDK_ControlSurfaceHost_Unknown` | -1 | Unrecognized host |
| `kADOBESDK_ControlSurfaceHost_Audition` | 0 | Adobe Audition |
| `kADOBESDK_ControlSurfaceHost_PremierePro` | 1 | Adobe Premiere Pro |

Some features, such as clip mixer mode, are only supported in Premiere Pro. Always check the host identifier before using host-specific functionality.

## Entry Point and Lifecycle

### Plugin Entry Point

The plugin must export a single entry point function:

```cpp
ADOBE_CONTROLSURFACE_API SPErr EntryPoint(
    struct SPBasicSuite*                inSPBasic,
    uint32_t                            inMajorVersion,
    uint32_t                            inMinorVersion,
    ADOBESDK_ControlSurfaceHostID       inHostIdentifier,
    ADOBESDK_ControlSurfacePluginFuncs* outPluginFuncs);
```

- `inSPBasic` -- The SweetPea basic suite for acquiring other suites.
- `inMajorVersion` / `inMinorVersion` -- The host's control surface API version.
- `inHostIdentifier` -- Either `kADOBESDK_ControlSurfaceHost_PremierePro` or `kADOBESDK_ControlSurfaceHost_Audition`.
- `outPluginFuncs` -- Fill in the `ADOBESDK_ControlSurfacePluginFuncs` callbacks.

The `ADOBE_CONTROLSURFACE_API` macro resolves to `__declspec(dllexport)` on Windows and `__attribute__((visibility("default")))` on macOS.

### Plugin Factory Functions

The entry point populates an `ADOBESDK_ControlSurfacePluginFuncs` struct:

```cpp
typedef struct
{
    SPErr (*Startup)();
    SPErr (*Shutdown)();
    SPErr (*CreatePluginInstance)(ADOBESDK_ControlSurfacePluginRef* outPluginInstanceRef);
    SPErr (*DeletePluginInstance)(ADOBESDK_ControlSurfacePluginRef inPluginInstanceRef);
    SPErr (*GetSuiteList)(SPSuiteListRef* outSuiteListRef);
} ADOBESDK_ControlSurfacePluginFuncs;
```

| Function | When Called | Purpose |
|----------|------------|---------|
| `Startup` | Once at load | Global initialization (open serial ports, scan for devices) |
| `Shutdown` | Once at unload | Global cleanup |
| `CreatePluginInstance` | When host creates a control surface entry | Allocate per-instance state |
| `DeletePluginInstance` | When host removes the control surface | Free per-instance state |
| `GetSuiteList` | After `Startup` | Return the SPSuiteListRef containing all plugin-side suites |

### Lifecycle Sequence

```
1. Host loads DLL, calls EntryPoint()
2. Host calls Startup()
3. Host calls GetSuiteList() to discover which plugin suites are available
4. Host calls CreatePluginInstance()
5. Host calls ControlSurfacePluginSuite::Connect() -- plugin receives HostRef
6. ... normal operation (Update cycles, state changes) ...
7. Host calls ControlSurfacePluginSuite::Suspend() when app loses focus
8. Host calls ControlSurfacePluginSuite::Resume() when app regains focus
9. Host calls ControlSurfacePluginSuite::Disconnect()
10. Host calls DeletePluginInstance()
11. Host calls Shutdown()
```

## ControlSurfacePluginSuite (Plugin Implements)

**Suite ID:** `"ADOBESDK ControlSurfacePlugin Suite"` (Version 1)

This is the core suite the plugin must implement:

```cpp
typedef struct
{
    SPErr (*Connect)(
        ADOBESDK_ControlSurfacePluginRef  inPluginRef,
        ADOBESDK_ControlSurfaceHostRef    inHostRef,
        ADOBESDK_ControlSurfaceRef*       outControlSurfaceRef);

    SPErr (*Disconnect)(ADOBESDK_ControlSurfacePluginRef inPluginRef);

    SPErr (*GetPluginID)(
        ADOBESDK_ControlSurfacePluginRef  inPluginRef,
        ADOBESDK_String*                  outPluginID);

    SPErr (*GetPluginDisplayString)(
        ADOBESDK_ControlSurfacePluginRef  inPluginRef,
        ADOBESDK_String*                  outDisplayString);

    SPErr (*GetPluginSettings)(
        ADOBESDK_ControlSurfacePluginRef  inPluginRef,
        ADOBESDK_String*                  outSettingsString);

    SPErr (*SetPluginSettings)(
        ADOBESDK_ControlSurfacePluginRef  inPluginRef,
        const ADOBESDK_String*            inSettings);

    SPErr (*HasSettingsDialog)(
        ADOBESDK_ControlSurfacePluginRef  inPluginRef,
        ADOBESDK_Boolean*                 outHasSettings);

    SPErr (*RunSettingsDialog)(
        ADOBESDK_ControlSurfacePluginRef  inPluginRef,
        void*                             inParentWindow,
        ADOBESDK_Boolean*                 outSettingsChanged);

    SPErr (*Suspend)(ADOBESDK_ControlSurfacePluginRef inPluginRef);
    SPErr (*Resume)(ADOBESDK_ControlSurfacePluginRef inPluginRef);
} ADOBESDK_ControlSurfacePluginSuite1;
```

Key points:

- **Connect** is where you receive the `HostRef` -- store it for later use with host suites. Return the `ControlSurfaceRef` so the host can call the ControlSurfaceSuite on you.
- **GetPluginSettings / SetPluginSettings** serialize state as strings for persistence in Premiere's preferences. This is how settings survive across sessions.
- **RunSettingsDialog** receives `NSWindow*` on macOS and `HWND` on Windows via the `inParentWindow` parameter.
- **Suspend / Resume** are called when the application loses and regains focus. Use these to quiet MIDI output or pause hardware polling.

## ControlSurfaceSuite (Plugin Implements)

**Suite ID:** `"ADOBESDK ControlSurface Suite"` (Version 2)

This suite provides the host access to the plugin's subsystem refs and an update tick:

```cpp
typedef struct
{
    SPErr (*GetConfigIdentifier)(
        ADOBESDK_ControlSurfaceRef  inControlSurfaceRef,
        ADOBESDK_String*            outConfigIdentifier);

    SPErr (*GetControlSurfaceFlags)(
        ADOBESDK_ControlSurfaceRef  inControlSurfaceRef,
        uint32_t*                   outControlSurfaceFlags);

    SPErr (*GetTransportRef)(
        ADOBESDK_ControlSurfaceRef              inControlSurfaceRef,
        ADOBESDK_ControlSurfaceTransportRef*    outTransportRef);

    SPErr (*GetMixerRef)(
        ADOBESDK_ControlSurfaceRef          inControlSurfaceRef,
        ADOBESDK_ControlSurfaceMixerRef*    outMixerRef);

    SPErr (*GetMarkerRef)(
        ADOBESDK_ControlSurfaceRef          inControlSurfaceRef,
        ADOBESDK_ControlSurfaceMarkerRef*   outMarkerRef);

    SPErr (*GetCommandRef)(
        ADOBESDK_ControlSurfaceRef          inControlSurfaceRef,
        ADOBESDK_ControlSurfaceCommandRef*  outCommandRef);

    SPErr (*Update)(ADOBESDK_ControlSurfaceRef inControlSurfaceRef);

    // New in version 2
    SPErr (*GetLumetriRef)(
        ADOBESDK_ControlSurfaceRef          inControlSurfaceRef,
        ADOBESDK_ControlSurfaceLumetriRef*  outLumetriRef);
} ADOBESDK_ControlSurfaceSuite2;
```

**Control Surface Flags:**

| Flag | Value | Meaning |
|------|-------|---------|
| `kADOBESDK_ControlSurfaceFlag_CanDisplayUnicode` | 1 | Surface LCD can show Unicode text |

When the flag is not set, the host will convert strings to ASCII before passing them to the plugin.

The `Update()` function is called periodically by the host. Use it to poll hardware (read MIDI, USB HID, etc.) and push any pending state changes back to the host.

## Host Suites (Plugin Calls Into Host)

### ControlSurfaceHostSuite

**Suite ID:** `"ADOBESDK ControlSurfaceHost Suite"` (Version 2)

The gateway suite. From the `HostRef` received during `Connect`, obtain refs for each subsystem:

```cpp
typedef struct
{
    SPErr (*GetCommandRef)(ADOBESDK_ControlSurfaceHostRef inHostRef,
                           ADOBESDK_ControlSurfaceHostCommandRef* outCommandRef);
    SPErr (*GetMarkerRef)(ADOBESDK_ControlSurfaceHostRef inHostRef,
                          ADOBESDK_ControlSurfaceHostMarkerRef* outMarkerRef);
    SPErr (*GetMixerRef)(ADOBESDK_ControlSurfaceHostRef inHostRef,
                         ADOBESDK_ControlSurfaceHostMixerRef* outMixerRef);
    SPErr (*GetTransportRef)(ADOBESDK_ControlSurfaceHostRef inHostRef,
                             ADOBESDK_ControlSurfaceHostTransportRef* outTransportRef);
    // Version 2
    SPErr (*GetLumetriRef)(ADOBESDK_ControlSurfaceHostRef inHostRef,
                           ADOBESDK_ControlSurfaceHostLumetriRef* outLumetriRef);
} ADOBESDK_ControlSurfaceHostSuite2;
```

### ControlSurfaceHostTransportSuite

**Suite ID:** `"ADOBESDK ControlSurfaceHost Transport Suite"` (Version 1)

Controls timeline playback from hardware:

| Function | Description |
|----------|-------------|
| `Play()` | Start playback |
| `Stop()` | Stop playback |
| `Pause()` | Pause playback |
| `Record()` | Start recording |
| `BeginRewind()` / `EndRewind()` | Press/release rewind |
| `BeginForward()` / `EndForward()` | Press/release fast forward |
| `ToggleCycle()` | Toggle loop playback |
| `GotoPrevious()` / `GotoNext()` | Jump to previous/next point of interest |
| `GotoStart()` | Jump to sequence start |
| `ShuttleLeft()` / `ShuttleStop()` / `ShuttleRight()` | J/K/L shuttle controls |
| `JogDial(inTicksMoved, inTicksPerRevolution, inShuttle)` | Jog wheel input |
| `GetTimeDisplayString(outMode, maxLen, outString)` | Current timecode as display string |
| `GetRecordState(outState)` | Is recording active? |
| `GetStopState(outState)` | Is stopped? |
| `GetPlayState(outState)` | Is playing? |
| `GetSoloState(outState)` | Any channel soloed? (rude solo) |

**Jog dial detail:** `inTicksMoved` can be negative (reverse). `inTicksPerRevolution` describes the physical resolution of the jog wheel. When `inShuttle` is true, the host should shuttle rather than scrub the CTI.

### ControlSurfaceHostMixerSuite

**Suite ID:** `"ADOBESDK ControlSurfaceHost Mixer Suite"` (Version 1)

The most extensive host suite, providing full mixer control:

#### Channel Management

```cpp
SPErr (*SetClipMixerMode)(ref, inClipMixerMode);     // Premiere Pro only
SPErr (*GetClipMixerMode)(ref, outClipMixerMode);     // Premiere Pro only
SPErr (*GetChannelCount)(ref, outChannelCount);
SPErr (*GetChannelIDForIndex)(ref, inIndex, outChannelID);
SPErr (*GetChannelIndexForID)(ref, inID, outIndex);    // Can be expensive
SPErr (*GetMasterChannelID)(ref, outMasterChannelID);
SPErr (*GetChannelName)(ref, inChannelID, inMaxLength, outName);
```

Channels are accessed by **ID** (`ADOBESDK_ControlSurfaceChannelID`), not by index. The ID is stable across mixer layout changes; the index is not. Use `GetChannelIDForIndex` for bank/channel strip mapping.

#### Channel State (Mute, Solo, Record, etc.)

```cpp
SPErr (*SetChannelState)(ref, channelID, stateIndex, state);
SPErr (*GetChannelState)(ref, channelID, stateIndex, outState);
SPErr (*GetChannelFlags)(ref, channelID, outFlags);
```

**Channel State Indices:**

| Constant | Value | Meaning |
|----------|-------|---------|
| `kADOBESDK_ControlSurfaceChannelStateIndex_Mute` | 0 | Track mute |
| `kADOBESDK_ControlSurfaceChannelStateIndex_Solo` | 1 | Track solo |
| `kADOBESDK_ControlSurfaceChannelStateIndex_Record` | 2 | Record arm |
| `kADOBESDK_ControlSurfaceChannelStateIndex_InputMonitor` | 3 | Input monitor |
| `kADOBESDK_ControlSurfaceChannelStateIndex_InvertPhase` | 4 | Phase invert |
| `kADOBESDK_ControlSurfaceChannelStateIndex_PreRender` | 5 | Pre-render |

**Channel Flags** (bitfield from `GetChannelFlags`):

| Flag | Bit | Meaning |
|------|-----|---------|
| `kADOBESDK_ControlSurfaceChannelFlag_HasRecord` | 0 | Channel supports record arm |
| `kADOBESDK_ControlSurfaceChannelFlag_HasSolo` | 1 | Supports solo |
| `kADOBESDK_ControlSurfaceChannelFlag_HasMute` | 2 | Supports mute |
| `kADOBESDK_ControlSurfaceChannelFlag_HasInputMonitor` | 3 | Supports input monitor |
| `kADOBESDK_ControlSurfaceChannelFlag_HasInvertPhase` | 4 | Supports phase invert |
| `kADOBESDK_ControlSurfaceChannelFlag_HasPreRender` | 5 | Supports pre-render |
| `kADOBESDK_ControlSurfaceChannelFlag_HasInserts` | 6 | Has insert slots |
| `kADOBESDK_ControlSurfaceChannelFlag_HasSends` | 7 | Has send slots |
| `kADOBESDK_ControlSurfaceChannelFlag_HasPan` | 8 | Has pan control |
| `kADOBESDK_ControlSurfaceChannelFlag_IsAudioClip` | 16 | Is an audio clip |
| `kADOBESDK_ControlSurfaceChannelFlag_IsBus` | 17 | Is a bus/submix |
| `kADOBESDK_ControlSurfaceChannelFlag_IsMetronome` | 18 | Is the metronome channel |
| `kADOBESDK_ControlSurfaceChannelFlag_IsMaster` | 19 | Is the master channel |

#### Fader, Pan, and Metering

```cpp
SPErr (*SetChannelFaderValue)(ref, channelID, inVolumeInDB);
SPErr (*GetChannelFaderValue)(ref, channelID, outVolumeInDB);
SPErr (*GetChannelNumAudioChannels)(ref, channelID, outNumAudioChannels);
SPErr (*GetChannelMeterValues)(ref, channelID, numChannels, outValues);
SPErr (*GetChannelPanType)(ref, channelID, outPanType);
```

Fader values are in **decibels** (float). Meter values are also floats, returned as an array sized to the number of audio channels (e.g., 6 for 5.1 surround).

**Pan types:**

| Constant | Value | Parameters |
|----------|-------|------------|
| `kADOBESDK_ControlSurfacePanType_None` | 0 | No pan control |
| `kADOBESDK_ControlSurfacePanType_Stereo` | 1 | Single pan knob (parameter 0 = Pan) |
| `kADOBESDK_ControlSurfacePanType_MonoTo51` | 2 | RadAngle, Radius, CenterLevel, LFELevel |
| `kADOBESDK_ControlSurfacePanType_StereoTo51` | 3 | Adds Spread to MonoTo51 |

#### Component Parameters (FX Inserts, Sends, EQ)

Components are addressed by `kADOBESDK_ControlSurfaceComponentIndex_*` values:

| Range | Component |
|-------|-----------|
| 0 | Fader |
| 1 | Pan |
| 2 | EQ |
| 101-116 | Insert slots 1-16 |
| 201-216 | Send slots 1-16 |
| 301-316 | Send Pan slots 1-16 |

```cpp
SPErr (*HasComponent)(ref, channelID, componentIndex, outHasComponent);
SPErr (*GetComponentName)(ref, channelID, componentIndex, maxLen, outName);
SPErr (*SetComponentBypass)(ref, channelID, componentIndex, inBypass);
SPErr (*GetComponentBypass)(ref, channelID, componentIndex, outBypass);
SPErr (*GetComponentParameterCount)(ref, channelID, componentIndex, outCount);
SPErr (*GetComponentParameterName)(ref, channelID, componentIndex, paramIndex, maxLen, outName);
SPErr (*GetComponentParameterDisplay)(ref, channelID, componentIndex, paramIndex, maxLen, outDisplay);
SPErr (*SetComponentParameterValue)(ref, channelID, componentIndex, paramIndex, inValue);  // 0..1 normalized
SPErr (*GetComponentParameterValue)(ref, channelID, componentIndex, paramIndex, outValue);
SPErr (*ComponentParameterTouched)(ref, channelID, componentIndex, paramIndex, inDownState);
SPErr (*ListenToComponentParameterChanges)(ref, channelID, componentIndex, inListen);
SPErr (*SpinComponentParameterValue)(ref, channelID, componentIndex, paramIndex, inAmount);
SPErr (*ShowComponentUI)(ref, channelID, componentIndex, inShow);
SPErr (*IsComponentUIShown)(ref, channelID, componentIndex, outIsShown);
```

**Important:** Component parameter values are **normalized 0.0 to 1.0**. The host maps to the actual effect parameter range internally. Use `SpinComponentParameterValue` for rotary encoders -- it accepts a signed percentage delta rather than an absolute value.

Call `ComponentParameterTouched(... true)` when the user grabs a fader and `ComponentParameterTouched(... false)` on release. This is critical for automation modes (Touch, Latch) to work correctly.

#### Automation and Track Controls

```cpp
SPErr (*SetAutomationMode)(ref, channelID, inAutomationMode);
SPErr (*GetAutomationMode)(ref, channelID, outAutomationMode);
SPErr (*SetTrackControlsMode)(ref, inTrackControlsMode);
SPErr (*GetTrackControlsMode)(ref, outTrackControlsMode);
```

**Automation modes:**

| Constant | Value |
|----------|-------|
| `kADOBESDK_ControlSurfaceAutomationMode_Off` | 0 |
| `kADOBESDK_ControlSurfaceAutomationMode_Read` | 1 |
| `kADOBESDK_ControlSurfaceAutomationMode_Latch` | 2 |
| `kADOBESDK_ControlSurfaceAutomationMode_Touch` | 3 |
| `kADOBESDK_ControlSurfaceAutomationMode_Write` | 4 |

**Track controls modes:**

| Constant | Value |
|----------|-------|
| `kADOBESDK_ControlSurfaceTrackControlsMode_IO` | 0 |
| `kADOBESDK_ControlSurfaceTrackControlsMode_FX` | 1 |
| `kADOBESDK_ControlSurfaceTrackControlsMode_Sends` | 2 |
| `kADOBESDK_ControlSurfaceTrackControlsMode_EQ` | 3 |

### ControlSurfaceHostCommandSuite

**Suite ID:** `"ADOBESDK ControlSurfaceHost Command Suite"` (Version 1)

Execute host keyboard shortcuts and navigate the editor:

```cpp
SPErr (*Navigate)(ref, inKeyDown, inDeltaX, inDeltaY, inZoom);
SPErr (*GetCommandCount)(ref, outCommandCount);
SPErr (*GetCommand)(ref, inIndex, outContextID, outContextName, outCommandID, outCommandName);
SPErr (*ExecuteCommand)(ref, inContextID, inCommandID);
```

- `Navigate` drives cursor-key style navigation. When `inZoom` is true, the host zooms instead of moving the edit focus.
- Commands are identified by a context/command ID pair. Pass `NULL` for `inContextID` to execute in the global context.
- Enumerate commands at startup to build a mapping table for assignable buttons on the hardware.

### ControlSurfaceHostMarkerSuite

**Suite ID:** `"ADOBESDK ControlSurfaceHost Marker Suite"` (Version 1)

```cpp
SPErr (*GetMarkerCount)(ref, outMarkerCount);
SPErr (*GetMarkerName)(ref, inMarkerIndex, inMaxLength, outMarkerName);
SPErr (*GotoMarker)(ref, inMarkerIndex);
```

### ControlSurfaceHostLumetriSuite

**Suite ID:** `"ADOBESDK ControlSurfaceHost Lumetri Suite"` (Version 1)

Added in version 2 of the control surface API. Provides direct control over the Lumetri Color panel parameters:

```cpp
SPErr (*GetParameterCount)(ref, outParameterCount);
SPErr (*GetParameter)(ref, inIndex, outParameterID, outParameterName, outMinValue, outMaxValue);
SPErr (*GetParameterValue)(ref, inParameterID, outValue);
SPErr (*ChangeParameterValue)(ref, inParameterID, inDelta, outNewValue);
SPErr (*ResetParameter)(ref, inParameterID, outNewValue);
SPErr (*GetMenuCount)(ref, outMenuCount);
SPErr (*GetMenu)(ref, inMenuIndex, outMenuID, outMenuName);
SPErr (*GetMenuValue)(ref, inMenuID, outMenuValue);
SPErr (*ChangeMenuValue)(ref, inMenuID, inDelta, outNewValue);
SPErr (*ResetMenu)(ref, inMenuID, outNewValue);
SPErr (*SetMode)(ref, inDesiredMode, outNewMode);
```

**Lumetri Panel Modes:**

| Constant | Value | Panel Section |
|----------|-------|--------------|
| `kADOBESDK_ControlSurfaceLumetriPanelMode_Hidden` | 0 | Panel hidden |
| `kADOBESDK_ControlSurfaceLumetriPanelMode_BasicCorrection` | 1 | Basic Correction |
| `kADOBESDK_ControlSurfaceLumetriPanelMode_Creative` | 2 | Creative |
| `kADOBESDK_ControlSurfaceLumetriPanelMode_Curves` | 3 | Curves |
| `kADOBESDK_ControlSurfaceLumetriPanelMode_ColorWheels` | 4 | Color Wheels & Match |
| `kADOBESDK_ControlSurfaceLumetriPanelMode_HSLSecondary` | 5 | HSL Secondary |
| `kADOBESDK_ControlSurfaceLumetriPanelMode_Vignette` | 6 | Vignette |

Note that `SetMode` may not set the exact mode you requested -- always check `outNewMode` for the actual result.

Use `ChangeParameterValue` with a signed delta rather than setting absolute values. This gives natural rotary-encoder behavior and respects parameter acceleration curves. Use `ResetParameter` to return a parameter to its default.

## Plugin-Side Notification Suites (Host Calls Into Plugin)

### ControlSurfaceMixerSuite (Plugin Implements)

**Suite ID:** `"ADOBESDK ControlSurface Mixer Suite"` (Version 2)

The host calls these to notify the plugin of mixer state changes:

```cpp
SPErr (*GetMaxChannelStripCount)(ref, outMaxCount);
SPErr (*ChannelConfigChanged)(ref, channelID, inSelector);
SPErr (*SelectionChanged)(ref);
SPErr (*SetChannelState)(ref, channelID, stateIndex, inState);
SPErr (*SetAutomationMode)(ref, channelID, inAutomationMode);
SPErr (*SetChannelOffset)(ref, inChannelOffset);
SPErr (*SetChannelCount)(ref, inChannelCount);
SPErr (*SetRudeSoloState)(ref, inValue);
SPErr (*SetComponentBypass)(ref, channelID, componentIndex, inBypass);
SPErr (*SetComponentParameter)(ref, channelID, componentIndex, paramIndex, inValue);
// Version 2:
SPErr (*AddMasterFaderAsRegularChannel)(ref, outValue);
```

**`GetMaxChannelStripCount`**: Return the number of physical fader strips on your hardware. Return `kADOBESDK_ControlSurfaceChannelChannelCount_Indeterminate` (0xFFFFFFFF) if the plugin manages its own banking internally, or `kADOBESDK_ControlSurfaceChannelChannelCount_None` (0) if there are no fader strips.

**`ChannelConfigChanged` selectors:**

| Constant | Value | Meaning |
|----------|-------|---------|
| `kADOBESDK_ControlSurfaceChannelConfig_Name` | 0 | Channel name changed |
| `kADOBESDK_ControlSurfaceChannelConfig_InsertRack` | 1 | Insert rack changed |
| `kADOBESDK_ControlSurfaceChannelConfig_SendRack` | 2 | Send rack changed |
| `kADOBESDK_ControlSurfaceChannelConfig_TrackInput` | 3 | Track input changed |
| `kADOBESDK_ControlSurfaceChannelConfig_TrackOutput` | 4 | Track output changed |

**`AddMasterFaderAsRegularChannel`** (Version 2): Return `true` if your hardware does not have a dedicated master fader and the host should include it in the regular channel strip bank.

### ControlSurfaceTransportSuite (Plugin Implements)

**Suite ID:** `"ADOBESDK ControlSurface Transport Suite"` (Version 1)

The host pushes transport state to update LEDs and displays:

```cpp
SPErr (*SetStopState)(ref, inState);
SPErr (*SetPlayState)(ref, inState);
SPErr (*SetRecordState)(ref, inState);
SPErr (*SetRewindState)(ref, inState);
SPErr (*SetForwardState)(ref, inState);
SPErr (*SetCycleState)(ref, inState);
SPErr (*SetPauseState)(ref, inState);
SPErr (*SetTimeDisplay)(ref, inTimeDisplayMode, inDisplayString);
```

**Time display modes:**

| Constant | Value | Format |
|----------|-------|--------|
| `kADOBESDK_ControlSurfaceTimeDisplayMode_None` | 0 | No display |
| `kADOBESDK_ControlSurfaceTimeDisplayMode_Decimal` | 1 | h:m:s.f or m:s.f |
| `kADOBESDK_ControlSurfaceTimeDisplayMode_SMPTE` | 2 | hh:mm:ss:ff or hh;mm;ss;ff |
| `kADOBESDK_ControlSurfaceTimeDisplayMode_Samples` | 3 | Audio sample count |
| `kADOBESDK_ControlSurfaceTimeDisplayMode_BarsAndBeats` | 4 | bar:beat.subdivision |

### ControlSurfaceCommandSuite (Plugin Implements)

**Suite ID:** `"ADOBESDK ControlSurface Command Suite"` (Version 1)

Exposes assignable user buttons:

```cpp
SPErr (*GetUserButtonCount)(ref, outButtonCount);
SPErr (*GetUserButtonIDAtIndex)(ref, inIndex, outButtonID);
SPErr (*GetUserButtonDisplayName)(ref, inButtonID, outDisplayName);
SPErr (*SetUserButtonCommand)(ref, inButtonID, inContextID, inCommandID);
SPErr (*GetUserButtonCommand)(ref, inButtonID, outContextID, outCommandID);
```

The host UI presents these buttons to the user, who can assign any host command to them.

### ControlSurfaceMarkerSuite (Plugin Implements)

**Suite ID:** `"ADOBESDK ControlSurface Marker Suite"` (Version 1)

```cpp
SPErr (*MarkerChanged)(ref, inMarkerIndex);
SPErr (*RebuildMarkers)(ref);
```

### ControlSurfaceLumetriSuite (Plugin Implements)

**Suite ID:** `"ADOBESDK ControlSurface Lumetri Suite"` (Version 1)

```cpp
SPErr (*ModeChanged)(ref, inMode);
SPErr (*ParameterValueChanged)(ref, inParameterID, inParameterValue);
SPErr (*MenuValueChanged)(ref, inMenuID, inDisplayString);
SPErr (*AllChanged)(ref);
```

## Using the C++ Wrapper Classes

The SDK provides base classes in `adobesdk/controlsurface/plugin/wrapper/` that simplify implementation. Rather than implementing raw C suite structs, derive from:

| Base Class | Wraps |
|------------|-------|
| `adobesdk::ControlSurfaceBase` | `ControlSurfaceSuite` |
| `adobesdk::ControlSurfaceMixerBase` | `ControlSurfaceMixerSuite` |
| `adobesdk::ControlSurfaceTransportBase` | `ControlSurfaceTransportSuite` |
| `adobesdk::ControlSurfaceCommandBase` | `ControlSurfaceCommandSuite` |
| `adobesdk::ControlSurfaceMarkerBase` | `ControlSurfaceMarkerSuite` |
| `adobesdk::ControlSurfaceLumetriBase` | `ControlSurfaceLumetriSuite` |

Each base class registers its suite automatically via `RegisterSuite()`. Override the virtual methods you need:

```cpp
class MyControlSurface : public adobesdk::ControlSurfaceBase
{
public:
    SPErr GetConfigIdentifier(ADOBESDK_String* outConfigIdentifier) const override;
    SPErr GetControlSurfaceFlags(uint32_t* outFlags) const override;
    SPErr GetTransportRef(ADOBESDK_ControlSurfaceTransportRef* outRef) override;
    SPErr GetMixerRef(ADOBESDK_ControlSurfaceMixerRef* outRef) override;
    SPErr Update() override;
};
```

## Implementation Example

A minimal entry point:

```cpp
#include <adobesdk/controlsurface/plugin/ControlSurfacePluginSuite.h>

static SPBasicSuite* sBasicSuite = nullptr;
static ADOBESDK_ControlSurfaceHostID sHostID = kADOBESDK_ControlSurfaceHost_Unknown;

// Your plugin instance (simplified)
static MyControlSurfacePlugin* sPluginInstance = nullptr;

static SPErr PluginStartup()
{
    // Initialize hardware communication (e.g., open MIDI port)
    return kSPNoError;
}

static SPErr PluginShutdown()
{
    // Close hardware communication
    return kSPNoError;
}

static SPErr CreatePluginInstance(ADOBESDK_ControlSurfacePluginRef* outRef)
{
    sPluginInstance = new MyControlSurfacePlugin();
    *outRef = reinterpret_cast<ADOBESDK_ControlSurfacePluginRef>(sPluginInstance);
    return kSPNoError;
}

static SPErr DeletePluginInstance(ADOBESDK_ControlSurfacePluginRef inRef)
{
    delete reinterpret_cast<MyControlSurfacePlugin*>(inRef);
    sPluginInstance = nullptr;
    return kSPNoError;
}

static SPErr GetSuiteList(SPSuiteListRef* outSuiteListRef)
{
    // Register your plugin-side suites here
    // ...
    return kSPNoError;
}

extern "C" ADOBE_CONTROLSURFACE_API SPErr EntryPoint(
    SPBasicSuite*                       inSPBasic,
    uint32_t                            inMajorVersion,
    uint32_t                            inMinorVersion,
    ADOBESDK_ControlSurfaceHostID       inHostIdentifier,
    ADOBESDK_ControlSurfacePluginFuncs* outPluginFuncs)
{
    sBasicSuite = inSPBasic;
    sHostID = inHostIdentifier;

    outPluginFuncs->Startup = PluginStartup;
    outPluginFuncs->Shutdown = PluginShutdown;
    outPluginFuncs->CreatePluginInstance = CreatePluginInstance;
    outPluginFuncs->DeletePluginInstance = DeletePluginInstance;
    outPluginFuncs->GetSuiteList = GetSuiteList;

    return kSPNoError;
}
```

## Common Pitfalls

**Channel IDs are not indices.** Never assume a channel ID equals its display position. Always use `GetChannelIDForIndex` and `GetChannelIndexForID` for conversions. Note that `GetChannelIndexForID` is documented as potentially expensive.

**Normalized vs. absolute values.** Component parameter values in the mixer suite are normalized 0.0-1.0, but Lumetri parameters use actual values with min/max ranges. Fader values are in decibels. Do not confuse these three scales.

**Thread safety.** The `Update()` call and notification suites may be called from different threads. Guard shared state (hardware I/O buffers, cached mixer state) with appropriate synchronization.

**Suspend/Resume.** Always honor these callbacks. If your hardware has motorized faders or active displays, quiet them during Suspend to avoid confusing the user when Premiere is not in focus.

**Banking arithmetic.** When implementing channel banking with `AddChannelOffset`, remember that the offset is cumulative (it adds to the current offset). Your `SetChannelOffset` notification callback receives the absolute offset after the host computes it.

**Lumetri suite availability.** The Lumetri suite was added in version 2. If connecting to an older host, `GetLumetriRef` may not be available. Check the API version from the entry point parameters before using it.

**String memory management.** Strings returned through `ADOBESDK_String` must be managed with the `ADOBESDK_StringSuite`. Always dispose strings you receive from the host, and allocate strings you return using the suite's `AllocateFromUTF8` or `AllocateFromUTF16`.

## Related Headers

- `adobesdk/controlsurface/ControlSurfaceTypes.h` -- All type definitions and enums
- `adobesdk/controlsurface/host/ControlSurfaceHost*.h` -- Host suites
- `adobesdk/controlsurface/plugin/ControlSurface*.h` -- Plugin suites
- `adobesdk/controlsurface/plugin/wrapper/` -- C++ base class wrappers
- `adobesdk/AdobesdkStringSuite.h` -- String allocation and disposal
- `SPBasic.h`, `SPSuites.h` -- SweetPea2 foundation
