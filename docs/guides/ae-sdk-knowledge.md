# After Effects SDK Knowledge Base

This document contains comprehensive knowledge for developing After Effects plugins, compiled from SDK examples, Adobe forums, and expert community contributions.

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Common Patterns](#common-patterns)
3. [Build Configuration](#build-configuration)
4. [Memory Management](#memory-management)
5. [Rendering Pipeline](#rendering-pipeline)
6. [Parameter Management](#parameter-management)
7. [Thread Safety & MFR](#thread-safety--mfr)
8. [Common Issues & Solutions](#common-issues--solutions)

## Core Concepts

### Plugin Architecture
Every After Effects plugin requires:
- **Entry Point**: `PluginDataEntryFunction2` for registration
- **Main Dispatcher**: `EffectMain` to handle commands
- **PiPL Resource**: Defines plugin properties and capabilities
- **Parameter Definition**: UI controls and their behavior

### Essential Headers
```cpp
#include "AEConfig.h"
#include "entry.h"
#include "AE_Effect.h"
#include "AE_EffectCB.h"
#include "AE_Macros.h"
#include "Param_Utils.h"
#include "AE_EffectCBSuites.h"
#include "AEFX_SuiteHelper.h"
#include "AEGP_SuiteHandler.h"
```

### Pixel Format Support
After Effects supports multiple pixel formats:
- **PF_Pixel8**: 8-bit ARGB (0-255)
- **PF_Pixel16**: 16-bit ARGB (0-32768)
- **PF_PixelFloat**: 32-bit float ARGB (0.0-1.0+)

## Common Patterns

### Standard Plugin Structure
```cpp
// Global Setup - Define capabilities
static PF_Err GlobalSetup(PF_InData* in_data, PF_OutData* out_data,
                         PF_ParamDef* params[], PF_LayerDef* output)
{
    out_data->my_version = PF_VERSION(MAJOR, MINOR, BUG, STAGE, BUILD);

    // Standard flags
    out_data->out_flags =
        PF_OutFlag_DEEP_COLOR_AWARE |      // 16-bit support
        PF_OutFlag_PIX_INDEPENDENT;        // Pixels can be processed independently

    // Advanced flags
    out_data->out_flags2 =
        PF_OutFlag2_FLOAT_COLOR_AWARE |    // 32-bit float support
        PF_OutFlag2_SUPPORTS_SMART_RENDER | // SmartFX enabled
        PF_OutFlag2_SUPPORTS_THREADED_RENDERING; // MFR compatible

    return PF_Err_NONE;
}
```

### Parameter Setup Pattern
```cpp
static PF_Err ParamsSetup(PF_InData* in_data, PF_OutData* out_data,
                         PF_ParamDef* params[], PF_LayerDef* output)
{
    PF_ParamDef def;
    AEFX_CLR_STRUCT(def);

    // Slider parameter
    PF_ADD_FLOAT_SLIDERX("Amount",
                        0,      // min
                        100,    // max
                        0,      // slider min
                        100,    // slider max
                        50,     // default
                        PF_Precision_INTEGER,
                        0,      // display flags
                        0,      // flags
                        PARAM_AMOUNT);

    out_data->num_params = PARAM_TOTAL;
    return PF_Err_NONE;
}
```

### Layer Checkout Methods

**Method 1: PF_CHECKOUT_PARAM (Legacy)**
- Returns layer WITHOUT effects applied
- Works regardless of whether layer has effects
```cpp
PF_ParamDef layer_param;
ERR(PF_CHECKOUT_PARAM(in_data, PARAM_LAYER,
                     in_data->current_time,
                     in_data->time_step,
                     in_data->time_scale,
                     &layer_param));
```

**Method 2: checkout_layer_pixels (SmartFX)**
- Returns layer WITH effects applied
- Only available in SmartRender
```cpp
PF_EffectWorld* input_worldP = NULL;
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref,
                                     PARAM_INPUT,
                                     &input_worldP));
```

## Build Configuration

### Visual Studio 2022 Build Command
```bash
"C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe" "D:\Work\Coding\MyPlugin\build\MyPlugin.sln" -p:Configuration=Release -p:Platform=x64
```

### PiPL Resource Requirements
The PiPL must include proper flags for advanced features:

> **Important:** Always verify PiPL flag hex values against your SDK headers -- incorrect values silently disable capabilities.

```r
AE_Effect_Global_OutFlags_2 {
    0x00000400  // FLOAT_COLOR_AWARE
    | 0x08000000  // SUPPORTS_THREADED_RENDERING (MFR) - bit 27
    | 0x00002000  // SUPPORTS_SMART_RENDER
}
```

### CMake Configuration
```cmake
# After Effects SDK paths
set(AESDK_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/AfterEffectsSDK")
include_directories(
    ${AESDK_ROOT}/Examples/Headers
    ${AESDK_ROOT}/Examples/Headers/SP
    ${AESDK_ROOT}/Examples/Util
)

# Link libraries
target_link_libraries(${PROJECT_NAME}
    ${AESDK_ROOT}/Examples/Util/AEGP_SuiteHandler.cpp
)
```

## Memory Management

### World Allocation
```cpp
// Allocate new world
PF_EffectWorld temp_world;
AEFX_CLR_STRUCT(temp_world);
ERR(PF_NEW_WORLD(width, height, PF_NewWorldFlag_NONE, &temp_world));

// Always dispose when done
ERR2(PF_DISPOSE_WORLD(&temp_world));
```

### Buffer Expansion in SmartFX
```cpp
// In PreRender - expand output buffer
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data,
                       PF_PreRenderExtra* extra)
{
    // Expand buffer by N pixels
    int expansion = 10;
    extra->output->result_rect.left -= expansion;
    extra->output->result_rect.top -= expansion;
    extra->output->result_rect.right += expansion;
    extra->output->result_rect.bottom += expansion;

    extra->output->max_result_rect = extra->output->result_rect;
    return PF_Err_NONE;
}
```

## Rendering Pipeline

### SmartFX Render Flow
1. **PreRender**: Setup, request inputs, define output size
2. **SmartRender**: Actual pixel processing

```cpp
static PF_Err SmartRender(PF_InData* in_data, PF_OutData* out_data,
                         PF_SmartRenderExtra* extra)
{
    PF_EffectWorld *input_worldP = NULL, *output_worldP = NULL;

    // Checkout input and output
    ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref,
                                         PARAM_INPUT, &input_worldP));
    ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));

    // Detect pixel format
    // Note: This rowbytes/width heuristic can fail if rowbytes includes padding.
    // The recommended approach is PF_WorldSuite2::PF_GetPixelFormat().
    int bytes_per_pixel = output_worldP->rowbytes / output_worldP->width;

    if (bytes_per_pixel == sizeof(PF_PixelFloat)) {
        // 32-bit float processing
    } else if (bytes_per_pixel == sizeof(PF_Pixel16)) {
        // 16-bit processing
    } else if (bytes_per_pixel == sizeof(PF_Pixel8)) {
        // 8-bit processing
    }

    return PF_Err_NONE;
}
```

## Parameter Management

### Parameter Types
- **PF_ADD_FLOAT_SLIDER**: Floating point slider
- **PF_ADD_CHECKBOX**: Boolean toggle
- **PF_ADD_COLOR**: Color picker
- **PF_ADD_POINT**: 2D point control
- **PF_ADD_LAYER**: Layer selection
- **PF_ADD_BUTTON**: Clickable button
- **PF_ADD_ARBITRARY**: Custom data

### Parameter Dependencies

> **Note:** PF_ParamFlag_SUPERVISE must be set during ParamsSetup, not at runtime.

```cpp
// Make param B depend on param A (set SUPERVISE flag in ParamsSetup)
// Then in PF_Cmd_USER_CHANGED_PARAM:
if (params[PARAM_A]->u.fd.value > 50) {
    params[PARAM_B]->ui_flags |= PF_PUI_DISABLED;
}
```

## Thread Safety & MFR

### MFR Requirements (Multi-Frame Rendering)
1. **No Global Variables**: All state must be local
2. **Thread-Safe Code**: No shared mutable state
3. **Proper Flags**: Must set `PF_OutFlag2_SUPPORTS_THREADED_RENDERING`
4. **Clean Functions**: No side effects

### Thread-Safe Pattern
```cpp
// BAD - Global variable
static int g_counter = 0;  // NOT MFR SAFE!

// GOOD - Local to render function
static PF_Err Render(...)
{
    int local_counter = 0;  // MFR SAFE
    // ... use local_counter
}
```

## Common Issues & Solutions

### Issue: Plugin crashes in older AE but works in newer
**Solution**: Use PF_AE_PLUG_IN_VERSION and the effect API version for feature detection, not the AE product version.
```cpp
// Use the effect API version from in_data for feature detection,
// not the AE application version.
// Check SDK headers for which API version introduced the function you need.
```

### Issue: Can't get full resolution during preview
**Solution**: Use SmartFX and request full resolution in PreRender
```cpp
// Request full resolution
extra->cb->checkout_layer(in_data->effect_ref,
                         PARAM_INPUT,
                         PARAM_INPUT,
                         &req,
                         in_data->current_time,
                         in_data->time_step,
                         in_data->time_scale,
                         &in_result);
```

### Issue: Memory leaks
**Solution**: Always match allocations with deallocations
```cpp
// Use RAII pattern or ensure cleanup
PF_EffectWorld* world = NULL;
PF_Err err = PF_Err_NONE;

ERR(PF_NEW_WORLD(...));
// ... use world
ERR2(PF_DISPOSE_WORLD(&world));  // Always dispose, even on error
```

### Issue: Effect not showing in menu
**Solution**: Check PiPL resource compilation
1. Ensure PiPL is properly compiled to .rc
2. Check Match Name is unique
3. Verify Category is valid
4. Confirm plugin exports correct entry points

### Issue: 32-bit float colors look wrong
**Solution**: Ensure proper flag and handle HDR values
```cpp
// In GlobalSetup
out_data->out_flags2 |= PF_OutFlag2_FLOAT_COLOR_AWARE;

// In render - don't clamp HDR values unnecessarily
float value = pixel.red;  // Can be > 1.0 for HDR
// Only clamp when absolutely necessary
```

## GPU Acceleration Notes

"The plug-in interacts very little with AE in regards to GPU usage. It's mostly plain old GPU usage with a final copy of the GPU render buffer to AE's output buffer."

Key points:
- GPU processing is independent of AE
- Final step copies GPU buffer back to AE
- Use CUDA, OpenGL, or Metal as needed
- Ensure proper context management

## Best Practices Summary

1. **Always use AEFX_CLR_STRUCT()** to initialize structures
2. **Check error codes** with ERR() macro
3. **Use ERR2()** for cleanup code that should run even on error
4. **Test on multiple AE versions** before release
5. **Profile with MFR enabled** to catch threading issues
6. **Document parameter ranges** clearly
7. **Handle all pixel formats** or clearly state limitations
8. **Validate inputs** before processing
9. **Use sequence_data** for persistent storage between frames
10. **Implement About dialog** with version info

## Useful Forum Threads

- [Pre-effect full resolution render](https://community.adobe.com/t5/after-effects-discussions/pre-effect-full-resolution-render/td-p/3472864)
- [Resize output buffer](https://community.adobe.com/t5/after-effects-discussions/resize-the-output/td-p/7223519)
- [Layer parameter and EffectWorld](https://community.adobe.com/t5/after-effects-discussions/aex-sdk-layer-parameter-and-effectworld-problem/m-p/13056405)
- [GPU acceleration in plugins](https://community.adobe.com/t5/after-effects-discussions/ae-2017-1-win-sdk-create-plugin-accelerated-by-gpu-cuda/m-p/11757007)

## Additional Resources

- [Official SDK Guide](https://ae-plugins.docsforadobe.dev/)
- [GitHub SDK Examples](https://github.com/docsforadobe/after-effects-plugin-guide)
- [AE Enhancers Forum](http://www.aenhancers.com/)
- [Adobe Creative Cloud Developer Forums](https://forums.creativeclouddeveloper.com/)
