# fad

**15 contributions** to AE SDK community knowledge.

Top topics: `debugging`, `rowbytes`, `raii`, `pixel-format`, `cmake`, `build-system`, `xcode`, `ninja`, `cross-platform`, `param-constraints`

---

## What are best practices for error handling and memory safety in AE SDK plugins?

Recommendations: (1) Wrap AE handles in C++ RAII classes with proper destructors, use smart pointers. (2) Use scope guards (like the pattern from Dr. Dobb's) for cleanup on any exit path. (3) Use std::expected (C++23) or Result types to chain operations safely. (4) Implement an allocation manager that clears allocations when exiting entry functions, since AE handle pointers are not valid between calls. (5) In debug builds, add a cache validator to detect extra memory still allocated at function exit points. (6) Use custom breakpoint assertions that trigger breakpoints when debugger is attached instead of crashing.

*Source: adobe-plugin-devs · 2025-07-27 · Tags: `error-handling`, `memory-safety`, `raii`, `scope-guard`, `smart-pointers`, `best-practice` · [View in Q&A](../qa/error-handling/)*

---

## What causes AE error code 1397908844?

Error 1397908844 is a generic exception handler/catch-all in Adobe's suites manager, dating back to CS2/CS3. It commonly appears when double-freeing/releasing a suite pointer. It appears more often in newer AE versions due to MFR with global suite handles shared between threads. Known triggers include: problems with arb params, crashes after exporting RAM previews, and double-releasing suites. The underlying issue is usually in the plugin's suite management code.

*Source: adobe-plugin-devs · 2025-07-05 · Tags: `error-code`, `suite-manager`, `double-free`, `mfr`, `debugging` · [View in Q&A](../qa/error-code/)*

---

## How do you debug AE plugins built with Rust on macOS?

Use VSCode with the CodeLLDB extension. Create a launch.json that launches AE directly and specifies 'sourceLanguages': ['rust']. The sourceLanguages line is important - without it, launching AE from LLDB can cause crashes. Use a preLaunchTask to build the plugin before launching.

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build",
      "sourceLanguages": ["rust"]
    }
  ]
}
```

*Source: adobe-plugin-devs · 2025-04-18 · Tags: `rust`, `debugging`, `lldb`, `vscode`, `macos` · [View in Q&A](../qa/rust/)*

---

## Why might Adobe report high numbers of After Effects force-quits during launch?

A significant portion of force-quit reports during AE launch may actually be caused by plugin developers stopping AE from their debuggers (e.g., hitting 'Stop Debugging' in Visual Studio or Xcode while AE is attached). This registers as a force quit in Adobe's telemetry but is not an actual crash or bug.

*Source: adobe-plugin-devs · 2025-01-18 · Tags: `debugging`, `force-quit`, `crash-reports`, `telemetry`, `visual-studio` · [View in Q&A](../qa/debugging/)*

---

## Why do rowbytes sometimes differ from width * bytes_per_pixel?

Rowbytes can be larger than expected because AE uses it as an optimization for layer cropping. When a layer is cropped (e.g., dragged partially off the comp), AE updates width/height and moves the data pointer to the new top-left pixel, but leaves rowbytes unchanged. This avoids memory reallocation. After a cache purge, rowbytes may return to the expected value. Always use rowbytes (not width * pixel_size) when iterating rows.

*Source: adobe-plugin-devs · 2024-12-27 · Tags: `rowbytes`, `memory-layout`, `cropping`, `pixel-access`, `buffer-stride` · [View in Q&A](../qa/rowbytes/)*

---

## How do you detect bit depth of a PF_LayerDef without the World Suite?

You can check world_flags: PF_WorldFlag_DEEP indicates 16bpc, and PF_WorldFlag_RESERVED1 indicates 32bpc (undocumented). However, this uses private/undocumented API and may not work in future versions. The official way in Premiere is PF_PixelFormatSuite1 (see SDK_Noise sample). You can also calculate from the rowbytes:width ratio, though rowbytes can be larger than expected due to AE's buffer cropping optimization.

```cpp
int get_bitdepth(const PF_LayerDef& layer) {
    if (layer.world_flags & PF_WorldFlag_DEEP) {
        return 16;
    } if (layer.world_flags & PF_WorldFlag_RESERVED1) {
        return 32;
    } else {
        return 8;
    }
}
```

*Source: adobe-plugin-devs · 2024-12-26 · Tags: `bit-depth`, `world-flags`, `pf-layer-def`, `undocumented`, `pixel-format` · [View in Q&A](../qa/bit-depth/)*

---

## How does the 'Blend colors using 1.0 gamma' checkbox affect compositing?

The 'Blend colors using 1.0 gamma' checkbox only affects blending modes between layers, not the final composite of a composition onto its background color. This is why a semitransparent layer in gamma 1.0 looks different atop a black background vs. atop nothing (transparency) - the compositing onto the background doesn't respect the gamma setting. The Solid Composite effect does respect this setting, however.

*Source: adobe-plugin-devs · 2024-11-15 · Tags: `gamma`, `compositing`, `blending`, `color-management` · [View in Q&A](../qa/gamma/)*

---

## Is PF_LayerDef::rowbytes always a multiple of 4 in After Effects?

Yes, you can be 100% confident that rowbytes will always be a multiple of sizeof(PF_Pixel8), i.e., 4 bytes. However, 16-byte alignment is NOT always guaranteed. According to Daniel Wilk from Adobe, rowbytes is technically arbitrary and your code should be prepared to deal with any stride. In practice it is often aligned to 16-byte boundaries for SIMD and GPU performance. Important: a buffer might be a sub-reference to another buffer, so you should never write into bytes outside your image reference frame into the rowbytes gutter. If rowbytes were not a multiple of the pixel type's alignment, you would be dereferencing misaligned pointers, which is undefined behavior per the C/C++ spec (C spec 6.3.2.3 paragraph 7).

*Source: adobe-plugin-devs · 2024-09-23 · Tags: `rowbytes`, `pf-layerdef`, `memory-alignment`, `pixel-buffer`, `simd`, `undefined-behavior` · [View in Q&A](../qa/rowbytes/)*

---

## What are the benefits of SmartFX (SmartRender) over regular Render?

SmartFX: (1) Only requests pixels needed for the specific processing area via PreRender. (2) Required for 32bpc support. (3) Allows checking out frames with prior effects applied via checkout_layer_pixels. (4) Supports MFR (Multi-Frame Rendering) better. (5) Provides more control over buffer management. Even if your effect needs the full image, SmartFX is recommended for 32bpc support and future-proofing.

*Source: adobe-plugin-devs · 2024-06-10 · Tags: `smart-render`, `smartfx`, `render`, `32bpc`, `pre-render` · [View in Q&A](../qa/smart-render/)*

---

## How do you properly copy pixel data from OpenCV cv::Mat to AE PF_LayerDef?

Never manually allocate layerDef->data - AE owns that memory. Copy pixel data respecting rowbytes alignment. Use the sampleIntegral32 pattern to correctly index pixels. AE uses ARGB format while OpenCV uses BGRA, so channel swizzling is needed. When creating a cv::Mat pointing to AE layer data, pass layerDef->rowbytes as the step parameter. For performance, use memcpy per row (if same pixel format) and the IterateSuite for multi-threaded processing.

```cpp
PF_Pixel *sampleIntegral32(PF_EffectWorld &def, int x, int y) {
    return (PF_Pixel*)((char*)def.data +
        (y * def.rowbytes) +
        (x * sizeof(PF_Pixel)));
}

static void copyMatToLayer(cv::Mat mat, PF_LayerDef* layer) {
    for (int y = 0; y < layer->height; ++y) {
        for (int x = 0; x < layer->width; ++x) {
            cv::Vec4b pixel = mat.at<cv::Vec4b>(y, x);
            PF_Pixel& aePixel = *sampleIntegral32(*layer, x, y);
            aePixel.alpha = pixel[3];
            aePixel.red = pixel[2];
            aePixel.green = pixel[1];
            aePixel.blue = pixel[0];
        }
    }
}

// Create cv::Mat from AE layer with proper stride:
cv::Mat argb(height, width, CV_8UC4, pixelData, layerDef->rowbytes);
```

*Source: adobe-plugin-devs · 2024-06-09 · Tags: `opencv`, `pixel-format`, `argb`, `bgra`, `rowbytes`, `cv-mat` · [View in Q&A](../qa/opencv/)*

---

## When is PF_Cmd_GLOBAL_SETUP called - during AE loading or when applying the effect?

In After Effects, GlobalSetup is called the first time the user applies the effect to a layer, not during the AE loading screen (during loading, AE only scans PiPLs). In Premiere Pro, GlobalSetup is called when the app is loading. You can show dialogs during GlobalSetup in AE since it happens after the app is fully loaded.

*Source: adobe-plugin-devs · 2024-05-30 · Tags: `global-setup`, `plugin-lifecycle`, `premiere`, `initialization` · [View in Q&A](../qa/global-setup/)*

---

## How do you use RenderDoc to debug GPU plugins in After Effects?

Use RenderDoc's in-application API with process injection. Steps: (1) Enable 'Enable process injection' in RenderDoc's Tools > Settings. (2) Compile your plugin using RenderDoc's API - in GlobalSetup, check for renderdoc.dll with GetModuleHandleA and get the API pointer. (3) Launch AE first, then inject RenderDoc via File > Inject into Process. (4) RenderDoc must be injected before applying your plugin. (5) Use rdoc_api in your render function to manually begin/end frame capture.

```cpp
// In PF_Cmd_GLOBAL_SETUP:
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll")) {
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
// Then use rdoc_api->StartFrameCapture() / EndFrameCapture() in render
```

*Source: adobe-plugin-devs · 2024-02-29 · Tags: `gpu`, `renderdoc`, `debugging`, `opengl`, `process-injection` · [View in Q&A](../qa/gpu/)*

---

## Why can't macOS Instruments detect PF_EffectWorld memory leaks?

Instruments won't report PF_EffectWorld leaks because AE legitimately thinks you're holding important data in those worlds. AE allocates the worlds deep in its engine, not directly in your plugin code. While memory allocated with new/malloc within the plugin gets reported accurately (because Instruments has more context), AE-managed allocations are invisible to the leak detector. Creating C++ RAII wrappers around AE SDK objects for automatic memory management is recommended.

*Source: adobe-plugin-devs · 2023-11-22 · Tags: `memory-leak`, `instruments`, `mac`, `debugging`, `effect-world`, `raii` · [View in Q&A](../qa/memory-leak/)*

---

## How do you restrict two float sliders relative to each other (e.g., Start cannot be >= End)?

Modifying param values in UserChangedParam works but causes problems with keyframing: a keyframe with an acceptable value can later become unacceptable, locking the value and preventing keyframe deletion. A better approach is to use the min()/max() pattern: treat the parameters as generic 'endpoints' and in your render code use min(A,B) as Start and max(A,B) as End.

*Source: adobe-plugin-devs · 2023-10-02 · Tags: `param-constraints`, `sliders`, `user-changed-param`, `keyframing` · [View in Q&A](../qa/param-constraints/)*

---

## Are there CMake-based build systems for AE plugins?

Yes, there are several examples on GitHub: Vulkanator (https://github.com/Wunkolo/Vulkanator) uses CMake, and after_effects_cmake (https://github.com/mobile-bungalow/after_effects_cmake) is based on Vulkanator's setup. There's also a Rust-based project (https://github.com/virtualritz/after-effects) with its own PiPL compiler written in Rust. On Mac, you can use the Rez tool for .rsrc files and potentially use Ninja for faster incremental compilation.

*Source: adobe-plugin-devs · 2023-07-06 · Tags: `cmake`, `build-system`, `xcode`, `ninja`, `cross-platform` · [View in Q&A](../qa/cmake/)*

---
