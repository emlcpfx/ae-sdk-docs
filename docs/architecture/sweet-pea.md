# Sweet Pea (SP) Plugin Framework: The Foundation of AE Plugins

## Overview

Sweet Pea -- internally codenamed "PICA" -- is Adobe's cross-application plugin manager framework. It is the lowest-level plugin infrastructure shared across After Effects, Photoshop, Illustrator, and other Adobe applications. Every After Effects plugin, whether it is an effect (PF_*), an AEGP, or an I/O plugin, ultimately lives inside the Sweet Pea runtime. The higher-level APIs you interact with daily (`PF_InData`, `AEGP_SuiteHandler`, etc.) are all built on top of SP.

Understanding Sweet Pea is not merely academic. When suite acquisition fails, when a plugin fails to load, or when you see cryptic four-character-code errors like `'S!Fd'` or `'!Acq'`, you are encountering the SP layer directly. Knowing how it works lets you debug these problems in minutes rather than hours.

### Key SP Headers

| Header | Purpose |
|--------|---------|
| `SP/SPBasic.h` | Core `SPBasicSuite` -- suite acquisition, memory allocation |
| `SP/SPSuites.h` | Suite list management (`SPSuitesSuite`) |
| `SP/SPPlugs.h` | Plugin lifecycle and management (`SPPluginsSuite`) |
| `SP/SPAccess.h` | Plugin loading, unloading, and inter-plugin calls (`SPAccessSuite`) |
| `SP/SPTypes.h` | Fundamental types: `SPErr`, `SPBoolean`, `SPAPI` |
| `SP/SPConfig.h` | Platform detection: `MAC_ENV`, `WIN_ENV` |
| `SP/SPErrorCodes.h` | All SP error code definitions |

---

## SPBasicSuite: The Core of Everything

The `SPBasicSuite` is the single most important structure in the entire plugin ecosystem. It is passed to every plugin entry point, embedded in every message structure, and available at all times. It is defined in `SP/SPBasic.h`.

```c
// SP/SPBasic.h
#define kSPBasicSuite       "SP Basic Suite"
#define kSPBasicSuiteVersion 4

typedef struct SPBasicSuite {
    SPAPI SPErr (*AcquireSuite)(const char *name, int32 version, const void **suite);
    SPAPI SPErr (*ReleaseSuite)(const char *name, int32 version);
    SPAPI SPBoolean (*IsEqual)(const char *token1, const char *token2);
    SPAPI SPErr (*AllocateBlock)(size_t size, void **block);
    SPAPI SPErr (*FreeBlock)(void *block);
    SPAPI SPErr (*ReallocateBlock)(void *block, size_t newSize, void **newblock);
    SPAPI SPErr (*Undefined)(void);
} SPBasicSuite;
```

### Function Reference

#### AcquireSuite

```c
SPAPI SPErr (*AcquireSuite)(const char *name, int32 version, const void **suite);
```

Acquires a function suite by name and version. This is the function you call (directly or indirectly) every time you need access to any AE functionality. It loads the suite provider if necessary and increments the suite's reference count.

**Parameters:**
- `name` -- The suite name string (e.g., `kPFWorldSuite`, `kAEGPCompSuite`)
- `version` -- The suite version number (e.g., `kPFWorldSuiteVersion2`)
- `suite` -- Output pointer that receives the suite function pointer table

**Returns:** `kSPNoError` (0) on success, or an error code.

**Example:**
```c
PF_EffectWorld *sWorld = NULL;
SPErr err = in_data->pica_basicP->AcquireSuite(
    kPFWorldSuite,
    kPFWorldSuiteVersion2,
    (const void **)&sWorld);

if (err == kSPNoError && sWorld) {
    // Use suite functions
    sWorld->PF_NewWorld(/* ... */);
}
```

#### ReleaseSuite

```c
SPAPI SPErr (*ReleaseSuite)(const char *name, int32 version);
```

Decrements the reference count of a previously acquired suite. When the count reaches 0, the suite may be unloaded. Every `AcquireSuite` must be balanced with a `ReleaseSuite`.

**Example:**
```c
in_data->pica_basicP->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2);
```

#### IsEqual

```c
SPAPI SPBoolean (*IsEqual)(const char *token1, const char *token2);
```

Compares two strings for equality. Used internally by PICA for suite name matching, but also available to plugins. Returns non-zero if the strings are identical, 0 otherwise.

#### AllocateBlock / FreeBlock / ReallocateBlock

```c
SPAPI SPErr (*AllocateBlock)(size_t size, void **block);
SPAPI SPErr (*FreeBlock)(void *block);
SPAPI SPErr (*ReallocateBlock)(void *block, size_t newSize, void **newblock);
```

PICA-managed memory allocation. These provide host-tracked memory that is properly accounted for and cleaned up. Use these for memory that must persist across calls or that you want the host to be aware of.

**Example:**
```c
void *buffer = NULL;
SPErr err = in_data->pica_basicP->AllocateBlock(1024, &buffer);
if (err == kSPNoError && buffer) {
    // Use buffer
    memset(buffer, 0, 1024);

    // When done:
    in_data->pica_basicP->FreeBlock(buffer);
}
```

> **Pitfall:** `ReallocateBlock` may return a different pointer in `newblock`. Always use the output pointer after reallocation, not the original.

#### Undefined

```c
SPAPI SPErr (*Undefined)(void);
```

A stub function whose address is used as a sentinel. When a suite is unloaded, the host replaces all of its function pointers with the address of `Undefined`. If a plugin mistakenly calls a function on a released suite, it calls `Undefined` instead of crashing on a dangling pointer.

This is a defensive mechanism. Plugins that export suites should use it when handling unload messages:

```c
SPErr UnloadMySuite(MySuite *mySuite, SPAccessMessage *message) {
    mySuite->DoSomething = (void *)message->d.basic->Undefined;
    mySuite->DoOther     = (void *)message->d.basic->Undefined;
    return kSPNoError;
}

SPErr ReloadMySuite(MySuite *mySuite, SPAccessMessage *message) {
    mySuite->DoSomething = ActualDoSomething;
    mySuite->DoOther     = ActualDoOther;
    return kSPNoError;
}
```

---

## How SPBasicSuite Reaches Your Plugin

The `SPBasicSuite` pointer arrives at your plugin through the message data structures. Every SP message includes it in the `SPMessageData` base. For After Effects effect plugins, it is available as `in_data->pica_basicP`.

### For Effect Plugins (PF_Cmd_* handlers)

```c
PF_Err MyPlugin(
    PF_Cmd       cmd,
    PF_InData    *in_data,
    PF_OutData   *out_data,
    PF_ParamDef  *params[],
    PF_LayerDef  *output,
    void         *extra)
{
    // SPBasicSuite is always here:
    SPBasicSuite *pica = in_data->pica_basicP;

    // Acquire any suite you need:
    PF_WorldSuite2 *wsP = NULL;
    pica->AcquireSuite(kPFWorldSuite, kPFWorldSuiteVersion2, (const void **)&wsP);
    // ...
    pica->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2);
}
```

### For AEGP Plugins

```c
// In your AEGP entry point, SPBasicSuite is passed directly:
A_Err MyAEGP_Entry(
    SPBasicSuite    *pica_basicP,
    A_long          major_version,
    A_long          minor_version,
    AEGP_PluginID   aegp_plugin_id,
    AEGP_GlobalRefcon *global_refconP)
{
    // Use pica_basicP directly
    AEGP_CompSuite11 *csP = NULL;
    pica_basicP->AcquireSuite(kAEGPCompSuite, kAEGPCompSuiteVersion11, (const void **)&csP);
    // ...
}
```

---

## SPSuitesSuite: Low-Level Suite Management

Defined in `SP/SPSuites.h`, the `SPSuitesSuite` provides deeper control over suite management than `SPBasicSuite`. Most plugins never need this directly, but it is useful for plugins that export their own suites or need to inspect suite metadata.

```c
#define kSPSuitesSuite          "SP Suites Suite"
#define kSPSuitesSuiteVersion   2
```

### Key Types

```c
typedef struct SPSuite *SPSuiteRef;
typedef struct SPSuiteList *SPSuiteListRef;
typedef struct SPSuiteListIterator *SPSuiteListIteratorRef;

// The global runtime suite list (pass NULL to functions that take a list):
#define kSPRuntimeSuiteList ((SPSuiteListRef)NULL)
```

### Key Functions

| Function | Purpose |
|----------|---------|
| `AllocateSuiteList` | Create a custom suite list |
| `FreeSuiteList` | Free a custom suite list |
| `AddSuite` | Register a new suite (name + version + function pointers) |
| `AcquireSuite` | Acquire with explicit list and internal version |
| `ReleaseSuite` | Release with explicit list and internal version |
| `FindSuite` | Look up a suite without acquiring it |
| `NewSuiteListIterator` / `NextSuite` / `DeleteSuiteListIterator` | Iterate all suites in a list |
| `GetSuiteHost` | Which plugin provides this suite |
| `GetSuiteName` | Get suite name string |
| `GetSuiteAPIVersion` | Get public version number |
| `GetSuiteInternalVersion` | Get internal version number |
| `GetSuiteProcs` | Get the raw function pointer table |
| `GetSuiteAcquireCount` | Current reference count |

### Exporting a Custom Suite

If your plugin provides functionality to other plugins, you can register a suite:

```c
// Define your suite
typedef struct {
    SPAPI SPErr (*MyFunction)(A_long param, A_long *result);
    SPAPI SPErr (*MyOtherFunction)(const char *name);
} MySuite1;

#define kMySuite          "My Custom Suite"
#define kMySuiteVersion1  1

// Register it during startup
SPSuitesSuite *suitesSuite = NULL;
pica->AcquireSuite(kSPSuitesSuite, kSPSuitesSuiteVersion, (const void **)&suitesSuite);

MySuite1 mySuiteImpl = { MyFunctionImpl, MyOtherFunctionImpl };
SPSuiteRef suiteRef = NULL;

suitesSuite->AddSuite(
    kSPRuntimeSuiteList,  // global list
    myPluginRef,          // this plugin
    kMySuite,             // name
    kMySuiteVersion1,     // public version
    kSPLatestInternalVersion, // internal version
    &mySuiteImpl,         // function pointers
    &suiteRef);           // output reference

pica->ReleaseSuite(kSPSuitesSuite, kSPSuitesSuiteVersion);
```

---

## SPPluginsSuite: Plugin Lifecycle Management

Defined in `SP/SPPlugs.h`, this suite manages plugin objects -- their state, properties, globals, and metadata. There are two versions of this suite: `SPPluginsSuite` (version 4/5, uses `SPPlatformFileSpecification`) and `SPXPlatPluginsSuite` (version 6, uses `XPlatFileSpec`).

```c
#define kSPPluginsSuite           "SP Plug-ins Suite"
#define kSPPluginsSuiteVersion4   4
#define kSPPluginsSuiteVersion5   5
#define kSPPluginsSuiteVersion6   6
```

### Key Types

```c
typedef struct SPPlugin *SPPluginRef;
typedef struct SPPluginList *SPPluginListRef;
typedef struct SPPluginListIterator *SPPluginListIteratorRef;

// Plugin entry point signature:
typedef SPAPI SPErr (*SPPluginEntryFunc)(const char *caller, const char *selector, void *message);
```

The entry point signature is crucial: all SP plugins receive messages as a `(caller, selector, message)` triple. The caller identifies the subsystem making the call, the selector identifies the specific event, and the message provides event-specific data.

### Plugin State Management

| Function | Purpose |
|----------|---------|
| `GetPluginGlobals` / `SetPluginGlobals` | Store/retrieve plugin global data across load/unload cycles |
| `GetPluginStarted` / `SetPluginStarted` | Track whether the plugin has completed initialization |
| `GetPluginBroken` / `SetPluginBroken` | Mark a plugin as broken (unavailable due to error) |
| `GetPluginSkipShutdown` / `SetPluginSkipShutdown` | Control shutdown behavior |
| `GetPluginName` / `SetPluginName` | Plugin display name |
| `GetNamedPlugin` | Look up a plugin by name |
| `GetPluginPropertyList` | Access PiPL properties at runtime |
| `FindPluginProperty` | Query a specific PiPL property |

### Plugin Globals Pattern

Plugins can store a global data pointer that persists across load/unload cycles. PICA saves this pointer when the plugin is unloaded and restores it when reloaded:

```c
typedef struct {
    A_long       initialized;
    AEGP_PluginID plugin_id;
    // ... other persistent state
} MyGlobals;

// During startup:
MyGlobals *gP = NULL;
SPPluginsSuite *plugSuite = NULL;
pica->AcquireSuite(kSPPluginsSuite, kSPPluginsSuiteVersion4, (const void **)&plugSuite);

plugSuite->GetPluginGlobals(myPluginRef, (void **)&gP);
if (!gP) {
    pica->AllocateBlock(sizeof(MyGlobals), (void **)&gP);
    memset(gP, 0, sizeof(MyGlobals));
    plugSuite->SetPluginGlobals(myPluginRef, gP);
}

pica->ReleaseSuite(kSPPluginsSuite, kSPPluginsSuiteVersion4);
```

---

## SPAccessSuite: Plugin Loading and Inter-Plugin Communication

Defined in `SP/SPAccess.h`, this suite controls the actual loading and unloading of plugin code, and provides the mechanism for one plugin to call another directly.

```c
#define kSPAccessSuite         "SP Access Suite"
#define kSPAccessSuiteVersion  3
```

### Access Messages

When PICA loads or unloads a plugin, it sends access messages:

| Caller | Selector | When |
|--------|----------|------|
| `kSPAccessCaller` ("SP Access") | `kSPAccessReloadSelector` ("Reload") | Plugin has just been loaded into memory |
| `kSPAccessCaller` ("SP Access") | `kSPAccessUnloadSelector` ("Unload") | Plugin is about to be unloaded from memory |

### Access Points

The `SPAccessMessage` includes an `SPAccessPoint` enum telling you the context:

```c
typedef enum {
    kStartup = 0,   // Initial load at application startup (with Reload selector)
    kRuntime,        // Loaded on-demand while app is running (with Reload selector)
    kShutdown,       // Normal unload (with Unload selector)
    kTerminal        // App is shutting down; plugin has non-zero access count
} SPAccessPoint;
```

### Access Reference Counting

Plugins are reference-counted through `SPAccessRef` objects:

```c
typedef struct SPAccess *SPAccessRef;
```

| Function | Purpose |
|----------|---------|
| `AcquirePlugin` | Load plugin if needed, increment reference count |
| `ReleasePlugin` | Decrement reference count, allow unload when 0 |
| `GetPluginAccess` | Get accessor for an already-loaded plugin |
| `GetAccessPlugin` | Get plugin reference from an accessor |
| `CallPlugin` | Send a message to a plugin through its accessor |
| `GetCurrentPlugin` / `SetCurrentPlugin` | Manage the "current" plugin context |

### Calling Another Plugin

```c
SPErr SendMessage(SPPluginRef plugin, const char *caller,
                  const char *selector, void *message, SPErr *error) {
    SPAccessRef access;
    *error = sAccess->AcquirePlugin(plugin, &access);
    if (*error != kSPNoError) return *error;

    SPErr result;
    *error = sAccess->CallPlugin(access, caller, selector, message, &result);

    sAccess->ReleasePlugin(access);
    return result;
}
```

> **Important:** Do not attempt to acquire suites (other than `SPBlocksSuite`) during access or property messages (`kSPAccessCaller`, `kSPPropertiesCaller`). Most suites are unavailable during load/unload operations.

### Self-Acquisition

A plugin can acquire itself to remain in memory even when no other plugin references it:

```c
SPAccessRef selfAccess;
sAccess->AcquirePlugin(myPluginRef, &selfAccess);
// Plugin will stay loaded until you release
```

---

## SPConfig.h: Platform Detection

`SP/SPConfig.h` defines the platform environment macros used throughout the SP and AE SDK:

```c
// Automatically defined based on compiler:
MAC_ENV     // macOS (detected via __APPLE_CC__)
WIN_ENV     // Windows (detected via _WINDOWS, _MSC_VER, or WINDOWS)
ANDROID_ENV // Android
UNIX_ENV    // Linux
WEB_ENV     // Emscripten/WASM
```

The file enforces that exactly one platform constant is defined -- it will `#error` if none or all are defined.

---

## SPTypes.h: Fundamental Types

`SP/SPTypes.h` defines the foundational types used throughout SP:

```c
typedef int32 SPErr;          // Error code (0 = success)
typedef uint8 SPBoolean;      // Boolean on Mac (uint8)
typedef int32 SPBoolean;      // Boolean on Windows (int32) -- NOTE THE DIFFERENCE

// Integer types (from PSIntTypes.h):
typedef uint8 SPUInt8;
typedef uint16 SPUInt16;
typedef uint32 SPUInt32;
typedef int32 SPInt32;

// SPAPI is the calling convention prefix (empty on modern platforms):
#define SPAPI   // Was 'pascal' on classic Mac
```

> **Pitfall:** `SPBoolean` is `uint8` on Mac but `int32` on Windows. This rarely matters in practice, but be aware of it if you are doing binary serialization or packing structures.

---

## SP Error Codes

All SP errors are defined in `SP/SPErrorCodes.h` as four-character codes:

### General Errors

| Constant | Value | Meaning |
|----------|-------|---------|
| `kSPNoError` | `0` | Success |
| `kSPUnimplementedError` | `'!IMP'` | Function not implemented |
| `kSPUserCanceledError` | `'stop'` | User canceled the operation |
| `kSPOperationInterrupted` | `'intr'` | Operation was interrupted |
| `kSPLogicError` | `'fbar'` | General programming error |
| `kSPBadParameterError` | `'Parm'` | Invalid parameter passed |

### Suite Errors

| Constant | Value | Meaning |
|----------|-------|---------|
| `kSPSuiteNotFoundError` | `'S!Fd'` | Suite name/version not found |
| `kSPSuiteAlreadyExistsError` | `'SExi'` | Suite already registered |
| `kSPSuiteAlreadyReleasedError` | `'SRel'` | Suite was already released |
| `kSPBadSuiteListIteratorError` | `'SLIt'` | Invalid iterator |
| `kSPBadSuiteInternalVersionError` | `'SIVs'` | Internal version mismatch |

### Access Errors

| Constant | Value | Meaning |
|----------|-------|---------|
| `kSPCantAcquirePluginError` | `'!Acq'` | Failed to load/acquire plugin |
| `kSPCantReleasePluginError` | `'!Rel'` | Failed to release plugin |
| `kSPPluginAlreadyReleasedError` | `'AlRl'` | Plugin already released |

### Memory Errors

| Constant | Value | Meaning |
|----------|-------|---------|
| `kSPOutOfMemoryError` | `0xFFFFFF6C` (-108) | Out of memory (same as classic Mac `memFullErr`) |
| `kSPBlockSizeOutOfRangeError` | `'BkRg'` | Requested block size invalid |

### Plugin Errors

| Constant | Value | Meaning |
|----------|-------|---------|
| `kSPPluginNotFound` | `'P!Fd'` | Plugin not found in list |
| `kSPUnknownAdapterError` | `'?Adp'` | Unknown plugin adapter |
| `kSPCorruptPiPLError` | `'CPPL'` | Corrupt PiPL resource |

---

## The Relationship Between SP and PF_*/AEGP_* APIs

SP is the foundation layer. The AE-specific APIs are built on top of it:

```
+--------------------------------------------------+
|  Your Plugin Code                                 |
+--------------------------------------------------+
|  PF_* API (Effects)  |  AEGP_* API (General)     |
|  PF_InData           |  AEGP_SuiteHandler        |
|  PF_Cmd_*            |  AEGP_*Suite*              |
+--------------------------------------------------+
|  AEFX_SuiteHandlerTemplate (C++ convenience)      |
+--------------------------------------------------+
|  SPBasicSuite::AcquireSuite / ReleaseSuite        |
+--------------------------------------------------+
|  Sweet Pea (PICA) Runtime                         |
|  SPSuitesSuite, SPPluginsSuite, SPAccessSuite     |
+--------------------------------------------------+
|  Host Application (After Effects)                 |
+--------------------------------------------------+
```

When you call `AEFX_AcquireSuite(in_data, ...)` or use the `AEFX_SuiteScoper` template, it is ultimately calling `in_data->pica_basicP->AcquireSuite()`. The C++ wrappers just add RAII-style acquire/release management.

---

## Practical Debugging Guide

### Suite Acquisition Failures

**Symptom:** `AcquireSuite` returns `kSPSuiteNotFoundError` (`'S!Fd'`).

**Common causes:**
1. Wrong suite name string -- these are case-sensitive. Check the `#define` in the header.
2. Wrong version number -- requesting a version newer than what the host provides.
3. Calling during load/unload -- most suites are unavailable during `kSPAccessCaller` messages.
4. Suite not available in this AE version -- older AE versions do not have newer suites.

**Debugging steps:**
```c
const void *suiteP = NULL;
SPErr err = pica->AcquireSuite(kMyDesiredSuite, kMyDesiredSuiteVersion, &suiteP);

if (err != kSPNoError) {
    // Log the error code
    char errStr[5] = {0};
    errStr[0] = (char)((err >> 24) & 0xFF);
    errStr[1] = (char)((err >> 16) & 0xFF);
    errStr[2] = (char)((err >> 8) & 0xFF);
    errStr[3] = (char)(err & 0xFF);
    // errStr now contains the readable four-char code like "S!Fd"
}
```

### Plugin Loading Failures

**Symptom:** Plugin does not appear in AE, or AE reports it as broken.

**Check:**
1. PiPL resource is present and well-formed (no `kSPCorruptPiPLError`).
2. Plugin DLL/bundle loads without missing dependencies (use Dependency Walker or `dumpbin /dependents`).
3. Entry point is exported with the correct name and calling convention.
4. Plugin returns `kSPNoError` from its `kSPAccessReloadSelector` handler.

### Reference Count Leaks

If suites are acquired but not released, the providing plugin cannot be unloaded. This typically manifests as AE hanging during shutdown.

**Best practice:** Use RAII wrappers. The SDK provides `AEFX_SuiteScoper`:

```c
// Automatically releases when scope exits
AEFX_SuiteScoper<PF_WorldSuite2> worldSuite(
    in_data->pica_basicP,
    kPFWorldSuite,
    kPFWorldSuiteVersion2,
    out_data);
// Use worldSuite->PF_NewWorld(...)
```

---

## Summary: Why SP Matters

1. **Every suite acquisition goes through SP.** If you understand `SPBasicSuite`, you understand the mechanism behind all AE plugin functionality.

2. **All error codes come from SP.** The four-character codes in `SPErrorCodes.h` appear in debugger sessions constantly.

3. **Plugin lifecycle is SP-managed.** Your plugin's load, unload, reload, and shutdown are all SP access messages.

4. **Suite version negotiation is SP-based.** When you request a suite version, SP finds the right provider and version match.

5. **Memory tracking is SP-provided.** `AllocateBlock`/`FreeBlock` give you host-tracked memory that survives plugin unload/reload cycles.

Treating SP as "just implementation detail" leads to mysterious bugs. Treating it as the foundation it is leads to robust, debuggable plugins.
