# After Effects Plugin Development Basics

This document captures the essential knowledge for creating working After Effects plugins, based on hard-won experience debugging initialization issues.

## Critical Rules for Plugin Loading

### 1. Parameter Structure is Everything

**RULE**: After Effects plugins MUST follow exact parameter patterns to load successfully.

```cpp
// CORRECT Parameter Setup (based on working Skeleton)
enum {
    PLUGIN_INPUT = 0,        // Input layer (automatic, don't explicitly add)
    PLUGIN_GAIN,             // Additional parameters (explicitly add)
    PLUGIN_NUM_PARAMS        // Total count
};

// In ParamsSetup():
static PF_Err ParamsSetup(PF_InData *in_data, PF_OutData *out_data,
                         PF_ParamDef *params[], PF_LayerDef *output) {
    PF_Err err = PF_Err_NONE;
    PF_ParamDef def;

    AEFX_CLR_STRUCT(def);

    // Input layer is parameter 0 - AE adds this automatically, DON'T add it yourself

    // Add additional parameters explicitly
    PF_ADD_FLOAT_SLIDERX("Gain", 0, 100, 0, 100, 10,
                         PF_Precision_HUNDREDTHS, 0, 0, GAIN_DISK_ID);

    out_data->num_params = PLUGIN_NUM_PARAMS;  // MUST match enum count

    return err;
}
```

> **Note:** A plugin can have just the input layer (num_params = 1) with no additional parameters. The Skeleton example uses additional parameters, but they are not strictly required.

### 2. PiPL Resource Compilation is Critical

**RULE**: The PiPL (Plugin Property List) must be properly compiled or AE won't recognize the plugin.

#### Three-Step PiPL Compilation Process:
```bash
# 1. Preprocess the .r file
cl /I "SDK\Headers" /EP "CamProjectionPiPL.r" > "CamProjectionPiPL.rr"

# 2. Run PiPLTool
PiPLTool.exe "CamProjectionPiPL.rr" "CamProjectionPiPL.rrc"

# 3. Final preprocessing
cl /D "MSWindows" /EP "CamProjectionPiPL.rrc" > "CamProjectionPiPL.rc"
```

#### PiPL Structure Template:
```r
#include "AEConfig.h"
#include "AE_EffectVers.h"

resource 'PiPL' (16000) {
    {
        Kind { AEEffect },
        Name { "Your Plugin Name" },
        Category { "Sample Plug-ins" },
#ifdef AE_OS_WIN
    #ifdef AE_PROC_INTELx64
        CodeWin64X86 {"EffectMain"},
    #endif
#endif
        AE_PiPL_Version { 2, 0 },
        AE_Effect_Spec_Version { PF_PLUG_IN_VERSION, PF_PLUG_IN_SUBVERS },
        AE_Effect_Version { 525824 /* 1.0.0 */ },
        AE_Effect_Info_Flags { 0 },
        AE_Effect_Global_OutFlags { 0x00400000 /* PF_OutFlag_DEEP_COLOR_AWARE */ },
        AE_Effect_Global_OutFlags_2 { 0x00000000 },
        AE_Effect_Match_Name { "ADBE Your Plugin Name" },
        AE_Reserved_Info { 0 },
        AE_Effect_Support_URL { "https://www.adobe.com" }
    }
};
```

### 3. Global Output Flags Must Match

**RULE**: Flags in code MUST exactly match flags in PiPL resource.

```cpp
// In GlobalSetup():
out_data->out_flags = PF_OutFlag_DEEP_COLOR_AWARE;    // = 0x00400000 (1L << 22)
out_data->out_flags2 = 0;                             // = 0x00000000

// In PiPL (.r file):
AE_Effect_Global_OutFlags { 0x00400000 },      // MUST match out_flags
AE_Effect_Global_OutFlags_2 { 0x00000000 },    // MUST match out_flags2
```

**Mismatch causes**: "effect has global 2outflags mismatch" error.

### 4. Function Export Requirements

**RULE**: Entry points must be properly exported for AE to find them.

```cpp
// Plugin Data Entry Point
extern "C" __declspec(dllexport) PF_Err PluginDataEntryFunction2(
    PF_PluginDataPtr inPtr,
    PF_PluginDataCB2 inPluginDataCallBackPtr,
    SPBasicSuite* inSPBasicSuitePtr,
    const char* inHostName,
    const char* inHostVersion)
{
    return PF_REGISTER_EFFECT_EXT2(
        inPtr, inPluginDataCallBackPtr,
        "Plugin Name",           // Name
        "ADBE Plugin Name",      // Match Name
        "Sample Plug-ins",       // Category
        0,                       // Reserved Info
        "EffectMain",            // Entry point name
        "https://www.adobe.com"  // Support URL
    );
}

// Main Effect Entry Point
extern "C" __declspec(dllexport) PF_Err EffectMain(
    PF_Cmd cmd, PF_InData *in_data, PF_OutData *out_data,
    PF_ParamDef *params[], PF_LayerDef *output, void *extra)
{
    // Command handling...
}
```

## Essential Command Handlers

### Minimum Required Handlers:

> **Note:** Always use `PF_Cmd_*` constants instead of hardcoded numeric values. The numeric values shown in comments are for reference only and may change between SDK versions.

```cpp
switch (cmd) {
    case PF_Cmd_ABOUT:
        err = About(in_data, out_data, params, output);
        break;

    case PF_Cmd_GLOBAL_SETUP:
        err = GlobalSetup(in_data, out_data, params, output);
        break;

    case PF_Cmd_PARAMS_SETUP:
        err = ParamsSetup(in_data, out_data, params, output);
        break;

    case PF_Cmd_SEQUENCE_SETUP:
        err = PF_Err_NONE;  // Can be empty
        break;

    case PF_Cmd_RENDER:
        err = Render(in_data, out_data, params, output);
        break;

    default:
        break;  // Unknown commands are OK to ignore
}
```

### Command Execution Flow:
```
1. PF_Cmd_GLOBAL_SETUP    - Set plugin capabilities
2. PF_Cmd_PARAMS_SETUP    - Define parameters
3. PF_Cmd_SEQUENCE_SETUP  - Per-sequence initialization
4. PF_Cmd_RENDER          - Process frames
```

## Common Failure Modes & Solutions

### "Plugin not found" / "Missing PiPL property"
- **Cause**: PiPL resource not properly compiled
- **Fix**: Use exact 3-step PiPL compilation process
- **Check**: Verify .rc file contains binary data, not just placeholder bytes

### "Couldn't find main entry point"
- **Cause**: Missing `extern "C" __declspec(dllexport)` on functions
- **Fix**: Add proper export declarations to both entry functions

### "Parameter count mismatch"
- **Cause**: Code parameter count doesn't match AE's expectations
- **Fix**: Ensure `out_data->num_params` matches your enum count exactly

### "Global outflags mismatch"
- **Cause**: Flags in GlobalSetup() don't match PiPL resource
- **Fix**: Ensure exact hex value matching between code and .r file

### "Cannot be initialized"
- **Cause**: Usually parameter setup issues or missing handlers
- **Fix**: Add debug logging to identify exact failure point

## Debug Logging Pattern

```cpp
// Windows debug logging
static void DebugLog(const char* message) {
    OutputDebugStringA("[PluginName] ");
    OutputDebugStringA(message);
    OutputDebugStringA("\n");
}

// Use in command handlers
case PF_Cmd_GLOBAL_SETUP:
    DebugLog("PF_Cmd_GLOBAL_SETUP called");
    err = GlobalSetup(in_data, out_data, params, output);
    if (err) DebugLog("GlobalSetup() failed");
    else DebugLog("GlobalSetup() succeeded");
    break;
```

## Build Requirements

### CMake Integration:
```cmake
# Add PiPL resource compilation
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/PluginPiPL.rc
    COMMAND cl /I "${AESDK_ROOT}/Headers" /EP "${CMAKE_CURRENT_SOURCE_DIR}/resources/PluginPiPL.r"
            > "${CMAKE_CURRENT_BINARY_DIR}/PluginPiPL.rr"
    COMMAND "${AESDK_ROOT}/Resources/PiPLTool.exe"
            "${CMAKE_CURRENT_BINARY_DIR}/PluginPiPL.rr"
            "${CMAKE_CURRENT_BINARY_DIR}/PluginPiPL.rrc"
    COMMAND cl /D "MSWindows" /EP "${CMAKE_CURRENT_BINARY_DIR}/PluginPiPL.rrc"
            > "${CMAKE_CURRENT_BINARY_DIR}/PluginPiPL.rc"
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/resources/PluginPiPL.r
)

# Link the compiled resource
target_sources(YourPlugin PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/PluginPiPL.rc)
```

### File Extension:
- **MUST** be `.aex` (not `.dll`)
- Output goes to AE plugins directory

## Testing Checklist

1. Plugin appears in Effects menu
2. Parameters visible in Effect Controls
3. Debug output shows successful command sequence:
   - `PF_Cmd_GLOBAL_SETUP` -> succeeded
   - `PF_Cmd_PARAMS_SETUP` -> succeeded
   - `PF_Cmd_SEQUENCE_SETUP` -> succeeded
   - `PF_Cmd_RENDER` -> called (success!)

## Key Insights

- **Input layer is automatic** - don't explicitly add parameter 0
- **PiPL compilation is everything** - broken PiPL = invisible plugin
- **Exact flag matching** - code and PiPL flags must be identical
- **Debug logging is essential** - use OutputDebugStringA for Windows

This foundation ensures your plugin loads successfully before adding complex functionality.
