# Part 2: Going Deeper -- Bit Depth, SmartFX, GPU, and MFR

You have built the Skeleton example. You have seen your plugin appear in the Effects menu. You have changed a slider value, rebuilt, and watched it take effect. You understand the basic conversation model between AE and your plugin -- commands come in, you respond.

Now it is time to move beyond the basics.

This guide covers the concepts that separate a toy plugin from a professional one: supporting all bit depths, using the modern rendering pipeline (SmartFX), understanding GPU acceleration, and making your plugin work with AE's multi-frame rendering. These are the topics that every commercial plugin must deal with, and they are also where the SDK documentation tends to get dense and confusing.

We will take them one at a time.

---

## 1. Understanding Bit Depth in AE Plugins

You already know this from the AE side. You have set your project to 8-bit, 16-bit, or 32-bit float. You have seen the difference in banding and color precision. You have probably seen a gradient fall apart in 8-bit and look smooth in 32-bit. Maybe you have worked with EXR files in 32-bit float mode and noticed that the sun in your HDR footage actually has values brighter than white.

Now you need to understand what this means for YOUR plugin.

### The Three Pixel Formats

When AE hands your plugin a frame to process, the pixels arrive in one of three formats:

| Format | Type | Channel Range | Size Per Pixel |
|--------|------|---------------|----------------|
| 8-bit | `PF_Pixel8` | 0 -- 255 | 4 bytes |
| 16-bit | `PF_Pixel16` | 0 -- 32768 | 8 bytes |
| 32-bit float | `PF_PixelFloat` | 0.0 -- 1.0 (and beyond) | 16 bytes |

Each pixel has four channels: **alpha, red, green, blue**. Four values, packed together in a structure.

!!! warning "16-bit Max Is 32768, NOT 65535"
    This is one of the most common gotchas in AE plugin development. If you have worked with 16-bit images in other software, you probably expect the range to be 0 to 65535 (the full range of an unsigned 16-bit integer). AE does not use the full range. AE's 16-bit mode uses 0 to 32768 (`PF_MAX_CHAN16`). If you divide by 65535 when normalizing, your colors will be half as bright as they should be. The SDK provides the constant `PF_MAX_CHAN16` (which equals 32768) -- always use it.

!!! warning "Channel Order Is ARGB, Not RGBA"
    After Effects uses **alpha first**: A, R, G, B. This is different from most image libraries and GPU APIs, which typically use RGBA. If you are porting code from another framework or reading pixel data with assumptions about channel order, this will bite you. The pixel structures reflect this:

    ```cpp
    // PF_Pixel8 (8-bit)
    struct {
        A_u_char    alpha;    // Alpha comes first
        A_u_char    red;
        A_u_char    green;
        A_u_char    blue;
    };
    ```

### 32-bit Float and HDR

The 32-bit float format is special. Unlike 8-bit and 16-bit, which have a hard ceiling (255 and 32768 respectively), float values can go above 1.0. A pixel with a red channel of 2.5 is perfectly valid -- it represents a color brighter than "full white." This is how HDR works. When someone is working with EXR footage from a film shoot, those bright highlights in the sky or the reflection on chrome might have values of 5.0, 10.0, or higher.

This means that in your 8-bit and 16-bit processing, you clamp values to the valid range. In 32-bit float, you generally should NOT clamp unless your algorithm specifically requires it. Clamping kills HDR data, and users will notice.

### Making Your Plugin Support All Three

To support all bit depths, you need three things:

**1. Set the flags in GlobalSetup:**

```cpp
static PF_Err GlobalSetup(PF_InData* in_data, PF_OutData* out_data,
                           PF_ParamDef* params[], PF_LayerDef* output)
{
    out_data->my_version = PLUGIN_VERSION_INT;

    // "I understand 16-bit"
    out_data->out_flags = PF_OutFlag_DEEP_COLOR_AWARE;

    // "I understand 32-bit float, and I use the modern render pipeline"
    out_data->out_flags2 = PF_OutFlag2_FLOAT_COLOR_AWARE |
                           PF_OutFlag2_SUPPORTS_SMART_RENDER;

    return PF_Err_NONE;
}
```

**2. Match those flags in your PiPL** (the hex values must be identical -- see the PiPL documentation for how to calculate them).

**3. Implement SmartFX** (PreRender + SmartRender). This is not optional for 32-bit float -- the legacy `PF_Cmd_RENDER` path does not support it.

!!! note "You Must Support SmartFX for 32-bit Float"
    This is a hard rule. If you only implement the legacy `PF_Cmd_RENDER` handler, your plugin will work in 8-bit and 16-bit but AE will never send you 32-bit float data. SmartFX is the gateway to full bit-depth support. We cover it in the next section.

---

## 2. SmartFX: The Modern Rendering Pipeline

If the legacy render path is like a simple recipe -- "here are ingredients, make dinner" -- SmartFX is more like a professional kitchen workflow: "tell us what ingredients you need, we will prep them, then you cook."

### Legacy Render vs. SmartFX

In the Skeleton example, you handled `PF_Cmd_RENDER`. AE called your render function, handed you the input pixels in `params[0]`, gave you an output buffer, and you processed every pixel. Simple and direct.

The problem with this approach:

- **No 32-bit float.** The legacy path does not support it, period.
- **No control over what AE gives you.** You get the full frame whether you need it or not.
- **No buffer expansion.** If your blur needs to read 50 pixels past the edge, tough luck.
- **Less efficient.** AE might render areas you never even look at.

SmartFX solves all of these by splitting rendering into two stages.

### Stage 1: PreRender -- "Tell Me What You Need"

Before AE renders anything, it calls your `PF_Cmd_SMART_PRE_RENDER` handler. This is your chance to say: "I need the input layer, and here is the area I am going to output."

Think of it like a lighting pre-vis. Before you set up the physical lights on set, you plan the shot: where will the camera be, what area of the set needs to be lit, what resources do you need? PreRender is the planning phase.

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data,
                         PF_PreRenderExtra* extra)
{
    PF_Err err = PF_Err_NONE;

    // "I need the input layer"
    PF_RenderRequest req = extra->input->output_request;
    PF_CheckoutResult in_result;

    ERR(extra->cb->checkout_layer(
        in_data->effect_ref,
        PARAM_INPUT,                // Which layer (the input)
        PARAM_INPUT,                // Checkout index
        &req,                       // What area to render
        in_data->current_time,
        in_data->time_step,
        in_data->time_scale,
        &in_result));

    // "Here is what I will output"
    extra->output->result_rect = in_result.result_rect;
    extra->output->max_result_rect = in_result.max_result_rect;

    return err;
}
```

### Stage 2: SmartRender -- "Now Do the Work"

After PreRender, AE prepares everything you asked for. Then it calls `PF_Cmd_SMART_RENDER`. Now you checkout the actual pixels, checkout your parameter values, process, and return.

```cpp
static PF_Err SmartRender(PF_InData* in_data, PF_OutData* out_data,
                           PF_SmartRenderExtra* extra)
{
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
    PF_EffectWorld *input_worldP = NULL, *output_worldP = NULL;

    // Checkout the input pixels
    ERR(extra->cb->checkout_layer_pixels(
        in_data->effect_ref, PARAM_INPUT, &input_worldP));

    // Checkout the output buffer
    ERR(extra->cb->checkout_output(
        in_data->effect_ref, &output_worldP));

    if (!err && input_worldP && output_worldP) {
        // Checkout parameter values
        PF_ParamDef amount_param;
        AEFX_CLR_STRUCT(amount_param);
        ERR(PF_CHECKOUT_PARAM(in_data, PARAM_AMOUNT,
            in_data->current_time, in_data->time_step,
            in_data->time_scale, &amount_param));

        float amount = (float)amount_param.u.fs_d.value / 100.0f;

        // Detect which pixel format AE gave us
        PF_PixelFormat format = PF_PixelFormat_INVALID;
        AEFX_SuiteScoper<PF_WorldSuite2> wsP(
            in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);
        ERR(wsP->PF_GetPixelFormat(input_worldP, &format));

        // Process based on the format
        switch (format) {
            case PF_PixelFormat_ARGB32:    // 8-bit
                err = Process8(in_data, input_worldP, output_worldP, amount);
                break;
            case PF_PixelFormat_ARGB64:    // 16-bit
                err = Process16(in_data, input_worldP, output_worldP, amount);
                break;
            case PF_PixelFormat_ARGB128:   // 32-bit float
                err = ProcessFloat(in_data, input_worldP, output_worldP, amount);
                break;
        }

        // ALWAYS check in the parameter, even if there was an error above
        ERR2(PF_CHECKIN_PARAM(in_data, &amount_param));
    }

    return err;
}
```

### The ERR / ERR2 Pattern

You will see `ERR()` and `ERR2()` all over SDK code. These are macros defined in the SDK headers, and they serve a critical purpose.

`ERR()` checks if the function inside it returned an error. If so, it stores the error code and subsequent `ERR()` calls become no-ops -- they skip their function calls entirely. This prevents a cascade of bad calls after something has already gone wrong.

`ERR2()` is different: it runs its function **even if a previous error occurred**. This is for cleanup code that must always execute.

If you know Python, think of it like this:

```python
# ERR() is like code inside a try block
# ERR2() is like code inside a finally block

try:
    pixels = checkout_pixels()      # ERR(checkout_pixels(...))
    params = checkout_param()       # ERR(checkout_param(...))
    process(pixels, params)         # ERR(process(...))
finally:
    checkin_param(params)           # ERR2(checkin_param(...))
```

The `ERR2` call ensures the parameter gets checked back in whether the processing succeeded or failed. Without it, you leak resources.

!!! tip "The Pattern You Will Use Everywhere"
    ```cpp
    PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;

    // Main work -- stops on first error
    ERR(do_something());
    ERR(do_something_else());

    // Cleanup -- always runs
    ERR2(cleanup_something());
    ERR2(cleanup_something_else());

    return err;
    ```
    Declare both `err` and `err2` at the top. Use `ERR()` for operations that depend on previous success. Use `ERR2()` for cleanup that must always happen.

### Key Differences to Remember

| | Legacy Render | SmartFX |
|---|---|---|
| **Commands** | `PF_Cmd_RENDER` | `PF_Cmd_SMART_PRE_RENDER` + `PF_Cmd_SMART_RENDER` |
| **Parameters** | Directly available via `params[]` | Must checkout with `PF_CHECKOUT_PARAM` |
| **Input pixels** | `params[PARAM_INPUT]->u.ld` | `checkout_layer_pixels()` |
| **32-bit float** | Not supported | Supported |
| **Buffer expansion** | Limited | Full control in PreRender |
| **When to use** | Never, for new plugins | Always |

!!! note "Keep Both Paths If You Want"
    You can implement both `PF_Cmd_RENDER` (for legacy compatibility) and SmartFX (for modern features). AE will prefer the SmartFX path when the `PF_OutFlag2_SUPPORTS_SMART_RENDER` flag is set. But for new plugins, you can skip the legacy path entirely and just implement SmartFX.

---

## 3. Working with Pixels

This is where the real work happens. Everything else is plumbing -- this is where you read pixel values, do math on them, and write new values. Whether you are building a color correction, a blur, a distortion, or a particle system, it all comes down to reading and writing pixels.

### The Rowbytes Rule

This is the single most important thing to understand about pixel access in AE, and getting it wrong is responsible for more plugin crashes than almost any other mistake.

When AE gives you a buffer of pixels (a `PF_EffectWorld`), the pixels are stored in memory row by row. You might expect that each row takes exactly `width * sizeof(pixel)` bytes. **That is often wrong.**

For performance reasons (memory alignment, SIMD optimization, cache line efficiency), AE may add padding bytes at the end of each row. The actual number of bytes per row is stored in `world->rowbytes`, and it may be larger than `width * sizeof(pixel)`.

Here is a visual analogy. Imagine a parking lot with spaces for 100 cars per row. But the lot was designed with extra space at the end of each row for a fire lane. If you try to drive straight from the last space in row 1 to the first space in row 2, you will drive into the fire lane and crash. You need to know the actual row width (including the fire lane) to navigate correctly.

```
Row 0: [pixel][pixel][pixel]...[pixel][padding..]
Row 1: [pixel][pixel][pixel]...[pixel][padding..]
Row 2: [pixel][pixel][pixel]...[pixel][padding..]
```

The correct way to get a pointer to the pixel at position (x, y):

```cpp
// 8-bit
PF_Pixel8* pixel = (PF_Pixel8*)((char*)world->data + y * world->rowbytes) + x;

// 16-bit
PF_Pixel16* pixel = (PF_Pixel16*)((char*)world->data + y * world->rowbytes) + x;

// 32-bit float
PF_PixelFloat* pixel = (PF_PixelFloat*)((char*)world->data + y * world->rowbytes) + x;
```

!!! warning "NEVER Use Width Times Sizeof"
    ```cpp
    // WRONG -- will crash or produce garbled output
    PF_Pixel8* pixel = (PF_Pixel8*)world->data + y * world->width + x;

    // RIGHT -- always use rowbytes for the row stride
    PF_Pixel8* pixel = (PF_Pixel8*)((char*)world->data + y * world->rowbytes) + x;
    ```
    Notice that `rowbytes` is a byte offset, so you must cast `world->data` to `char*` before adding the offset, THEN cast to your pixel type. The `+ x` at the end adds x pixels (not bytes) because at that point you are working with a typed pixel pointer.

### The Iterate Suites: Let AE Handle the Loops

Instead of writing your own nested for-loops to visit every pixel, AE provides iterate suites that do this for you. They handle the rowbytes math, they handle the coordinate bookkeeping, and they can distribute the work across multiple CPU cores for better performance.

You write a small function that processes a single pixel, and AE calls it for every pixel in the buffer:

```cpp
// This function processes ONE pixel. AE calls it for every pixel.
static PF_Err GainPixel8(
    void*        refcon,   // Your custom data (passed through from iterate)
    A_long       xL,       // X coordinate of this pixel
    A_long       yL,       // Y coordinate of this pixel
    PF_Pixel8*   inP,      // Pointer to the input pixel
    PF_Pixel8*   outP)     // Pointer to the output pixel -- write here
{
    float gain = *((float*)refcon);   // Get the gain value we passed

    outP->alpha = inP->alpha;
    outP->red   = (A_u_char)MIN(inP->red   * gain, PF_MAX_CHAN8);
    outP->green = (A_u_char)MIN(inP->green * gain, PF_MAX_CHAN8);
    outP->blue  = (A_u_char)MIN(inP->blue  * gain, PF_MAX_CHAN8);

    return PF_Err_NONE;
}
```

Then in your SmartRender, you call the iterate suite:

```cpp
AEGP_SuiteHandler suites(in_data->pica_basicP);

float gain = 1.5f;  // example value

ERR(suites.Iterate8Suite1()->iterate(
    in_data,
    0,                            // Progress base
    output_worldP->height,        // Number of lines to process
    input_worldP,                 // Input buffer
    NULL,                         // Area (NULL = entire buffer)
    (void*)&gain,                 // Your custom data -- passed to refcon
    GainPixel8,                   // Your per-pixel function
    output_worldP));              // Output buffer
```

For 16-bit, use `Iterate16Suite1` with a function that takes `PF_Pixel16*` parameters. For 32-bit float, use `IterateFloatSuite1` with `PF_PixelFloat*` parameters.

### Handling All Bit Depths: The Template Pattern

A clean approach for supporting all three formats is to write one processing function as a C++ template, then call it from format-specific wrappers:

```cpp
// One function that works for any pixel type
template <typename PixelT, int MaxVal>
static PF_Err GainPixelT(
    void* refcon, A_long xL, A_long yL,
    PixelT* inP, PixelT* outP)
{
    float gain = *((float*)refcon);

    outP->alpha = inP->alpha;
    outP->red   = (decltype(inP->red))MIN(inP->red   * gain, (float)MaxVal);
    outP->green = (decltype(inP->green))MIN(inP->green * gain, (float)MaxVal);
    outP->blue  = (decltype(inP->blue))MIN(inP->blue  * gain, (float)MaxVal);

    return PF_Err_NONE;
}

// The iterate suites need plain function pointers, so wrap them:
static PF_Err GainPixel8(void* r, A_long x, A_long y,
    PF_Pixel8* in, PF_Pixel8* out)
{
    return GainPixelT<PF_Pixel8, PF_MAX_CHAN8>(r, x, y, in, out);
}

static PF_Err GainPixel16(void* r, A_long x, A_long y,
    PF_Pixel16* in, PF_Pixel16* out)
{
    return GainPixelT<PF_Pixel16, PF_MAX_CHAN16>(r, x, y, in, out);
}
```

For 32-bit float, you typically do not clamp (to preserve HDR), so you might handle that case separately:

```cpp
static PF_Err GainPixelFloat(void* refcon, A_long xL, A_long yL,
    PF_PixelFloat* inP, PF_PixelFloat* outP)
{
    float gain = *((float*)refcon);

    outP->alpha = inP->alpha;
    outP->red   = inP->red   * gain;   // No clamping -- HDR is valid
    outP->green = inP->green * gain;
    outP->blue  = inP->blue  * gain;

    return PF_Err_NONE;
}
```

### Detecting Pixel Format

In SmartRender, after checking out the pixel buffers, use the World Suite to determine which format you received:

```cpp
PF_PixelFormat format = PF_PixelFormat_INVALID;
AEFX_SuiteScoper<PF_WorldSuite2> wsP(
    in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);
ERR(wsP->PF_GetPixelFormat(input_worldP, &format));

AEGP_SuiteHandler suites(in_data->pica_basicP);

switch (format) {
    case PF_PixelFormat_ARGB32:   // 8-bit
        ERR(suites.Iterate8Suite1()->iterate(
            in_data, 0, output_worldP->height,
            input_worldP, NULL, (void*)&gain,
            GainPixel8, output_worldP));
        break;

    case PF_PixelFormat_ARGB64:   // 16-bit
        ERR(suites.Iterate16Suite1()->iterate(
            in_data, 0, output_worldP->height,
            input_worldP, NULL, (void*)&gain,
            GainPixel16, output_worldP));
        break;

    case PF_PixelFormat_ARGB128:  // 32-bit float
        ERR(suites.IterateFloatSuite1()->iterate(
            in_data, 0, output_worldP->height,
            input_worldP, NULL, (void*)&gain,
            GainPixelFloat, output_worldP));
        break;
}
```

!!! tip "The Pure Float Workflow"
    An alternative approach is to normalize everything to float (0.0 to 1.0) regardless of input format, do all your processing in float, then convert back. This means you only write one processing path:

    - 8-bit input: `value / 255.0f`
    - 16-bit input: `value / 32768.0f` (use `PF_MAX_CHAN16`)
    - 32-bit input: already float

    This is simpler to maintain but slightly slower for 8-bit due to the conversion overhead. For complex effects with lots of math, the simplicity is worth it.

---

## 4. GPU Rendering

Your effect currently runs on the CPU. Every pixel, every frame, one calculation at a time -- or at best, a handful at a time with SIMD and iterate suite parallelism. A 4K frame has over 8 million pixels, and at 30fps, that is 250 million pixels per second your CPU needs to chew through.

GPUs have thousands of cores and can process millions of pixels simultaneously. If you have ever watched a Resolve grade update in real time on 4K footage, or seen a Sapphire glow render instantly in the viewport, that is GPU acceleration at work.

### What GPU Rendering Means in AE

Instead of iterating over pixels on the CPU, your effect runs a **shader** or **compute kernel** on the GPU. AE manages the GPU memory, gives you device pointers to the input and output buffers, and you dispatch a kernel that processes all pixels in parallel.

AE supports several GPU frameworks:

| Framework | Platform | Notes |
|-----------|----------|-------|
| **CUDA** | NVIDIA GPUs (Windows/Linux) | Best performance on NVIDIA hardware |
| **Metal** | Apple GPUs (macOS) | Required for modern Mac support |
| **OpenCL** | Cross-platform | Deprecated on macOS, still works on Windows |
| **DirectX** | Windows | AE 2025 and later |

### The Flow

1. In GlobalSetup, you set `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` to tell AE you can render on the GPU.
2. AE sends `PF_Cmd_GPU_DEVICE_SETUP` when a GPU becomes available. You check which framework AE is offering (CUDA, Metal, etc.) and accept or decline.
3. During SmartRender, you check whether AE is giving you GPU buffers. If so, `PF_EffectWorld.data` will be `NULL` -- the pixel data lives on the GPU, not in CPU memory. You dispatch a compute kernel to process it.
4. If GPU rendering is not available (wrong GPU, user disabled it, etc.), AE falls back to the CPU path.

### Important GPU Rules

!!! warning "PF_EffectWorld.data Is NULL in GPU Mode"
    When rendering on the GPU, the input and output `PF_EffectWorld` structures still exist, but their `data` pointer is `NULL`. The actual pixel data lives in GPU memory. If you try to dereference `world->data` in GPU mode, you will crash. Always check whether you are in GPU mode before touching CPU pixel pointers.

!!! warning "GPU Buffers Use Straight Alpha"
    CPU buffers in AE are **premultiplied** (RGB channels are pre-multiplied by alpha). GPU buffers are **unpremultiplied** (straight alpha). If you are porting CPU code to GPU, you need to account for this difference, especially in blending and compositing operations.

### What About Vulkan and WebGPU?

You might be wondering about **Vulkan** and **WebGPU** — two modern GPU APIs you hear about a lot in game development and graphics programming.

**Vulkan** is not one of AE's offered GPU frameworks. AE will never send you a Vulkan device context the way it sends you a CUDA or Metal one. But you *can* use Vulkan in your plugin by managing your own Vulkan device and using a **hybrid approach**: AE gives you CPU pixels via SmartRender, you upload them to the GPU, run your Vulkan compute shader, download the results, and hand them back to AE. This adds complexity (you manage the Vulkan instance, device, command pools, synchronization) but gives you access to Vulkan's full feature set and cross-platform GPU compute on both Windows and Linux.

The hybrid approach looks like this:

1. In `GlobalSetup`, create your Vulkan device and compile your shaders.
2. In `SmartRender`, checkout CPU pixels from AE, upload to Vulkan staging buffers, dispatch your compute shader, wait for the GPU to finish, download results back to CPU, return to AE.
3. In `GlobalSetdown`, destroy all Vulkan resources.

The key challenge with Vulkan in AE is **thread safety under MFR**. Multiple render threads will call SmartRender simultaneously, so you need per-thread command pools, mutex-protected queue submission, and careful resource lifecycle management. This is well-documented in the [Vulkan GPU Plugins](../gpu/gpu-memory.md) guide.

**WebGPU** is even newer — it is a cross-platform GPU API originally designed for browsers but increasingly used in native applications. Like Vulkan, AE does not offer WebGPU natively, so you would use the same hybrid approach. WebGPU has a simpler API than Vulkan (no explicit synchronization, automatic resource tracking) which makes it more approachable, but integrating it into an AE plugin is non-trivial and the native C++ ecosystem is still maturing. Consider it an emerging option rather than a proven path.

!!! tip "When to Use Which GPU API"
    - **CUDA** — Best performance on NVIDIA, easiest AE integration, most AE plugins use this
    - **Metal** — Required for Mac, good AE integration
    - **OpenCL** — Cross-platform but deprecated on macOS, simpler than Vulkan
    - **Vulkan** — Cross-platform, most control, but requires hybrid architecture in AE
    - **WebGPU** — Promising future option, simplest cross-platform API, but newest and least proven for AE plugins
    - **DirectX** — Windows only, AE 2025+

    For your first GPU plugin, start with **CUDA** (if targeting NVIDIA) or **Metal** (if targeting Mac). Consider Vulkan or WebGPU only if you need cross-platform GPU support without maintaining separate CUDA and Metal codepaths.

### The Practical Advice

**Do not start with GPU rendering.** Get your CPU version working first. Make sure it handles all three bit depths, produces correct results, and is stable. Then port to GPU as an optimization.

GPU development introduces a whole new layer of complexity: shader compilation, device memory management, synchronization, debugging without breakpoints. It is absolutely worth doing for performance-critical effects, but it is not where you want to be on day one.

When you are ready, study the SDK's GPU examples and the focused GPU docs on this site:

- [CUDA Kernel Patterns](../gpu/cuda-kernels.md)
- [Metal Compute Shaders](../gpu/metal-compute.md)
- [GPU Memory Management](../gpu/gpu-memory.md)
- [OpenCL Setup](../gpu/opencl-setup.md)

!!! tip "CPU Fallback Is Not Optional"
    Always implement a CPU path alongside your GPU path. Not every user has a supported GPU, and AE may fall back to CPU rendering for various reasons (composition settings, preview quality, etc.). Your plugin should produce identical results on both paths.

---

## 5. Multi-Frame Rendering (MFR)

AE used to render one frame at a time. Frame 1 finishes, then frame 2 starts, then frame 3. Sequential. Safe. Slow.

Modern AE renders multiple frames simultaneously on different threads. Frame 1, frame 5, frame 12, and frame 23 might all be rendering at the same instant on different CPU cores. This is called **Multi-Frame Rendering (MFR)**, and it can dramatically speed up RAM previews and exports.

### What This Means for Your Plugin

Your render function (SmartRender) may be called from multiple threads at the same time, processing different frames. Two threads might be inside your SmartRender function simultaneously, each working on a different frame.

If your code is well-behaved -- each call reads its own inputs, does its own math, writes to its own output -- this just works. But if your code touches shared state (global variables, static variables that get modified, shared data structures), you are in trouble. Two threads writing to the same variable at the same time is a **race condition**, and the results are unpredictable: corrupted output, crashes, or bugs that only appear sometimes and are impossible to reproduce.

### The Golden Rule

**No global mutable state.**

```cpp
// FORBIDDEN -- will cause race conditions with MFR
static int g_frameCount = 0;          // Written during render = disaster

// FORBIDDEN -- same problem
static float g_lastBrightness = 0.0f; // Shared between render calls

// SAFE -- constant, never modified
static const float PI = 3.14159265f;

// SAFE -- local to each render call, each thread gets its own copy
static PF_Err SmartRender(...) {
    float localValue = 0.0f;   // On the stack, thread-local
    int pixelCount = 0;        // Also safe -- each call has its own
}
```

The rule is simple: if a variable is declared `static` or at file/global scope, and your render code writes to it, your plugin is not MFR-safe.

### Enabling MFR

If your plugin follows the golden rule (no global mutable state during render), enabling MFR is straightforward:

```cpp
// In GlobalSetup:
out_data->out_flags2 |= PF_OutFlag2_SUPPORTS_THREADED_RENDERING;
```

And the matching hex value in your PiPL.

That is it. If your effect is **stateless** -- each frame depends only on its current inputs and parameter values, with no memory of previous frames -- then MFR support may require nothing more than adding this flag.

### Sequence Data and MFR

If you use sequence data (persistent per-instance data set up in `PF_Cmd_SEQUENCE_SETUP`), there is an important detail. During render, you must use `PF_GetConstSequenceData()` from the `PF_EffectSequenceDataSuite1` instead of directly accessing `in_data->sequence_data`. The const version gives you a read-only pointer that is safe to access from multiple threads.

If you genuinely need to write to sequence data during render (which is rare and should be avoided if possible), you can set `PF_OutFlag2_MUTABLE_RENDER_SEQUENCE_DATA_SLOWER` in addition to the threaded rendering flag. As the name strongly hints, this is slower because AE has to serialize access to the data.

!!! tip "Test MFR Thoroughly"
    MFR bugs are the worst kind of bugs because they are non-deterministic. A race condition might only show up one in fifty renders, or only when the system is under load, or only on machines with many CPU cores. When testing MFR:

    - RAM preview a long sequence multiple times -- do you get identical results each time?
    - Look for visual artifacts that appear on some frames but not others
    - Look for flickering or inconsistent behavior between renders
    - Test with AE's MFR enabled (Preferences > Memory & Performance) with multiple render threads

---

## 6. Buffer Expansion

What if your blur effect needs to read pixels outside the current frame boundary? What if your glow extends 50 pixels past the edge of the layer? What if your drop shadow needs to appear below and to the right of the source?

In standard rendering, your output is the same size as your input. Pixels that would fall outside the layer boundaries simply do not exist. Buffer expansion lets you tell AE: "I need a bigger output than my input."

### The VFX Analogy

Think of it like overscan in a film scan. The negative has image data past the edges of the standard frame. When you need to stabilize the shot, you use that extra data to fill in the edges after the transformation. Buffer expansion is the same idea: you are asking AE for a bigger canvas so your effect has room to grow.

### How It Works with SmartFX

**Step 1: Set the flag in GlobalSetup:**

```cpp
out_data->out_flags |= PF_OutFlag_I_EXPAND_BUFFER;
```

**Step 2: Request the expansion in PreRender:**

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data,
                         PF_PreRenderExtra* extra)
{
    PF_Err err = PF_Err_NONE;
    PF_RenderRequest req = extra->input->output_request;
    PF_CheckoutResult in_result;

    ERR(extra->cb->checkout_layer(
        in_data->effect_ref, PARAM_INPUT, PARAM_INPUT,
        &req, in_data->current_time, in_data->time_step,
        in_data->time_scale, &in_result));

    A_long expansion = 10;  // Pixels to expand on each side

    // Expand the output rectangles
    extra->output->result_rect = in_result.result_rect;
    extra->output->result_rect.left   -= expansion;
    extra->output->result_rect.top    -= expansion;
    extra->output->result_rect.right  += expansion;
    extra->output->result_rect.bottom += expansion;

    extra->output->max_result_rect = in_result.max_result_rect;
    extra->output->max_result_rect.left   -= expansion;
    extra->output->max_result_rect.top    -= expansion;
    extra->output->max_result_rect.right  += expansion;
    extra->output->max_result_rect.bottom += expansion;

    // Tell AE we are returning extra pixels
    extra->output->flags = PF_RenderOutputFlag_RETURNS_EXTRA_PIXELS;

    return err;
}
```

**Step 3: Account for the offset in SmartRender:**

When the output buffer is bigger than the input, you need to know where the input "sits" within the expanded output. AE provides this via `in_data->output_origin_x` and `in_data->output_origin_y`.

!!! warning "The output_origin Trap"
    Forgetting to account for `output_origin` is one of the most common buffer expansion bugs. If your input is 1920x1080 and your output is 1940x1100 (10px expansion on each side), the input image does not start at (0,0) in the output buffer -- it starts at (10, 10). If you write pixels without accounting for this offset, your effect will be shifted by the expansion amount.

### A Practical Example: 10-Pixel Blur

Imagine you are building a simple box blur with a 10-pixel radius. For each output pixel, you need to read input pixels within a 10-pixel neighborhood. But at the edges of the frame, those neighbors do not exist -- they are outside the layer boundary.

With buffer expansion:

1. In PreRender, you request an input area that is 10 pixels larger on every side than the output area.
2. AE gives you an input buffer with 10 extra pixels of "runway" on each side.
3. In SmartRender, your blur can safely read those extra pixels without going out of bounds.
4. The output is the standard size, but the effect uses the expanded input to produce clean edges.

Without buffer expansion, you would either get black edges (reading zero-initialized memory) or have to clamp your sampling coordinates at the edges (which changes the look of the blur near borders).

---

## 7. Parameters Deep Dive

In Part 1, you added a basic float slider. That is the most common parameter type, but the SDK offers much more. Here is the full toolkit.

### Float Sliders (Use These for New Plugins)

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_FLOAT_SLIDERX(
    "Brightness",                // Label in Effect Controls
    -100.0, 100.0,               // Valid range (user can type any value in this range)
    -100.0, 100.0,               // Slider range (what the visual slider covers)
    0.0,                         // Default value
    PF_Precision_HUNDREDTHS,     // Display precision (0.01)
    0,                           // Display flags
    0,                           // Param flags
    PARAM_BRIGHTNESS);           // Disk ID
```

!!! tip "Float vs Fixed vs Integer Sliders"
    The SDK has three slider types: `PF_ADD_FLOAT_SLIDERX` (float), `PF_ADD_FIXED` (fixed-point), and `PF_ADD_SLIDER` (integer). **Use float sliders for new plugins.** Fixed-point is a legacy format from before floating-point math was fast on CPUs. Integer sliders are fine when you genuinely need whole numbers (like "number of iterations"), but float is the default choice.

### Angle Parameters

For rotation controls, AE provides a dial widget:

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_ANGLE(
    "Rotation",       // Label
    45,               // Default (degrees)
    PARAM_ANGLE);     // Disk ID
```

!!! note "Angles Use Fixed-Point"
    Angle parameter values arrive as fixed-point numbers, not floats. To get degrees as a float, use: `float degrees = FIX_2_FLOAT(params[PARAM_ANGLE]->u.ad.value);`. In SmartRender, after checking out the parameter: `float degrees = FIX_2_FLOAT(angle_param.u.ad.value);`

### Point Parameters

Creates a crosshair control on the composition viewer that the user can drag:

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_POINT(
    "Center",         // Label
    50, 50,           // Default X, Y (percentage of layer size)
    0,                // Restrict bounds (0 = no restriction)
    PARAM_CENTER);    // Disk ID
```

Point values are also fixed-point. Convert with `FIX_2_FLOAT()`. The values are in layer coordinates, not composition coordinates.

### Layer Parameters

Let the user select another layer as input -- useful for displacement maps, matte inputs, or texture references:

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_LAYER(
    "Displacement Map",        // Label
    PF_LayerDefault_NONE,      // Default (NONE = no layer selected)
    PARAM_MAP_LAYER);          // Disk ID
```

To get the pixels from the selected layer, you checkout the layer parameter the same way you checkout the input:

```cpp
ERR(extra->cb->checkout_layer(
    in_data->effect_ref,
    PARAM_MAP_LAYER,           // The layer parameter index
    PARAM_MAP_LAYER,           // Checkout index
    &req,
    in_data->current_time,
    in_data->time_step,
    in_data->time_scale,
    &map_result));
```

### Popup Menus (Dropdowns)

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_POPUP(
    "Blend Mode",              // Label
    3,                         // Number of choices
    1,                         // Default selection
    "Normal|Multiply|Screen",  // Choices separated by |
    PARAM_BLEND_MODE);         // Disk ID
```

!!! warning "Popups Are 1-Based"
    The first item is index 1, not 0. If the user selects "Normal" in the example above, `params[PARAM_BLEND_MODE]->u.pd.value` will be `1`. "Multiply" is `2`. "Screen" is `3`. This is different from how arrays work in C++, where the first element is index 0. If you use the popup value to index into a zero-based array, subtract 1.

### Checkbox Parameters

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_CHECKBOX(
    "Invert",          // Label
    "Enabled",         // Comment text (shown next to the checkbox)
    FALSE,             // Default (TRUE = checked)
    0,                 // Flags
    PARAM_INVERT);     // Disk ID
```

Value is `0` (unchecked) or `1` (checked): `bool invert = (params[PARAM_INVERT]->u.bd.value != 0);`

### Color Parameters

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_COLOR(
    "Tint Color",      // Label
    255, 128, 0,       // Default R, G, B (0-255 regardless of project bit depth)
    PARAM_TINT_COLOR); // Disk ID
```

The value is a `PF_Pixel` with 8-bit channels. To use in 16-bit or float processing, normalize it:

```cpp
PF_Pixel color = params[PARAM_TINT_COLOR]->u.cd.value;
float r = color.red   / 255.0f;
float g = color.green / 255.0f;
float b = color.blue  / 255.0f;
```

### Parameter Groups (Topics)

Group related parameters under a collapsible twirly:

```cpp
AEFX_CLR_STRUCT(def);
PF_ADD_TOPICX("Advanced Settings", 0, PARAM_TOPIC_START);

    // Parameters here will be grouped under "Advanced Settings"
    AEFX_CLR_STRUCT(def);
    PF_ADD_FLOAT_SLIDERX("Detail", 0, 100, 0, 100, 50,
        PF_Precision_TENTHS, 0, 0, PARAM_DETAIL);

PF_END_TOPIC(PARAM_TOPIC_END);
```

!!! note "Topics Count as Parameters"
    Both `PF_ADD_TOPICX` and `PF_END_TOPIC` each add an entry to your parameter list. They need their own disk IDs and their own entries in your parameter enum. Do not forget to count them when setting `out_data->num_params`.

### Parameter Supervision: Reacting to Changes

If you want to be notified when the user changes a specific parameter (for example, to grey out other parameters based on a mode selection), add the `PF_ParamFlag_SUPERVISE` flag:

```cpp
AEFX_CLR_STRUCT(def);
def.flags = PF_ParamFlag_SUPERVISE;  // Set BEFORE the PF_ADD macro
PF_ADD_POPUP("Mode", 2, 1, "Simple|Advanced", PARAM_MODE);
```

Then handle `PF_Cmd_USER_CHANGED_PARAM` in your command dispatcher. The `extra` pointer tells you which parameter changed.

### Greying Out Parameters

You can make parameters appear disabled (greyed out) based on the state of other parameters. This requires three pieces:

1. Set `PF_OutFlag_SEND_UPDATE_PARAMS_UI` in GlobalSetup
2. Handle `PF_Cmd_UPDATE_PARAMS_UI` in your dispatcher
3. In your handler, copy the parameter, modify its `ui_flags`, and call `PF_UpdateParamUI`

```cpp
static PF_Err UpdateParamsUI(PF_InData* in_data, PF_OutData* out_data,
                              PF_ParamDef* params[], PF_LayerDef* output)
{
    PF_Err err = PF_Err_NONE;
    AEGP_SuiteHandler suites(in_data->pica_basicP);

    // Check the mode: if "Simple" (value 1), grey out the detail slider
    bool is_simple = (params[PARAM_MODE]->u.pd.value == 1);

    // ALWAYS work on a COPY -- never modify params[] directly
    PF_ParamDef param_copy = *params[PARAM_DETAIL];

    if (is_simple) {
        param_copy.ui_flags |= PF_PUI_DISABLED;    // Grey it out
    } else {
        param_copy.ui_flags &= ~PF_PUI_DISABLED;   // Enable it
    }

    ERR(suites.ParamUtilsSuite3()->PF_UpdateParamUI(
        in_data->effect_ref, PARAM_DETAIL, &param_copy));

    out_data->out_flags |= PF_OutFlag_REFRESH_UI;
    return err;
}
```

!!! warning "Never Modify params[] Directly"
    Always copy the parameter to a local variable, modify the copy, and pass the copy to `PF_UpdateParamUI`. Modifying the original `params[]` entries directly can corrupt AE's internal state.

---

## 8. Where to Go from Here

You now understand the core concepts that every professional AE plugin uses. Here is where to go deeper on each topic:

### Study the SDK Examples

The SDK ships with example plugins that demonstrate specific features. Here are the most useful ones for the topics in this guide:

| Example | What It Demonstrates |
|---------|---------------------|
| **SDK_Noise** | SmartFX rendering, pixel format detection |
| **Gamma_Table** | 16-bit (deep color) pixel processing |
| **Checkout** | Checking out layers and parameters at specific times |
| **Resizer** | Buffer expansion, output larger than input |
| **GLator** | GPU rendering with OpenGL (older, but shows the pattern) |

### Read the Focused Docs

This site has detailed guides on specific topics:

- **[Bit Depth](bit-depth.md)** -- detailed coverage of 8/16/32-bit handling
- **[Buffer Expansion](../guides/buffer-expansion.md)** -- advanced buffer expansion patterns
- **[GPU Framework Notes](../guides/gpu-framework-notes.md)** -- GPU rendering architecture in depth
- **[Parameter Greying Out](../guides/ae-greyout.md)** -- complete parameter UI dynamics guide
- **[Premultiplication and Black Edges](../guides/black-edges-premult.md)** -- solving edge artifacts

### Join the Community

- **[Adobe Community Forums — After Effects SDK](https://community.adobe.com/t5/after-effects-sdk/ct-p/ct-after-effects-sdk)** -- years of archived questions and answers from Adobe engineers and experienced developers
- **Community Discord servers** -- there are active communities of plugin developers who help each other. Search around and you will find them.
- **[DocsForAdobe.dev](https://docsforadobe.dev/)** -- community-maintained SDK documentation that fills gaps in the official docs.

### Build Something Real

Your first real plugin will teach you more than any documentation. Pick a simple effect you wish existed -- maybe a specific color correction, a stylized blur, a procedural pattern -- and build it. You will hit walls. You will stare at debugger output. You will spend an hour on a bug caused by rowbytes. And every single one of those struggles will cement your understanding in a way that reading never can.

Here is a suggested progression:

1. **A tint effect** -- read each pixel, blend toward a target color by an amount. Straightforward per-pixel math, no spatial operations.
2. **A threshold effect** -- convert to luminance, compare to a threshold, output black or white. Introduces decision-making per pixel.
3. **A simple box blur** -- your first spatial filter. Requires reading neighboring pixels, which means dealing with edge cases and buffer expansion.
4. **A vignette** -- distance-based darkening from center. Introduces point parameters and coordinate math.
5. **A displacement map** -- uses a second layer input. Introduces layer parameters and multi-input effects.

Each one builds on the previous and introduces exactly one new concept.

!!! tip "The Learning Curve Is Real, But It Levels Off"
    The first hundred hours of plugin development are the hardest. Every concept is new, every error message is cryptic, and the feedback loop of build-copy-restart-test is slow compared to scripting. But the patterns repeat. Once you have built three or four effects, you will find that new ones come together much faster. The rendering pipeline, the parameter system, the error handling -- all of it becomes muscle memory. You are building a skill set that very few people have, and it only gets easier from here.
