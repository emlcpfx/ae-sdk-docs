# Alex Bizeau (maxon)

**19 contributions** to AE SDK community knowledge.

Top topics: `aegp`, `premiere`, `global-data`, `smart-render`, `global-setup`, `pipl`, `extendscript`, `scope-guard`, `custom-ui`, `arb-data`

---

## Where can I find resources and community support for developing OFX plugins for Flame on Linux?

There are several resources: (1) The official OpenFX Slack channel (now under ASWF - Academy Software Foundation) is the recommended place, though access may be restricted to specific email domains like linuxfoundation.org or aswf.io. (2) There is an unofficial OFX Discord server with a dedicated Flame channel, though it's not very active. (3) OpenFX also has a mailing list you can join. (4) You can request a free developer license from Autodesk for Flame testing. For DaVinci Resolve OFX debugging, Maxon has published documentation that may also be helpful.

*Source: adobe-plugin-devs · 2026-02-09 · Tags: `ofx`, `flame`, `linux`, `openfx`, `autodesk`, `resolve`, `community` · [View in Q&A](../qa/ofx/)*

---

## What floating license server options exist for OFX plugins (e.g., for Flame)?

RLM (Reprise License Manager) is one option used by companies like Maxon for their plugins. However, the options for floating license servers tend to be pricey. The aescripts licensing system does not support floating licenses. For Flame-specific plugins with Linux floating license requirements, RLM is the most commonly referenced solution in the community.

*Source: adobe-plugin-devs · 2026-01-30 · Tags: `floating-license`, `rlm`, `ofx`, `flame`, `linux`, `licensing` · [View in Q&A](../qa/floating-license/)*

---

## What are best practices for error handling and memory safety in AE SDK plugins?

Recommendations: (1) Wrap AE handles in C++ RAII classes with proper destructors, use smart pointers. (2) Use scope guards (like the pattern from Dr. Dobb's) for cleanup on any exit path. (3) Use std::expected (C++23) or Result types to chain operations safely. (4) Implement an allocation manager that clears allocations when exiting entry functions, since AE handle pointers are not valid between calls. (5) In debug builds, add a cache validator to detect extra memory still allocated at function exit points. (6) Use custom breakpoint assertions that trigger breakpoints when debugger is attached instead of crashing.

*Source: adobe-plugin-devs · 2025-07-27 · Tags: `error-handling`, `memory-safety`, `raii`, `scope-guard`, `smart-pointers`, `best-practice` · [View in Q&A](../qa/error-handling/)*

---

## How do you handle Custom UI mouse-leave events to un-hover buttons?

AE has a known bug where no PF_Event is sent when the mouse leaves the Custom UI area, causing hover states to persist. A workaround is to use a debounce pattern: create a timer callback that triggers after a delay (e.g., 2 seconds). On every mouse-over event, reset the timer. When the timer fires (meaning no mouse-over received recently), trigger an UpdateUI on the main thread to force a Custom UI re-render that resets the hover state.

```cpp
template <typename Func, typename... Args>
std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {
    std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =
        std::make_shared<std::chrono::steady_clock::time_point>();
    return [=](Args... args) {
        *lastCallTime = std::chrono::steady_clock::now();
        std::thread([=]() {
            std::this_thread::sleep_for(delay);
            if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {
                func(args...);
            }
        }).detach();
    };
}
```

*Source: adobe-plugin-devs · 2025-07-25 · Tags: `custom-ui`, `mouse-events`, `hover`, `debounce`, `workaround` · [View in Q&A](../qa/custom-ui/)*

---

## What is PF_REGISTER_EFFECT and will it replace PiPL?

AE's internal plugins now use two entry points: EffectMainExtra and PluginDataEntryFunction, defined via PF_REGISTER_EFFECT / PF_Register_effect_ext2. This appears in SDK samples like SDK_Invert_ProcAmp. While it seems like it could eventually replace PiPL, as of now PiPL is still required for AE - attempting to build without a PiPL/rsrc still results in AE not finding the effect. Premiere Pro no longer requires PiPL and uses this newer system. No official announcement about PiPL deprecation has been made.

*Source: adobe-plugin-devs · 2025-06-05 · Tags: `pipl`, `pf-register-effect`, `plugin-entry-point`, `future`, `premiere` · [View in Q&A](../qa/pipl/)*

---

## How do you set per-host out_flags when supporting both AE and Premiere?

PiPL doesn't support per-host flags. Instead, set different flags in GlobalSetup by checking in_data->appl_id. For example, to use PF_OutFlag_NON_PARAM_VARY only in AE: check if(in_data->appl_id != 'PrMr') before setting the flag. Note: PiPLs are no longer used by Premiere Pro, so you can write PiPL for AE only. Premiere uses PluginDataEntryFunction / PF_REGISTER_EFFECT instead.

```cpp
if (in_data->appl_id != 'PrMr') {
    out_data->out_flags = AE_OUT_FLAGS;
} else {
    out_data->out_flags = PR_OUT_FLAGS;
}
```

*Source: adobe-plugin-devs · 2025-06-04 · Tags: `pipl`, `out-flags`, `premiere`, `per-host`, `global-setup` · [View in Q&A](../qa/pipl/)*

---

## How does GLator (deprecated OpenGL sample) have memory leaks?

The GLator sample wraps GL rendering in try/catch. The suites.IterateFloatSuite1()->iterate call can throw PF_Interrupt_CANCEL in the download texture function, causing bufferH to never be deallocated. This leaks an entire output buffer of 32bpc pixels per frame from CPU, plus GPU FBO resources. Using scope guards (C++ RAII cleanup on scope exit) prevents this pattern - they handle cleanup regardless of how the scope is exited (exception, return, etc.).

*Source: adobe-plugin-devs · 2025-04-18 · Tags: `glator`, `opengl`, `memory-leak`, `exception-handling`, `scope-guard` · [View in Q&A](../qa/glator/)*

---

## What caused plugins to fail to load in AE 25.2 on macOS?

The issue was related to code signature validation. Adobe built AE 25.2 with a newer Xcode that enforces stricter code signature checks. Multiple root causes were identified: (1) Unsigned plugins now fail to load on Apple Silicon. (2) Installers that don't properly clean up old files can break code signatures - if v1 had file X and v2 removes it, but the installer leaves it, the signature breaks. (3) Some plugins had 0-byte .plugin files after the AE upgrade. Solution: Ensure proper code signing, notarization, and clean installation (remove all old files before installing new version).

*Source: adobe-plugin-devs · 2025-04-08 · Tags: `code-signing`, `macos`, `ae-25-2`, `notarization`, `installer`, `apple-silicon` · [View in Q&A](../qa/code-signing/)*

---

## How do you share state across multiple effect plugins in the same AE session?

Options: (1) Use a companion AEGP plugin that communicates with all your effect plugins. (2) Use dlopen/symbol loading: define a global variable in one plugin and have other plugins dynamically load a function from it to get/set the shared state. (3) Use PlugPlug DLL for IPC communication. For state that should reset on next AE launch (not persist to prefs), the dlopen approach or AEGP companion are preferred.

*Source: adobe-plugin-devs · 2025-03-30 · Tags: `inter-plugin-communication`, `global-state`, `dlopen`, `aegp`, `shared-data` · [View in Q&A](../qa/inter-plugin-communication/)*

---

## How do you get the AE/Premiere version number programmatically?

in_data->version major/minor are not reliably updated between AE versions. Options: (1) Use ExtendScript via AEGP_ExecuteScript with 'app.version' - but may fail with 'cannot run script while modal dialog waiting'. (2) Use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP to get the AE install folder path, then parse the version from the folder name or binary metadata. (3) For plugins with Premiere GPU entry points, PPixSuite's appinfo provides version data (format like '24.3', add 2000 for year).

*Source: adobe-plugin-devs · 2025-03-25 · Tags: `version-detection`, `aegp`, `extendscript`, `premiere`, `compatibility` · [View in Q&A](../qa/version-detection/)*

---

## How do you serialize/capture arb data from arbitrary third-party plugins?

Arb data is serialized via PF_Cmd_Arbitrary_Callback using flatten/unflatten functions, similar to sequence data. The flatten/unflatten versions are not accessible via AEGP functions. For plugins with pure stack-variable arb data (like Curves), you can reinterpret_cast the handle as char* and serialize directly. However, plugins with pointers in their arb structs (like Liquify) cannot be serialized this way because deserialized pointer addresses won't be valid. ExtendScript may offer a path through stream values, but it's at the limit of what's possible.

*Source: adobe-plugin-devs · 2025-02-15 · Tags: `arb-data`, `serialization`, `flatten`, `unflatten`, `aegp` · [View in Q&A](../qa/arb-data/)*

---

## How do you handle backward compatibility when adding new parameters to existing plugins?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on new parameters. This flag tells AE to use the parameter's 'value' field (not 'dephault') when loading projects saved before this parameter existed. For more complex migration scenarios, add a hidden parameter storing a version number, and in sequence data setup, parse that version to reset values accordingly.

*Source: adobe-plugin-devs · 2025-01-16 · Tags: `backward-compatibility`, `params`, `versioning`, `pf-param-flag`, `migration` · [View in Q&A](../qa/backward-compatibility/)*

---

## How does AE's Custom UI / arb params bug in AE 2025 manifest?

In AE 2025 (25.0.1), custom UI and arb params display glitches: one arb param's UI can corrupt other arb UIs, causing flashes or scrambled/duplicated content. The likely culprit is NewImageFromBuffer in DrawBot - every call may be setting pixels of all retained DRAWBOT_ImageRef within the same context. Adobe acknowledged fixing something in arb handling, but it may have introduced this unintended consequence. The issue affects multiple third-party plugins including Trapcode.

*Source: adobe-plugin-devs · 2024-12-10 · Tags: `custom-ui`, `arb-data`, `drawbot`, `ae-2025`, `bug`, `regression` · [View in Q&A](../qa/custom-ui/)*

---

## How do you store a struct in global_data?

Create a pointer to the struct in your global data type, then in GlobalSetup allocate it with new: globaldata->structPointer = new StructName(). Use new/malloc (not smart pointers) since AE does special things with the global_data pointer. Optionally delete it in GlobalSetdown, but it's not strictly necessary since GlobalSetdown only gets called when AE closes and the OS will clean up.

*Source: adobe-plugin-devs · 2024-09-03 · Tags: `global-data`, `memory-management`, `global-setup`, `global-setdown` · [View in Q&A](../qa/global-data/)*

---

## Is checking out the current layer with prior effects applied possible in Premiere?

In AE, you can use checkout_layer_pixels in SmartRender to get the layer with prior effects applied. However, this does not work in Premiere. In Premiere, there is no straightforward way to check out the current layer with prior effects applied - PF_CHECKOUT_PARAM always returns the pre-effects layer. A potential hacky workaround might involve calling Premiere's JS scripting API to render, but this is not a clean solution.

*Source: adobe-plugin-devs · 2024-06-19 · Tags: `premiere`, `checkout-layer`, `prior-effects`, `smart-render`, `limitations` · [View in Q&A](../qa/premiere/)*

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

## What is the XMP Packet and can it be used to store plugin data in AE projects?

XMP Packets can be read/written via ExtendScript to store custom metadata in AE projects. However, limitations include: the data is not part of AE's undo stack, practical size limit around 500KB (though AE itself can bloat XMP to 50MB+), and it's not undoable. AE Viewer tools exist that can strip XMP to reduce file sizes dramatically (e.g., 50MB to 50KB).

*Source: adobe-plugin-devs · 2024-04-09 · Tags: `xmp`, `metadata`, `extendscript`, `project-data`, `storage` · [View in Q&A](../qa/xmp/)*

---

## Why does Media Encoder not render plugins correctly or fail to load AEGP plugins?

Media Encoder does not load AEGP plugins. If your effect plugin relies on a companion AEGP plugin for rendering, it will fail in Media Encoder. Issues may also be related to static global variables or elements defined in GlobalData/GlobalSetup. The AEGP functionality would need to be incorporated directly into the effect plugin, or you'd need a standalone/command-line tool alternative.

*Source: adobe-plugin-devs · 2024-02-27 · Tags: `media-encoder`, `aegp`, `render-issues`, `global-data` · [View in Q&A](../qa/media-encoder/)*

---
