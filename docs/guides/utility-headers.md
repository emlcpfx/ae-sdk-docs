# SDK Utility Headers: ArbParseHelper, ChannelDepthTpl, String_Utils, and More

The After Effects SDK ships a collection of utility files in `Examples/Util/` that solve common problems plugin developers face repeatedly. These are not part of the core SDK headers -- they are convenience code provided as source, ready to be included directly in your project.

This guide covers each utility file, what it does, and how to use it.

---

## AEFX_ArbParseHelper.h / .c

**Purpose**: Parsing tab-delimited text data for arbitrary parameter PRINT and SCAN callbacks.

When you implement arbitrary parameters (`PF_ADD_ARBITRARY2`), After Effects may ask your plugin to serialize the data to a text string (PRINT) and parse it back (SCAN). This helper provides functions for reading and writing tab-delimited "cells" in such strings.

### Constants

```cpp
#define AEFX_Char_TAB    '\t'    // Cell separator
#define AEFX_Char_EOL    '\r'    // End of line
#define AEFX_Char_SPACE  ' '     // Trimmed from cells
#define AEFX_CELL_SIZE   256     // Maximum characters per cell
```

### Error Codes

```cpp
enum {
    AEFX_ParseError_EXPECTING_MORE_DATA = 0x00FEBE00,
    AEFX_ParseError_APPEND_ERROR,
    AEFX_ParseError_EXPECTING_A_NUMBER,
    AEFX_ParseError_MATCH_ERROR
};
```

### Functions

#### AEFX_AppendText

```cpp
PF_Err AEFX_AppendText(
    A_char       *srcAC,             // Source string to append
    const A_u_long dest_sizeL,       // Max size of destination buffer
    A_char       *destAC,            // Destination buffer
    A_u_long     *current_indexPLu); // Current write position (updated)
```

Appends text to a buffer at the given index, updating the index. Used when building PRINT output strings.

**Example -- Serializing arbitrary data to text:**

```cpp
PF_Err MyArb_Print(PF_InData *in_data, PF_OutData *out_data,
                   PF_ArbitraryH arbH, A_u_long buf_size,
                   A_char *bufAC)
{
    PF_Err err = PF_Err_NONE;
    A_u_long index = 0;
    MyArbData *dataP = *(MyArbData**)arbH;
    char temp[64];

    // Write header
    ERR(AEFX_AppendText("MyArb", buf_size, bufAC, &index));
    ERR(AEFX_AppendText("\t", buf_size, bufAC, &index));

    // Write numeric value
    sprintf(temp, "%f", dataP->value);
    ERR(AEFX_AppendText(temp, buf_size, bufAC, &index));
    ERR(AEFX_AppendText("\t", buf_size, bufAC, &index));

    // Write another value
    sprintf(temp, "%d", dataP->mode);
    ERR(AEFX_AppendText(temp, buf_size, bufAC, &index));

    return err;
}
```

#### AEFX_ParseCell

```cpp
PF_Err AEFX_ParseCell(
    PF_InData    *in_data,
    PF_OutData   *out_data,
    const A_char *startPC,           // Full input string
    A_u_long     *current_indexPL,   // Current read position (updated)
    A_char       *bufAC);            // Output buffer (AEFX_CELL_SIZE)
```

Reads the next tab-delimited cell from the input string. Trims leading and trailing spaces. Advances the index past the cell and its separator.

#### AEFX_ParseFpLong

```cpp
PF_Err AEFX_ParseFpLong(
    PF_InData    *in_data,
    PF_OutData   *out_data,
    const A_char *startPC,           // Full input string
    A_u_long     *current_indexPL,   // Current read position (updated)
    PF_FpLong    *dPF);              // Output double value
```

Reads the next cell and converts it to a `PF_FpLong` (double) via `strtod`. Returns `AEFX_ParseError_EXPECTING_A_NUMBER` if the cell does not start with a digit, decimal point, or minus sign.

#### AEFX_MatchCell

```cpp
PF_Err AEFX_MatchCell(
    PF_InData    *in_data,
    PF_OutData   *out_data,
    const A_char *strPC,             // Expected string
    const A_char *startPC,           // Full input string
    A_u_long     *current_indexPL,   // Current read position (updated)
    PF_Boolean   *matchPB0);         // Output: did it match? (NULL = require match)
```

Reads the next cell and compares it to an expected string. If `matchPB0` is NULL, a mismatch is an error. If non-NULL, the match result is stored and the index is only advanced on match.

**Example -- Parsing arbitrary data from text:**

```cpp
PF_Err MyArb_Scan(PF_InData *in_data, PF_OutData *out_data,
                  const A_char *bufAC, A_u_long buf_size,
                  PF_ArbitraryH arbH)
{
    PF_Err err = PF_Err_NONE;
    A_u_long index = 0;
    PF_FpLong value;
    PF_FpLong mode;
    MyArbData *dataP = *(MyArbData**)arbH;

    // Verify header
    ERR(AEFX_MatchCell(in_data, out_data, "MyArb", bufAC, &index, NULL));

    // Read numeric values
    ERR(AEFX_ParseFpLong(in_data, out_data, bufAC, &index, &value));
    ERR(AEFX_ParseFpLong(in_data, out_data, bufAC, &index, &mode));

    if (!err) {
        dataP->value = value;
        dataP->mode  = (int)mode;
    }
    return err;
}
```

---

## AEFX_ChannelDepthTpl.h

**Purpose**: A C++ template system for writing pixel-processing code once that works across 8-bit and 16-bit pixel formats.

This header defines `PixelTraits<>` template specializations for `PF_Pixel8` and `PF_Pixel16`, providing type information and a LUT lookup function for each depth.

### Template Structure

```cpp
// Base template (never instantiated directly)
template <typename Pixel>
struct PixelTraits {
    typedef int PixType;
    typedef int DataType;
    static DataType LutFunc(DataType input, const DataType *map);
    enum { max_value = 0 };
};
```

### 8-bit Specialization

```cpp
template <>
struct PixelTraits<PF_Pixel8> {
    typedef PF_Pixel8   PixType;
    typedef u_char      DataType;
    static DataType
    LutFunc(DataType input, const DataType *map) { return map[input]; }
    enum { max_value = PF_MAX_CHAN8 };  // 255
};
```

For 8-bit pixels, the LUT function is a direct array lookup -- no interpolation needed since the table can cover all 256 possible values.

### 16-bit Specialization

```cpp
template <>
struct PixelTraits<PF_Pixel16> {
    typedef PF_Pixel16  PixType;
    typedef u_short     DataType;
    static u_short
    LutFunc(u_short input, const u_short *map);
    enum { max_value = PF_MAX_CHAN16 };  // 32768
};

inline u_short
PixelTraits<PF_Pixel16>::LutFunc(u_short input, const u_short *map)
{
    u_short index  = input >> (15 - PF_TABLE_BITS);
    uint32_t fract = input & ((1 << (15 - PF_TABLE_BITS)) - 1);
    A_long  result = map[index];

    if (fract) {
        result += ((((A_long)map[index + 1] - result) * fract) +
                   (1 << (14 - PF_TABLE_BITS))) >> (15 - PF_TABLE_BITS);
    }
    return (u_short)result;
}
```

For 16-bit, a full table would need 32,769 entries. Instead, the SDK uses a smaller table (size determined by `PF_TABLE_BITS`, typically 12) and **linear interpolation** between adjacent entries. The function:

1. Extracts the table index from the high bits of the input
2. Extracts the fractional part from the low bits
3. Linearly interpolates between `map[index]` and `map[index + 1]`

### Usage Pattern

Write a single templated pixel function:

```cpp
template <typename PixelT>
static PF_Err ApplyLUT(
    void *refcon, A_long x, A_long y,
    PixelT *inP, PixelT *outP)
{
    typedef PixelTraits<PixelT> Traits;
    typedef typename Traits::DataType Chan;

    MyRefcon *info = (MyRefcon*)refcon;

    outP->alpha = inP->alpha;
    outP->red   = Traits::LutFunc(inP->red,   info->lut_r);
    outP->green = Traits::LutFunc(inP->green,  info->lut_g);
    outP->blue  = Traits::LutFunc(inP->blue,   info->lut_b);

    return PF_Err_NONE;
}
```

Then dispatch based on bit depth in your render function:

```cpp
if (PF_WORLD_IS_DEEP(output)) {
    ERR(suites.Iterate16Suite1()->iterate(..., ApplyLUT<PF_Pixel16>, ...));
} else {
    ERR(suites.Iterate8Suite1()->iterate(..., ApplyLUT<PF_Pixel8>, ...));
}
```

> **Note**: No specialization exists for `PF_PixelFloat` (32-bit). For float pixels, you should compute the transfer function directly per pixel since the input domain is continuous and unbounded.

---

## String_Utils.h / .c

**Purpose**: A minimal string lookup mechanism used by nearly every SDK example.

### The Header

```cpp
// String_Utils.h
#ifdef __cplusplus
extern "C" {
#endif
A_char *GetStringPtr(int strNum);
#ifdef __cplusplus
}
#endif

#define STR(_foo) GetStringPtr(_foo)
```

### How It Works

Each plugin provides its own implementation of `GetStringPtr()` by defining a static string table:

```cpp
// YourPlugin_Strings.cpp
typedef struct {
    unsigned long   index;
    char            str[256];
} TableString;

TableString g_strs[StrID_NUMTYPES] = {
    StrID_NONE,          "",
    StrID_Name,          "My Plugin",
    StrID_Description,   "Does something useful.\rCopyright 2024.",
    // ...
};

char *GetStringPtr(int strNum)
{
    return g_strs[strNum].str;
}
```

> **Important**: The `String_Utils.c` file in the SDK Util directory contains a reference implementation of `GetStringPtr` that assumes a global `g_strs` array. However, most SDK examples provide their own `*_Strings.cpp` file with a local implementation. If you include both, you will get linker errors from duplicate symbols. Use one or the other.

### Convention

- Define string IDs in a `*_Strings.h` enum
- Implement the table in a `*_Strings.cpp` file
- Use `STR(StrID_Name)` everywhere strings are needed
- This makes localization possible by swapping the table at runtime

---

## Smart_Utils.h / .cpp

**Purpose**: Rectangle utility functions for SmartFX `PreRender` and `SmartRender` implementations.

### Functions

#### IsEmptyRect

```cpp
PF_Boolean IsEmptyRect(const PF_LRect *r);
```

Returns `TRUE` if `left >= right` or `top >= bottom`. Used to check whether a computed rectangle has any area.

#### UnionLRect

```cpp
void UnionLRect(const PF_LRect *src, PF_LRect *dst);
```

Computes the union of two rectangles, storing the result in `dst`. Handles the case where `dst` is empty (copies `src`) or `src` is empty (no-op).

```cpp
// Example: Computing the output rect in PreRender
PF_LRect result_rect = {0};
UnionLRect(&in_result.result_rect, &result_rect);
// result_rect now contains the union
```

#### IsEdgePixel

```cpp
PF_Boolean IsEdgePixel(PF_LRect *rectP, A_long x, A_long y);
```

Returns `TRUE` if the pixel at (x, y) lies on the edge of the rectangle. Useful for border effects or edge detection.

### Typical Usage

These functions appear in virtually every SmartFX plugin's `PreRender` handler:

```cpp
PF_Err SmartPreRender(PF_InData *in_data, PF_OutData *out_data,
                      PF_PreRenderExtra *extra)
{
    PF_Err err = PF_Err_NONE;
    PF_RenderRequest req = extra->input->output_request;
    PF_CheckoutResult in_result;

    // Check out input at the requested rect
    ERR(extra->cb->checkout_layer(in_data->effect_ref, 0,
                                   0, &req, in_data->current_time,
                                   in_data->time_step, in_data->time_scale,
                                   &in_result));

    // Union with any existing result
    UnionLRect(&in_result.result_rect, &extra->output->result_rect);

    // Check for degenerate rects
    if (!IsEmptyRect(&extra->output->result_rect)) {
        extra->output->solid = in_result.solid;
    }

    return err;
}
```

---

## AEGP_Utils.h / .cpp

**Purpose**: A helper function for AEGP plugins to retrieve the first layer in the first composition.

### Function

```cpp
A_Err GetNewFirstLayerInFirstComp(
    SPBasicSuite  *sP,
    AEGP_LayerH   *first_layerPH);
```

This function:

1. Gets the first project via `ProjSuite5()->AEGP_GetProjectByIndex(0, ...)`
2. Iterates project items to find the first composition (`AEGP_ItemType_COMP`)
3. Gets the first layer in that composition

### Implementation Details

The function uses multiple AEGP suites:

```cpp
A_Err GetNewFirstLayerInFirstComp(SPBasicSuite *sP, AEGP_LayerH *first_layerPH)
{
    A_Err err = A_Err_NONE;
    AEGP_ItemH      itemH    = NULL;
    AEGP_ItemType   type     = AEGP_ItemType_NONE;
    AEGP_CompH      compH    = NULL;
    AEGP_ProjectH   projH    = NULL;
    A_long          num_layersL = 0;

    AEGP_SuiteHandler suites(sP);

    ERR(suites.ProjSuite5()->AEGP_GetProjectByIndex(0, &projH));
    ERR(suites.ItemSuite8()->AEGP_GetFirstProjItem(projH, &itemH));
    ERR(suites.ItemSuite6()->AEGP_GetItemType(itemH, &type));

    while ((itemH != NULL) && (type != AEGP_ItemType_COMP)) {
        ERR(suites.ItemSuite6()->AEGP_GetNextProjItem(projH, itemH, &itemH));
        ERR(suites.ItemSuite6()->AEGP_GetItemType(itemH, &type));
    }

    if (!err && (type == AEGP_ItemType_COMP)) {
        err = suites.CompSuite4()->AEGP_GetCompFromItem(itemH, &compH);
    }
    if (!err && compH) {
        err = suites.LayerSuite5()->AEGP_GetCompNumLayers(compH, &num_layersL);
    }
    if (!err && num_layersL) {
        err = suites.LayerSuite5()->AEGP_GetCompLayerByIndex(compH, 0, first_layerPH);
    }

    return err;
}
```

This is primarily used by AEGP example plugins (like Artie, Easy Cheese) that need a quick way to find a layer to operate on. It demonstrates the standard pattern for navigating the AE project model: Project -> Items -> Comp -> Layers.

> **Pitfall**: This function does not handle the case where the project has no compositions. In that scenario, `itemH` becomes NULL and the subsequent `AEGP_GetItemType` call on a NULL handle can cause issues. Always guard against empty projects in production code.

---

## DirectXUtils.h / .cpp

**Purpose**: DirectX 12 helper classes for GPU-accelerated effect plugins on Windows.

This is a substantial utility providing a complete DirectX 12 compute pipeline wrapper. It is used by the SDK's DirectX GPU effect examples.

### Key Types

#### DXContext

Manages the core DirectX 12 state:

```cpp
struct DXContext {
    bool Initialize(ID3D12Device* inDevice, ID3D12CommandQueue* inCommandQueue);
    bool LoadShader(LPCWSTR inCSOPath, LPCWSTR inRootSignaturePath,
                    ShaderObjectPtr& outShaderObject);
    bool ReserveDescriptorHeapSlots(UINT inNumDescriptors, UINT& outHeapOffset);
    void CloseWaitAndReset();

    // D3D12 objects
    ComPtr<ID3D12Device>              mDevice;
    ComPtr<ID3D12CommandQueue>        mCommandQueue;
    ComPtr<ID3D12CommandAllocator>    mCommandAllocator;
    ComPtr<ID3D12GraphicsCommandList> mCommandList;
    ComPtr<ID3D12DescriptorHeap>      mDescriptorHeap;
    ComPtr<ID3D12Fence>               mFence;
    // ...
};
```

`Initialize()` takes a device and command queue (provided by AE's GPU framework via `PF_GPUDeviceSetupExtra`) and creates the command allocator, command list, descriptor heap, and fence needed for compute shader dispatch.

#### ShaderObject

Bundles a root signature and pipeline state:

```cpp
struct ShaderObject {
    ComPtr<ID3D12RootSignature> mRootSignature;
    ComPtr<ID3D12PipelineState> mPipelineState;
};
```

#### DXShaderExecution

Manages a single compute shader dispatch:

```cpp
class DXShaderExecution {
public:
    DXShaderExecution(const DXContextPtr& inContext,
                      const ShaderObjectPtr& inShaderObject,
                      const UINT inNumDescriptors);

    bool SetParamBuffer(void* inParamBuffer, UINT inParamBufferSize);
    bool SetUnorderedAccessView(ID3D12Resource* inBuffer, UINT inBufferSize);
    bool SetShaderResourceView(ID3D12Resource* inBuffer, UINT inBufferSize);
    bool Execute(const UINT inGridSizeX, const UINT inGridSizeY);
};
```

### Descriptor Binding Convention

The utilities follow a fixed binding convention matching the HLSL root signature:

| Slot Index | Type | Binding |
|---|---|---|
| 0 (`kCBVSlotIndex`) | Constant Buffer View | Parameter data |
| 1 (`kUAVSlotIndex`) | Unordered Access View | Output buffer(s) |
| 2 (`kSRVSlotIndex`) | Shader Resource View | Input buffer(s) |

### Usage Example

```cpp
// In your SmartRender GPU path:
DXContextPtr context = std::make_shared<DXContext>();
context->Initialize(device, commandQueue);

ShaderObjectPtr shader = std::make_shared<ShaderObject>();
context->LoadShader(L"MyShader", L"MyShader", shader);

// 3 descriptors: 1 CBV (params) + 1 UAV (output) + 1 SRV (input)
DXShaderExecution exec(context, shader, 3);
exec.SetParamBuffer(&myParams, sizeof(myParams));
exec.SetUnorderedAccessView(outputBuffer, outputSize);
exec.SetShaderResourceView(inputBuffer, inputSize);
exec.Execute(gridX, gridY);
```

#### GetShaderPath

```cpp
bool GetShaderPath(const wchar_t* inModuleName,
                   std::wstring& outCSOPath,
                   std::wstring& outSigPath);
```

Locates compiled shader files relative to the plugin DLL. Expects shaders in a `DirectX_Assets/` subfolder next to the plugin:

```
MyPlugin.aex
DirectX_Assets/
    MyShader.cso          <- Compiled Shader Object
    MyShader.rs           <- Root Signature blob
```

> **Important**: `DirectXUtils` is Windows-only. There is no macOS equivalent in the SDK. For cross-platform GPU effects, consider the Metal framework on macOS or the newer WebGPU approach.

> **Performance Note**: The `Execute()` method calls `CloseWaitAndReset()` synchronously, which stalls the CPU until the GPU finishes. The SDK comments this as "sub-optimal" and suggests deferred execution for production use.

---

## Summary Table

| File | Language | Purpose | Typical Users |
|---|---|---|---|
| `AEFX_ArbParseHelper.h/.c` | C | Parse/generate tab-delimited text for arbitrary params | Effects with `PF_ADD_ARBITRARY2` |
| `AEFX_ChannelDepthTpl.h` | C++ | Templated pixel traits for 8/16-bit LUT operations | Effects with `PF_OutFlag_DEEP_COLOR_AWARE` |
| `String_Utils.h/.c` | C | `STR()` macro and `GetStringPtr()` pattern | Nearly every SDK example |
| `Smart_Utils.h/.cpp` | C++ | Rectangle utility functions | SmartFX plugins (`PreRender`) |
| `AEGP_Utils.h/.cpp` | C++ | Find first layer in first comp | AEGP command plugins |
| `DirectXUtils.h/.cpp` | C++ | DirectX 12 compute shader wrapper | GPU-accelerated effects (Windows) |

---

## How to Include These in Your Project

1. Add `Examples/Util/` to your include path
2. Add the `.c` or `.cpp` files you need to your build
3. For `String_Utils`, provide your own `GetStringPtr()` implementation or include the default `String_Utils.c`
4. For `AEFX_ChannelDepthTpl.h`, just `#include` it -- it is header-only
5. For `DirectXUtils`, link against `d3d12.lib` and `d3dcompiler.lib`

> **Pitfall**: Several of these utilities use global symbols (`g_strs`, `GetStringPtr`). If your plugin uses multiple translation units that each define their own string tables, ensure only one `GetStringPtr` symbol is compiled and linked, or use namespaces/static linkage to avoid collisions.
