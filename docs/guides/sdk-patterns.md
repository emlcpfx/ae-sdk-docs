# After Effects SDK Code Patterns

This document contains recurring patterns found in After Effects SDK examples, providing templates for common plugin development tasks.

## Table of Contents
1. [Plugin Initialization](#plugin-initialization)
2. [Parameter Patterns](#parameter-patterns)
3. [Rendering Patterns](#rendering-patterns)
4. [Memory Management Patterns](#memory-management-patterns)
5. [Suite Usage Patterns](#suite-usage-patterns)
6. [Error Handling Patterns](#error-handling-patterns)
7. [Pixel Processing Patterns](#pixel-processing-patterns)
8. [Utility Patterns](#utility-patterns)

## Plugin Initialization

### Complete Entry Point Pattern
```cpp
// Standard Windows export
#ifdef AE_OS_WIN
    #define DllExport __declspec(dllexport)
#else
    #define DllExport __attribute__((visibility("default")))
#endif

// Entry point for plugin registration
extern "C" DllExport
PF_Err PluginDataEntryFunction2(
    PF_PluginDataPtr inPtr,
    PF_PluginDataCB2 inPluginDataCallBackPtr,
    SPBasicSuite* inSPBasicSuitePtr,
    const char* inHostName,
    const char* inHostVersion)
{
    PF_Err result = PF_Err_INVALID_CALLBACK;

    result = PF_REGISTER_EFFECT_EXT2(
        inPtr,
        inPluginDataCallBackPtr,
        "MyEffect",                 // Name shown in UI
        "ADBE MyCompany MyEffect",  // Unique match name
        "MyCompany",                // Category in effects menu
        AE_RESERVED_INFO,           // Reserved, must be this
        "EffectMain",               // Main function name
        "https://mycompany.com");   // Support URL

    return result;
}
```

### Main Dispatcher Pattern
```cpp
extern "C" DllExport
PF_Err EffectMain(
    PF_Cmd cmd,
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_ParamDef* params[],
    PF_LayerDef* output,
    void* extra)
{
    PF_Err err = PF_Err_NONE;

    try {
        switch(cmd) {
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

            case PF_Cmd_SMART_PRE_RENDER:
                err = PreRender(in_data, out_data,
                              reinterpret_cast<PF_PreRenderExtra*>(extra));
                break;

            case PF_Cmd_SMART_RENDER:
                err = SmartRender(in_data, out_data,
                                reinterpret_cast<PF_SmartRenderExtra*>(extra));
                break;

            case PF_Cmd_USER_CHANGED_PARAM:
                err = UserChangedParam(in_data, out_data, params, output,
                                     reinterpret_cast<PF_UserChangedParamExtra*>(extra));
                break;

            case PF_Cmd_UPDATE_PARAMS_UI:
                err = UpdateParamsUI(in_data, out_data, params, output);
                break;
        }
    } catch(PF_Err &thrown_err) {
        err = thrown_err;
    }

    return err;
}
```

## Parameter Patterns

### Complete Parameter Setup
```cpp
enum {
    PARAM_INPUT = 0,
    PARAM_SLIDER,
    PARAM_CHECKBOX,
    PARAM_COLOR,
    PARAM_POINT,
    PARAM_POPUP,
    PARAM_LAYER,
    PARAM_TOTAL
};

static PF_Err ParamsSetup(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_ParamDef* params[],
    PF_LayerDef* output)
{
    PF_Err err = PF_Err_NONE;
    PF_ParamDef def;

    // Float slider
    AEFX_CLR_STRUCT(def);
    PF_ADD_FLOAT_SLIDERX(
        "Amount",           // Parameter name
        0.0,                // Min value
        100.0,              // Max value
        0.0,                // Slider min
        100.0,              // Slider max
        50.0,               // Default
        PF_Precision_TENTHS,// Precision
        0,                  // Display flags
        0,                  // Flags
        PARAM_SLIDER);

    // Checkbox
    AEFX_CLR_STRUCT(def);
    PF_ADD_CHECKBOX(
        "Enable",           // Name
        "On",               // Label when checked
        TRUE,               // Default (checked)
        0,                  // Flags
        PARAM_CHECKBOX);

    // Color picker
    AEFX_CLR_STRUCT(def);
    PF_ADD_COLOR(
        "Tint",             // Name
        128,                // Default red
        128,                // Default green
        128,                // Default blue
        PARAM_COLOR);

    // Point control
    AEFX_CLR_STRUCT(def);
    PF_ADD_POINT(
        "Center",           // Name
        50,                 // Default X (% of width)
        50,                 // Default Y (% of height)
        0,                  // Restrict bounds (bool)
        PARAM_POINT);

    // Popup menu
    AEFX_CLR_STRUCT(def);
    PF_ADD_POPUP(
        "Mode",             // Name
        3,                  // Number of choices
        1,                  // Default choice (1-based)
        "Linear|"           // Choices separated by |
        "Radial|"
        "Square",
        PARAM_POPUP);

    // Layer parameter
    AEFX_CLR_STRUCT(def);
    PF_ADD_LAYER(
        "Map Layer",        // Name
        PF_LayerDefault_NONE,// Default
        PARAM_LAYER);

    out_data->num_params = PARAM_TOTAL;
    return err;
}
```

### Parameter Checkout Pattern
```cpp
static PF_Err CheckoutAndUseParam(
    PF_InData* in_data,
    PF_ParamDef* params[])
{
    PF_Err err = PF_Err_NONE;
    PF_ParamDef param;
    AEFX_CLR_STRUCT(param);

    // Checkout parameter at current time
    ERR(PF_CHECKOUT_PARAM(
        in_data,
        PARAM_SLIDER,
        in_data->current_time,
        in_data->time_step,
        in_data->time_scale,
        &param));

    // Use the parameter value
    PF_FpLong value = param.u.fs_d.value;

    // Always check in when done
    ERR2(PF_CHECKIN_PARAM(in_data, &param));

    return err;
}
```

## Rendering Patterns

### Standard Render Pattern (8-bit)
```cpp
static PF_Err Render(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_ParamDef* params[],
    PF_LayerDef* output)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    // Setup iteration parameters
    PF_Iterate8Suite1* iterate8Suite = NULL;
    ERR(suites.PFIterate8Suite1()->get(&iterate8Suite));

    // Define iteration area
    PF_LRect iterate_rect;
    iterate_rect.left = 0;
    iterate_rect.top = 0;
    iterate_rect.right = output->width;
    iterate_rect.bottom = output->height;

    // User data for iteration
    MyIterateData iterate_data;
    iterate_data.amount = params[PARAM_SLIDER]->u.fs_d.value;

    // Iterate over pixels
    ERR(iterate8Suite->iterate(
        in_data,
        0,                          // Progress base
        iterate_rect.bottom - iterate_rect.top,  // Progress height
        &params[PARAM_INPUT]->u.ld,// Input layer
        NULL,                       // Area (NULL = all)
        (void*)&iterate_data,       // User data
        MyPixelFunc8,               // Pixel function
        output));                   // Output layer

    return err;
}

// Pixel function for 8-bit
static PF_Err MyPixelFunc8(
    void* refcon,
    A_long xL,
    A_long yL,
    PF_Pixel8* inP,
    PF_Pixel8* outP)
{
    MyIterateData* data = (MyIterateData*)refcon;

    // Process pixel
    outP->alpha = inP->alpha;
    outP->red = (A_u_char)(inP->red * data->amount / 100.0);
    outP->green = (A_u_char)(inP->green * data->amount / 100.0);
    outP->blue = (A_u_char)(inP->blue * data->amount / 100.0);

    return PF_Err_NONE;
}
```

### SmartFX Render Pattern
```cpp
static PF_Err SmartRender(
    PF_InData* in_data,
    PF_OutData* out_data,
    PF_SmartRenderExtra* extra)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    PF_EffectWorld *input_worldP = NULL, *output_worldP = NULL;

    // Checkout input and output
    ERR(extra->cb->checkout_layer_pixels(
        in_data->effect_ref,
        PARAM_INPUT,
        &input_worldP));

    ERR(extra->cb->checkout_output(
        in_data->effect_ref,
        &output_worldP));

    // Get parameter
    PF_ParamDef param;
    AEFX_CLR_STRUCT(param);
    ERR(PF_CHECKOUT_PARAM(
        in_data,
        PARAM_SLIDER,
        in_data->current_time,
        in_data->time_step,
        in_data->time_scale,
        &param));

    // Process based on bit depth
    if (!err && input_worldP && output_worldP) {
        // Use PF_WorldSuite2::PF_GetPixelFormat() to determine the pixel format.
        PF_PixelFormat format = PF_PixelFormat_INVALID;
        AEFX_SuiteScoper<PF_WorldSuite2> worldSuite =
            AEFX_SuiteScoper<PF_WorldSuite2>(
                in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);
        ERR(worldSuite->PF_GetPixelFormat(input_worldP, &format));

        switch(format) {
            case PF_PixelFormat_ARGB32:
                // 8-bit processing
                Process8(input_worldP, output_worldP, param.u.fs_d.value);
                break;

            case PF_PixelFormat_ARGB64:
                // 16-bit processing
                Process16(input_worldP, output_worldP, param.u.fs_d.value);
                break;

            case PF_PixelFormat_ARGB128:
                // 32-bit float processing
                Process32(input_worldP, output_worldP, param.u.fs_d.value);
                break;
        }
    }

    // Checkin parameter
    ERR2(PF_CHECKIN_PARAM(in_data, &param));

    return err;
}
```

## Memory Management Patterns

### World Allocation Pattern
```cpp
static PF_Err AllocateAndUseWorld(
    PF_InData* in_data,
    PF_LayerDef* output)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    PF_EffectWorld temp_world;
    AEFX_CLR_STRUCT(temp_world);

    // Allocate world
    ERR(PF_NEW_WORLD(
        output->width,
        output->height,
        PF_NewWorldFlag_NONE,
        &temp_world));

    if (!err && temp_world.data) {
        // Use the world
        ProcessWorld(&temp_world);

        // Copy to output
        ERR(PF_COPY(
            &temp_world,
            output,
            NULL,  // Origin
            NULL)); // Dest rect
    }

    // Always dispose
    ERR2(PF_DISPOSE_WORLD(&temp_world));

    return err;
}
```

### Handle Allocation Pattern
```cpp
typedef struct {
    PF_FpLong value;
    A_long count;
} MySequenceData;

static PF_Err SequenceSetup(
    PF_InData* in_data,
    PF_OutData* out_data)
{
    PF_Err err = PF_Err_NONE;

    // Allocate sequence data
    out_data->sequence_data = PF_NEW_HANDLE(sizeof(MySequenceData));

    if (out_data->sequence_data) {
        MySequenceData* seq_data = reinterpret_cast<MySequenceData*>(
            PF_LOCK_HANDLE(out_data->sequence_data));

        // Initialize
        seq_data->value = 0.0;
        seq_data->count = 0;

        PF_UNLOCK_HANDLE(out_data->sequence_data);
    } else {
        err = PF_Err_OUT_OF_MEMORY;
    }

    return err;
}

static PF_Err SequenceSetdown(
    PF_InData* in_data,
    PF_OutData* out_data)
{
    if (in_data->sequence_data) {
        PF_DISPOSE_HANDLE(in_data->sequence_data);
        out_data->sequence_data = NULL;
    }
    return PF_Err_NONE;
}
```

## Suite Usage Patterns

### Suite Handler Pattern
```cpp
static PF_Err UseSuites(PF_InData* in_data)
{
    PF_Err err = PF_Err_NONE;

    // Create suite handler
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    // Use various suites
    PF_ANSICallbacksSuite1* ansiSuite = NULL;
    ERR(suites.ANSICallbacksSuite1()->get(&ansiSuite));

    if (ansiSuite) {
        char buffer[256];
        ansiSuite->sprintf(buffer, "Value: %d", 42);
    }

    // Use world suite
    PF_WorldSuite2* worldSuite = NULL;
    ERR(suites.PFWorldSuite2()->get(&worldSuite));

    if (worldSuite) {
        PF_EffectWorld* new_world = NULL;
        ERR(worldSuite->PF_NewWorld(
            in_data->effect_ref,
            100, 100,
            PF_NewWorldFlag_NONE,
            &new_world));

        // Use world...

        ERR(worldSuite->PF_DisposeWorld(
            in_data->effect_ref,
            &new_world));
    }

    return err;
}
```

### AEFX Suite Scoper Pattern
```cpp
static PF_Err UsePixelFormatSuite(PF_InData* in_data)
{
    PF_Err err = PF_Err_NONE;

    // Scoped suite access
    AEFX_SuiteScoper<PF_PixelFormatSuite1> pixelFormatSuite =
        AEFX_SuiteScoper<PF_PixelFormatSuite1>(
            in_data,
            kPFPixelFormatSuite,
            kPFPixelFormatSuiteVersion1,
            out_data);

    if (pixelFormatSuite) {
        // Add supported pixel formats
        (*pixelFormatSuite->ClearSupportedPixelFormats)(in_data->effect_ref);

        (*pixelFormatSuite->AddSupportedPixelFormat)(
            in_data->effect_ref,
            PF_PixelFormat_ARGB32);

        (*pixelFormatSuite->AddSupportedPixelFormat)(
            in_data->effect_ref,
            PF_PixelFormat_ARGB128);
    }

    return err;
}
```

## Error Handling Patterns

### Standard Error Macro Pattern

> **Important:** The ERR macro does NOT prevent the wrapped function call from executing -- it only prevents overwriting a previous error code. The wrapped function still runs regardless of the current error state.

```cpp
// Error checking macros
#define ERR(FUNC) \
    do { \
        PF_Err _err = (FUNC); \
        if (_err && !err) { err = _err; } \
    } while(0)

#define ERR2(FUNC) \
    do { \
        PF_Err _err = (FUNC); \
        if (_err && !err2) { err2 = _err; } \
    } while(0)

// Usage
static PF_Err FunctionWithErrorHandling()
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;

    ERR(DoSomething());           // Sets err on failure
    ERR(DoSomethingElse());        // Still executes even if err is set;
                                   // only the error code assignment is skipped

    // Cleanup - always executes
    ERR2(CleanupFunction());       // Sets err2, doesn't affect err

    if (!err) err = err2;          // Propagate cleanup errors
    return err;
}
```

### Try-Catch Pattern
```cpp
static PF_Err SafeFunction(PF_InData* in_data)
{
    PF_Err err = PF_Err_NONE;

    try {
        // Potentially throwing code
        std::vector<float> buffer(1000000);

        // SDK calls that might throw
        ERR(PF_ABORT(in_data));

    } catch(PF_Err& thrown_err) {
        err = thrown_err;
    } catch(std::bad_alloc&) {
        err = PF_Err_OUT_OF_MEMORY;
    } catch(...) {
        err = PF_Err_INTERNAL_STRUCT_DAMAGED;
    }

    return err;
}
```

## Pixel Processing Patterns

### Template-Based Processing Pattern
```cpp
template<typename PixelType, typename ValueType>
static void ProcessPixels(
    const PixelType* src,
    PixelType* dst,
    int width,
    int height,
    int src_stride,
    int dst_stride,
    float amount)
{
    for (int y = 0; y < height; y++) {
        const PixelType* src_row = src + y * src_stride;
        PixelType* dst_row = dst + y * dst_stride;

        for (int x = 0; x < width; x++) {
            dst_row[x].alpha = src_row[x].alpha;
            dst_row[x].red = (ValueType)(src_row[x].red * amount);
            dst_row[x].green = (ValueType)(src_row[x].green * amount);
            dst_row[x].blue = (ValueType)(src_row[x].blue * amount);
        }
    }
}

// Usage for different bit depths
// Note: This rowbytes/width heuristic can fail if rowbytes includes padding.
// The recommended approach is PF_WorldSuite2::PF_GetPixelFormat().
static PF_Err ProcessAllBitDepths(
    PF_EffectWorld* input,
    PF_EffectWorld* output,
    float amount)
{
    int bytes_per_pixel = output->rowbytes / output->width;

    if (bytes_per_pixel == sizeof(PF_Pixel8)) {
        ProcessPixels<PF_Pixel8, A_u_char>(
            (PF_Pixel8*)input->data,
            (PF_Pixel8*)output->data,
            output->width, output->height,
            input->rowbytes / sizeof(PF_Pixel8),
            output->rowbytes / sizeof(PF_Pixel8),
            amount);
    } else if (bytes_per_pixel == sizeof(PF_Pixel16)) {
        ProcessPixels<PF_Pixel16, A_u_short>(
            (PF_Pixel16*)input->data,
            (PF_Pixel16*)output->data,
            output->width, output->height,
            input->rowbytes / sizeof(PF_Pixel16),
            output->rowbytes / sizeof(PF_Pixel16),
            amount);
    } else if (bytes_per_pixel == sizeof(PF_PixelFloat)) {
        ProcessPixels<PF_PixelFloat, PF_FpShort>(
            (PF_PixelFloat*)input->data,
            (PF_PixelFloat*)output->data,
            output->width, output->height,
            input->rowbytes / sizeof(PF_PixelFloat),
            output->rowbytes / sizeof(PF_PixelFloat),
            amount);
    }

    return PF_Err_NONE;
}
```

## Utility Patterns

### Progress Reporting Pattern
```cpp
static PF_Err LongOperation(PF_InData* in_data, PF_OutData* out_data)
{
    PF_Err err = PF_Err_NONE;
    int total_steps = 100;

    for (int i = 0; i < total_steps; i++) {
        // Do work...

        // Update progress
        ERR(PF_PROGRESS(in_data, i, total_steps));

        // Check for user abort
        ERR(PF_ABORT(in_data));
    }

    return err;
}
```

### String Handling Pattern
```cpp
static PF_Err GetAndUseString(PF_InData* in_data)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    A_char name[PF_MAX_EFFECT_NAME_LEN + 1] = {0};

    // Get effect name
    ERR(suites.PFAppSuite4()->PF_GetEffectName(
        in_data->effect_ref,
        name));

    // Format string
    A_char formatted[256];
    suites.ANSICallbacksSuite1()->sprintf(
        formatted,
        "Effect: %s, Version: %d.%d",
        name,
        MAJOR_VERSION,
        MINOR_VERSION);

    return err;
}
```

### Time Conversion Pattern
```cpp
static PF_Err ConvertTime(PF_InData* in_data)
{
    // Current time in seconds
    PF_FpLong time_in_seconds =
        (PF_FpLong)in_data->current_time / in_data->time_scale;

    // Frame number
    A_long frame_num = (A_long)(
        time_in_seconds * in_data->local_time_scale + 0.5);

    // Time at specific frame
    A_long frame = 30;  // Frame 30
    A_long time_at_frame =
        (frame * in_data->time_scale) / in_data->local_time_scale;

    return PF_Err_NONE;
}
```

## PiPL Resource Pattern

### Standard PiPL Template (for .r file)
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
            "MyEffect"
        },
        Category {
            "MyCompany"
        },
        CodeWin64X86 {
            "EffectMain"
        },
        AE_PiPL_Version {
            2,
            0
        },
        AE_Effect_Spec_Version {
            PF_PLUG_IN_VERSION,
            PF_PLUG_IN_SUBVERS
        },
        AE_Effect_Version {
            (1L << 16) | 0L  // 1.0
        },
        AE_Effect_Info_Flags {
            0
        },
        AE_Effect_Global_OutFlags {
            0x00008000 |  // PF_OutFlag_DEEP_COLOR_AWARE (1L << 15)
            0x02        // pix independent
        },
        AE_Effect_Global_OutFlags_2 {
            0x00000400 |  // float aware
            0x08000000 |  // threaded rendering (bit 27)
            0x00002000    // smart render
        },
        AE_Effect_Match_Name {
            "ADBE MyCompany MyEffect"
        },
        AE_Reserved_Info {
            0
        }
    }
};
```

> **Note:** Always verify PiPL flag hex values against your SDK headers. PF_OutFlag_DEEP_COLOR_AWARE is `1L << 15` = `0x00008000`. PF_OutFlag2_SUPPORTS_THREADED_RENDERING is `1L << 27` = `0x08000000`. Incorrect values silently disable capabilities.
