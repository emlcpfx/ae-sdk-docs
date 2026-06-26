# Reference After Effects Plugin Patterns

Extracted patterns and techniques from real-world After Effects plugins to enhance plugin development knowledge.

## Table of Contents

1. [Common Plugin Architecture](#common-plugin-architecture)
2. [Parameter Setup Patterns](#parameter-setup-patterns)
3. [Pixel Processing Techniques](#pixel-processing-techniques)
4. [Memory Management Patterns](#memory-management-patterns)
5. [Utility Functions & Helpers](#utility-functions--helpers)
6. [PiPL Resource Patterns](#pipl-resource-patterns)
7. [Error Handling Patterns](#error-handling-patterns)
8. [Performance Optimizations](#performance-optimizations)

---

## Common Plugin Architecture

### Standard Plugin Structure
All reference plugins follow this consistent structure:

```cpp
// Standard entry points
static PF_Err About(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output)
static PF_Err GlobalSetup(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output)
static PF_Err ParamsSetup(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output)
static PF_Err Render(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output)

// Main entry point
extern "C" DllExport
PF_Err PluginDataEntryFunction(PF_PluginDataPtr inPtr, PF_PluginDataCB inPluginDataCallBackPtr,
                               SPBasicSuite* inSPBasicSuitePtr, const char* inHostName, const char* inHostVersion)

// Command dispatcher
PF_Err EffectMain(PF_Cmd cmd, PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output, void *extra)
```

### About Function Pattern
```cpp
static PF_Err About(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output) {
    AEGP_SuiteHandler suites(in_data->pica_basicP);
    suites.ANSICallbacksSuite1()->sprintf(out_data->return_msg, "%s v%d.%d\r%s",
        "PluginName", MAJOR_VERSION, MINOR_VERSION, "Description");
    return PF_Err_NONE;
}
```

### GlobalSetup Pattern
```cpp
static PF_Err GlobalSetup(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output) {
    out_data->out_flags = PF_OutFlag_DEEP_COLOR_AWARE;  // 16-bit support
    out_data->my_version = PF_VERSION(MAJOR_VERSION, MINOR_VERSION, BUG_VERSION, STAGE_VERSION, BUILD_VERSION);
    return PF_Err_NONE;
}
```

---

## Parameter Setup Patterns

### Parameter Definition with Clear Structure
```cpp
static PF_Err ParamsSetup(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output) {
    PF_ParamDef def;

    // Clear structure before each parameter
    AEFX_CLR_STRUCT(def);
    PF_ADD_POPUPX("Type", 2, 1, "Directional|Radial", NULL, PARAM_TYPE);

    AEFX_CLR_STRUCT(def);
    PF_ADD_SLIDER("Max Size", min, max, min, max, default, PARAM_SIZE);

    AEFX_CLR_STRUCT(def);
    PF_ADD_ANGLE("Direction", 0, PARAM_DIRECTION);

    AEFX_CLR_STRUCT(def);
    PF_ADD_FLOAT_SLIDERX("Amount", 0, 100, 0, 100, 50, PF_Precision_TENTHS, NULL, NULL, PARAM_AMOUNT);

    AEFX_CLR_STRUCT(def);
    PF_ADD_POINT("Center", 50, 50, false, PARAM_CENTER);

    AEFX_CLR_STRUCT(def);
    PF_ADD_LAYER("Displacement Map", PF_LayerDefault_NONE, PARAM_MAP);

    out_data->num_params = TOTAL_NUM_PARAMS;
    return PF_Err_NONE;
}
```

### Parameter Enum Pattern
```cpp
enum {
    PLUGIN_INPUT = 0,        // Always first
    PLUGIN_TYPE,
    PLUGIN_SIZE,
    PLUGIN_DIRECTION,
    PLUGIN_AMOUNT,
    PLUGIN_CENTER,
    PLUGIN_DISPLACEMENT_MAP,
    PLUGIN_NUM_PARAMS        // Always last
};
```

---

## Pixel Processing Techniques

### Direct Pixel Access Pattern
```cpp
// Safe pixel access with bounds checking
PF_Pixel8 *getPixel8(PF_LayerDef *layer, int x, int y) {
    return (PF_Pixel8*)((char *)layer->data + (y * layer->rowbytes) + (x * sizeof(PF_Pixel8)));
}

PF_Pixel16 *getPixel16(PF_LayerDef *layer, int x, int y) {
    return (PF_Pixel16*)((char *)layer->data + (y * layer->rowbytes) + (x * sizeof(PF_Pixel16)));
}

bool isOnScreen(PF_LayerDef *layer, int x, int y) {
    return (x > -1 && y > -1 && x < layer->width && y < layer->height);
}
```

### Pixel Data Structures
```cpp
typedef struct {
    PF_Pixel8 p;
    double value;    // Processed luminance or other metric
} Pixel8Data;

typedef struct {
    PF_Pixel16 p;
    double value;
} Pixel16Data;

// Extract brightness for sorting/processing
// Note: AE uses weighted luminance (Rec.709), not simple (R+G+B)/3.
// For accurate luminance: 0.2126*R + 0.7152*G + 0.0722*B
Pixel8Data getPixel8Data(PF_Pixel8 *p) {
    Pixel8Data res;
    res.p = *p;
    res.value = ((p->red + p->green + p->blue) / (double)PF_MAX_CHAN8) / 3.0;
    return res;
}
```

### Iterate Suite Pattern for Per-Pixel Processing
```cpp
static PF_Err ProcessPixel8(void *refcon, A_long xL, A_long yL, PF_Pixel8 *inP, PF_Pixel8 *outP) {
    MyPluginInfo *info = (MyPluginInfo*)refcon;

    // Your pixel processing logic here
    // Access info for parameters and state

    outP->alpha = inP->alpha;  // Preserve alpha
    outP->red = processed_red;
    outP->green = processed_green;
    outP->blue = processed_blue;

    return PF_Err_NONE;
}

// In Render function:
if (!err) {
    bool world_is_deep = PF_WORLD_IS_DEEP(output);
    A_long lines = output->extent_hint.bottom - output->extent_hint.top;

    if (!world_is_deep) {
        ERR(suites.Iterate8Suite1()->iterate(in_data, 0, lines, input_layer, NULL, (void*)&info, ProcessPixel8, output));
    } else {
        // Handle 16-bit processing
        ERR(suites.Iterate16Suite1()->iterate(in_data, 0, lines, input_layer, NULL, (void*)&info, ProcessPixel16, output));
    }
}
```

---

## Memory Management Patterns

### Dynamic Memory Allocation Pattern
```cpp
// Allocate memory using AE's memory suite
AEGP_MemHandle mem_handle;
MyDataType *data_ptr;
int mem_size = count * sizeof(MyDataType);

info->mem_suite->AEGP_NewMemHandle(NULL, "Memory Description", mem_size, AEGP_MemFlag_NONE, &mem_handle);
info->mem_suite->AEGP_LockMemHandle(mem_handle, (void**)&data_ptr);

// Use data_ptr for processing...

// Always cleanup
info->mem_suite->AEGP_UnlockMemHandle(mem_handle);
info->mem_suite->AEGP_FreeMemHandle(mem_handle);
```

### Plugin Info Structure Pattern
```cpp
typedef struct {
    // Memory suite reference
    AEGP_MemorySuite1 *mem_suite;

    // User parameters (processed values)
    int type;
    int chunk_size;
    double lower_threshold;
    double upper_threshold;
    double direction_angle;
    double center_x, center_y;
    double scale_factor;

    // Derived/computed values
    double vec_x, vec_y;         // Direction vectors
    bool use_displacement;

    // Layer references
    PF_LayerDef *input_layer;
    PF_LayerDef *displacement_map;
    A_long rowbytes;
} MyPluginInfo;
```

---

## Utility Functions & Helpers

### Mathematical Helpers
```cpp
// Safe clamping
#define CLAMP(value, min, max) (((value) < (min)) ? (min) : (((value) > (max)) ? (max) : (value)))

// Fixed point conversions
#define D2FIX(x) (((long)x)<<16)
#define FIX2D(x) (x / ((double)(1L << 16)))
#define FIX2LONG(x) ((x) >> 16)

// Mathematical utilities
double clamp(double value, double min, double max) {
    return (value < min) ? min : (value > max) ? max : value;
}

int mirror(int value, int bounds) {
    if (value < 0) return -value;
    if (value >= bounds) return 2 * bounds - value - 1;
    return value;
}

bool filter(double value, double lower, double upper) {
    return (value >= lower && value <= upper);
}
```

### Parameter Extraction Pattern
```cpp
// In Render function - extract all parameters at once
MyPluginInfo info;
AEFX_CLR_STRUCT(info);

info.type = params[PARAM_TYPE]->u.pd.value;
info.chunk_size = (int)round(params[PARAM_SIZE]->u.sd.value * scale);
info.direction = (FIX2D(params[PARAM_DIRECTION]->u.ad.value) - 90) * PF_RAD_PER_DEGREE;
info.lower_threshold = params[PARAM_LOWER]->u.fs_d.value / 100.0;
info.upper_threshold = params[PARAM_UPPER]->u.fs_d.value / 100.0;
info.center_x = FIX2D(params[PARAM_CENTER]->u.td.x_value);
info.center_y = FIX2D(params[PARAM_CENTER]->u.td.y_value);

// Compute derived values
info.vec_x = cos(info.direction);
info.vec_y = sin(info.direction);
```

---

## PiPL Resource Patterns

### Cross-Platform PiPL Template
```r
#include "AEConfig.h"
#include "AE_EffectVers.h"

#ifndef AE_OS_WIN
    #include <AE_General.r>
#endif

resource 'PiPL' (16000) {
    {
        Kind {
            AEEffect
        },
        Name {
            "Plugin Name"
        },
        Category {
            "Category Name"
        },
#ifdef AE_OS_WIN
    #ifdef AE_PROC_INTELx64
        CodeWin64X86 {"EntryPointFunc"},
    #else
        CodeWin32X86 {"EntryPointFunc"},
    #endif
#else
    #ifdef AE_OS_MAC
        /* Note: CodeMachOPowerPC is legacy (PowerPC Mac era) and should not
           be included in modern plugins. It is shown here for historical
           reference only. Modern Mac plugins need only CodeMacIntel64
           and/or CodeMacARM64. */
        CodeMacIntel64 {"EntryPointFunc"},
    #endif
#endif
        AE_PiPL_Version {
            2,
            0
        },
        AE_Effect_Spec_Version {
            PF_PLUG_IN_VERSION,
            PF_PLUG_IN_SUBVERS
        },
        AE_Effect_Version {
            524289    /* v1.0 */
        },
        AE_Effect_Info_Flags {
            0
        },
        AE_Effect_Global_OutFlags {
            0x00008000    // PF_OutFlag_DEEP_COLOR_AWARE (1L << 15)
        },
        AE_Effect_Global_OutFlags_2 {
            0x00000000
        },
        AE_Effect_Match_Name {
            "ADBE_PluginName_v1.0"  // Must be unique!
        },
        AE_Reserved_Info {
            0
        }
    }
};
```

---

## Error Handling Patterns

### Standard Error Handling

> **Important:** The ERR macro does NOT prevent the wrapped function call from executing -- it only prevents overwriting a previous error code. The wrapped function still runs regardless of the current error state. If you need to skip calls on error, use an explicit `if (!err)` check.

```cpp
// ERR macro - always executes the function, only conditionally stores the error
#define ERR(FUNC) \
    do { \
        PF_Err _err = (FUNC); \
        if (_err && !err) { err = _err; } \
    } while(0)

// ERR2 for secondary cleanup
#define ERR2(FUNC) \
    do { \
        PF_Err _err = (FUNC); \
        if (_err && !err2) { err2 = _err; } \
    } while(0)

// Usage in render function:
static PF_Err Render(PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output) {
    PF_Err err = PF_Err_NONE;
    PF_Err err2 = PF_Err_NONE;

    // Parameter checkout with error handling
    PF_ParamDef checkout;
    AEFX_CLR_STRUCT(checkout);
    ERR(PF_CHECKOUT_PARAM(in_data, PARAM_LAYER, in_data->current_time, in_data->time_step, in_data->time_scale, &checkout));

    // Processing logic here...

    // Always check in parameters
    ERR2(PF_CHECKIN_PARAM(in_data, &checkout));

    return err;
}
```

### Try-Catch Exception Handling
```cpp
PF_Err EffectMain(PF_Cmd cmd, PF_InData *in_data, PF_OutData *out_data, PF_ParamDef *params[], PF_LayerDef *output, void *extra) {
    PF_Err err = PF_Err_NONE;

    try {
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
        case PF_Cmd_RENDER:
            err = Render(in_data, out_data, params, output);
            break;
        default:
            break;
        }
    }
    catch (const PF_Err &thrown_err) {
        err = thrown_err;
    }
    return err;
}
```

---

## Performance Optimizations

### Scaling and Downsampling Awareness
```cpp
// Account for composition scaling
PF_LayerDef *input_layer = &params[PARAM_INPUT]->u.ld;
double scale = input_layer->width / (double)in_data->width;
int effective_size = (int)round(params[PARAM_SIZE]->u.sd.value * scale);

// Handle quarter/half resolution previews
int qscale = max(1, (int)round((float)in_data->width / input_layer->width));
```

### Bit Depth Detection and Processing
```cpp
if (!err) {
    bool world_is_deep = PF_WORLD_IS_DEEP(output);
    A_long lines = output->extent_hint.bottom - output->extent_hint.top;

    if (!world_is_deep) {
        // 8-bit processing path
        ERR(suites.Iterate8Suite1()->iterate(in_data, 0, lines, input_layer, NULL, (void*)&info, Process8, output));
    } else {
        // 16-bit processing path
        ERR(suites.Iterate16Suite1()->iterate(in_data, 0, lines, input_layer, NULL, (void*)&info, Process16, output));
    }
}
```

### Memory Efficient Buffer Creation
```cpp
// Create working buffer only when needed
PF_LayerDef buffer;
ERR(suites.WorldSuite1()->new_world(in_data->effect_ref,
    input_layer->width, input_layer->height,
    PF_NewWorldFlag_CLEAR_PIXELS, &buffer));

// Process using buffer...

// Always dispose of created worlds
ERR2(suites.WorldSuite1()->dispose_world(in_data->effect_ref, &buffer));
```

---

## Key Architectural Insights

### Plugin Registration Pattern
```cpp
extern "C" DllExport
PF_Err PluginDataEntryFunction(PF_PluginDataPtr inPtr, PF_PluginDataCB inPluginDataCallBackPtr,
                               SPBasicSuite* inSPBasicSuitePtr, const char* inHostName, const char* inHostVersion) {
    PF_Err result = PF_Register_EFFECT(
        inPtr,
        inPluginDataCallBackPtr,
        "Plugin Display Name",      // What users see
        "ADBE_PluginName_v1.0",    // Unique match name
        "Category Name",           // Menu category
        AE_RESERVED_INFO
    );
    return result;
}
```

### Version Management
```cpp
// Consistent version defines
#define MAJOR_VERSION 1
#define MINOR_VERSION 0
#define BUG_VERSION   0
#define STAGE_VERSION PF_Stage_DEVELOP  // or PF_Stage_RELEASE
#define BUILD_VERSION 1
```

### Suite Handler Usage
```cpp
// Always initialize suite handler
AEGP_SuiteHandler suites(in_data->pica_basicP);

// Access suites through handler
suites.ANSICallbacksSuite1()->sprintf(...);
suites.Iterate8Suite1()->iterate(...);
suites.WorldSuite1()->new_world(...);
suites.MemorySuite1()->AEGP_NewMemHandle(...);
```

---

## Advanced Techniques Found

### Time-Based Processing
- Multiple frame sampling for temporal effects
- Frame interpolation and time offset handling
- Memory management for frame buffers

### Displacement Mapping
- Using secondary layer as displacement source
- Vector field distortion techniques
- Safe parameter checkout/checkin patterns

### Sampling and Filtering
- Area sampling with PF_SampPB structures
- Multi-resolution processing awareness
- Threshold-based pixel filtering

### Custom Data Structures
- Plugin-specific info structures passed via refcon
- Memory suite integration for dynamic allocation
- Thread-safe data access patterns

---

*These patterns represent battle-tested techniques from real After Effects plugins and demonstrate professional-grade plugin development practices.*
