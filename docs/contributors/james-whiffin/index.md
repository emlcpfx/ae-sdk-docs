# James Whiffin

**17 contributions** to AE SDK community knowledge.

Top topics: `premiere`, `drawbot`, `custom-ui`, `global-setup`, `aegp`, `ae-2025`, `regression`, `scope-guard`, `resources`, `arb-data`

---

## What are best practices for error handling and memory safety in AE SDK plugins?

Recommendations: (1) Wrap AE handles in C++ RAII classes with proper destructors, use smart pointers. (2) Use scope guards (like the pattern from Dr. Dobb's) for cleanup on any exit path. (3) Use std::expected (C++23) or Result types to chain operations safely. (4) Implement an allocation manager that clears allocations when exiting entry functions, since AE handle pointers are not valid between calls. (5) In debug builds, add a cache validator to detect extra memory still allocated at function exit points. (6) Use custom breakpoint assertions that trigger breakpoints when debugger is attached instead of crashing.

*Source: adobe-plugin-devs · 2025-07-27 · Tags: `error-handling`, `memory-safety`, `raii`, `scope-guard`, `smart-pointers`, `best-practice` · [View in Q&A](../qa/error-handling/)*

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

## How do you serialize/capture arb data from arbitrary third-party plugins?

Arb data is serialized via PF_Cmd_Arbitrary_Callback using flatten/unflatten functions, similar to sequence data. The flatten/unflatten versions are not accessible via AEGP functions. For plugins with pure stack-variable arb data (like Curves), you can reinterpret_cast the handle as char* and serialize directly. However, plugins with pointers in their arb structs (like Liquify) cannot be serialized this way because deserialized pointer addresses won't be valid. ExtendScript may offer a path through stream values, but it's at the limit of what's possible.

*Source: adobe-plugin-devs · 2025-02-15 · Tags: `arb-data`, `serialization`, `flatten`, `unflatten`, `aegp` · [View in Q&A](../qa/arb-data/)*

---

## What are the AE SDK documentation URLs?

The old URL ae-plugin-sdk.aenhancers.com is defunct. The correct current URL for AE plugin SDK documentation is https://ae-plugins.docsforadobe.dev/

*Source: adobe-plugin-devs · 2025-01-15 · Tags: `documentation`, `sdk`, `resources` · [View in Q&A](../qa/documentation/)*

---

## What causes AEGP_StartUndoGroup(null) to crash in AE 2025?

Starting from AE 2025 beta, passing null/NULL to AEGP_StartUndoGroup causes a crash. Use an empty string "" instead. AEGP_StartUndoGroup("") works correctly and behaves as expected (no entry in the undo stack). This regression has occurred before in earlier AE betas but was corrected.

```cpp
// CRASHES in AE 2025:
suites.UtilitySuite5()->AEGP_StartUndoGroup(null);

// WORKS:
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Source: adobe-plugin-devs · 2024-12-29 · Tags: `undo`, `aegp`, `crash`, `ae-2025`, `regression` · [View in Q&A](../qa/undo/)*

---

## How does AE's Custom UI / arb params bug in AE 2025 manifest?

In AE 2025 (25.0.1), custom UI and arb params display glitches: one arb param's UI can corrupt other arb UIs, causing flashes or scrambled/duplicated content. The likely culprit is NewImageFromBuffer in DrawBot - every call may be setting pixels of all retained DRAWBOT_ImageRef within the same context. Adobe acknowledged fixing something in arb handling, but it may have introduced this unintended consequence. The issue affects multiple third-party plugins including Trapcode.

*Source: adobe-plugin-devs · 2024-12-10 · Tags: `custom-ui`, `arb-data`, `drawbot`, `ae-2025`, `bug`, `regression` · [View in Q&A](../qa/custom-ui/)*

---

## Is PF_LayerDef::rowbytes always a multiple of 4 in After Effects?

Yes, you can be 100% confident that rowbytes will always be a multiple of sizeof(PF_Pixel8), i.e., 4 bytes. However, 16-byte alignment is NOT always guaranteed. According to Daniel Wilk from Adobe, rowbytes is technically arbitrary and your code should be prepared to deal with any stride. In practice it is often aligned to 16-byte boundaries for SIMD and GPU performance. Important: a buffer might be a sub-reference to another buffer, so you should never write into bytes outside your image reference frame into the rowbytes gutter. If rowbytes were not a multiple of the pixel type's alignment, you would be dereferencing misaligned pointers, which is undefined behavior per the C/C++ spec (C spec 6.3.2.3 paragraph 7).

*Source: adobe-plugin-devs · 2024-09-23 · Tags: `rowbytes`, `pf-layerdef`, `memory-alignment`, `pixel-buffer`, `simd`, `undefined-behavior` · [View in Q&A](../qa/rowbytes/)*

---

## How do you check if an input frame has changed to avoid unnecessary re-processing?

Use PF_GetCurrentState to detect if an input has changed. In SmartFX, the PreRender/SmartRender pipeline handles this more naturally. You can also use GuidMixInPtr to mix in any data that should trigger re-rendering - if parameters or other state change, mix that into the GUID and AE will know to call SmartRender again. Note: if the layer is continuously rasterized, transforms are applied before your effect receives the input.

*Source: adobe-plugin-devs · 2024-06-15 · Tags: `caching`, `input-change`, `pf-get-current-state`, `guid-mixin`, `optimization` · [View in Q&A](../qa/caching/)*

---

## When is PF_Cmd_GLOBAL_SETUP called - during AE loading or when applying the effect?

In After Effects, GlobalSetup is called the first time the user applies the effect to a layer, not during the AE loading screen (during loading, AE only scans PiPLs). In Premiere Pro, GlobalSetup is called when the app is loading. You can show dialogs during GlobalSetup in AE since it happens after the app is fully loaded.

*Source: adobe-plugin-devs · 2024-05-30 · Tags: `global-setup`, `plugin-lifecycle`, `premiere`, `initialization` · [View in Q&A](../qa/global-setup/)*

---

## How do you check out a frame with prior effects applied (not the raw layer)?

Use checkout_layer_pixels (from PF_SmartRenderCallbacks in SmartRender) instead of PF_CHECKOUT_PARAM. PF_CHECKOUT_PARAM always returns the layer before all effects were applied. checkout_layer_pixels returns the frame with all prior effects applied. You must first checkout the layer in PreRender, then checkout_layer_pixels in SmartRender.

*Source: adobe-plugin-devs · 2024-02-02 · Tags: `smart-render`, `checkout-layer`, `pre-render`, `effect-stack`, `frame-checkout` · [View in Q&A](../qa/smart-render/)*

---

## What causes slow AE loading when debugging from Xcode 15?

After upgrading to macOS 14/Xcode 15, AE loading becomes ~3x slower due to 'dlsym cannot find symbol xSDKExport' messages for MediaCore bundles. This is Xcode trying to load symbols for all loaded bundles. Workarounds: (1) Run Xcode 14 binary directly from Terminal. (2) Open AE standalone first, then attach the debugger after AE is fully loaded. (3) Check if 'load symbols' option is accidentally selected in your IDE settings.

*Source: adobe-plugin-devs · 2023-11-29 · Tags: `xcode`, `debugging`, `performance`, `macos`, `symbol-loading` · [View in Q&A](../qa/xcode/)*

---

## Is there an equivalent to PF_ABORT() in Premiere for cancelling long renders?

PF_ABORT(in_data) does not work in Premiere -- it never sets err to PF_Interrupt_CANCEL even when scrubbing the timeline with frames taking over half a second to render. There may be a realtime flag that hints whether the effect should be treated as realtime or not, but no confirmed workaround for render cancellation in Premiere was identified.

*Source: adobe-plugin-devs · 2023-08-17 · Tags: `premiere`, `pf-abort`, `render-cancel`, `realtime`, `scrubbing` · [View in Q&A](../qa/premiere/)*

---

## How do you pass custom data structs to GPU kernels in AE plugins?

Define the struct in both the kernel code and your C++ header, then use void pointers to transfer the data from AE to the kernel. Pointers work with CUDA. For Metal, the approach may differ. For OpenCL it works with a few caveats.

*Source: adobe-plugin-devs · 2023-07-29 · Tags: `gpu`, `cuda`, `metal`, `opencl`, `kernel`, `custom-struct` · [View in Q&A](../qa/gpu/)*

---

## Why does DrawBot draw colors at ~90% brightness compared to what you specify?

This can be caused by using DrawImage with opacity less than 1.0, which makes the drawing darker while alpha still appears as 1.0 (since AE's UI is behind with solid alpha). It could also be related to AE's UI brightness preferences or the project's color space settings, as AE's drawing suites may compensate for those things.

*Source: adobe-plugin-devs · 2023-06-24 · Tags: `drawbot`, `custom-ui`, `color-accuracy`, `brightness` · [View in Q&A](../qa/drawbot/)*

---

## Why can I only select the first two items in a PF_ADD_POPUPX dropdown with 3 elements in Premiere, and the third snaps back to 2?

This can be caused by opening a Premiere project with the transition already applied (stale cached state). Applying the transition plugin fresh resolves the issue and the dropdowns work normally. Also ensure that each popup choice string has the pipe separator '|' at the end of each line (except the last), and that you have a proper AEFX_CLR_STRUCT(def) before the popup definition.

```cpp
AEFX_CLR_STRUCT(def);

PF_ADD_POPUPX("Color", 3, 2, 
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Source: adobe-plugin-devs · 2022-12-17 · Tags: `premiere`, `popup`, `dropdown`, `parameter`, `transition`, `cache-bug` · [View in Q&A](../qa/premiere/)*

---

## Where should After Effects SDK developers share code snippets and get help?

Stack Overflow with 'After Effects SDK' tags is recommended as a good system for searching and hosting code. The AE SDK is very niche so resources are scarce, but building up a Stack Overflow presence helps the community. The adobe-plugin-devs Discord server also serves as a community resource, though Discord's chat format has limitations for code sharing compared to a forum system.

*Source: adobe-plugin-devs · 2022-02-27 · Tags: `community`, `stack-overflow`, `resources`, `code-sharing` · [View in Q&A](../qa/community/)*

---
