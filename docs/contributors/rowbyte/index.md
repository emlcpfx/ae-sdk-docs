# rowbyte

**12 contributions** to AE SDK community knowledge.

Top topics: `arb-data`, `debugging`, `memory-management`, `cs6`, `arbitrary-data`, `param-setup`, `new-func`, `copy-func`, `memory-leak`, `instruments`

---

## Can After Effects CS6 be installed and run on Windows 11?

Yes, AE CS6 runs fine on Windows 11. If you have activation issues with the license key, you may need to run it in compatibility mode. Windows 10 is a fallback option if Windows 11 doesn't cooperate with activation.

*Source: adobe-plugin-devs · 2026-01-28 · Tags: `cs6`, `windows-11`, `compatibility`, `installation`, `legacy` · [View in Q&A](../qa/cs6/)*

---

## What causes AE error code 1397908844?

Error 1397908844 is a generic exception handler/catch-all in Adobe's suites manager, dating back to CS2/CS3. It commonly appears when double-freeing/releasing a suite pointer. It appears more often in newer AE versions due to MFR with global suite handles shared between threads. Known triggers include: problems with arb params, crashes after exporting RAM previews, and double-releasing suites. The underlying issue is usually in the plugin's suite management code.

*Source: adobe-plugin-devs · 2025-07-05 · Tags: `error-code`, `suite-manager`, `double-free`, `mfr`, `debugging` · [View in Q&A](../qa/error-code/)*

---

## Do PF_LOCK_HANDLE / host_lock_handle functions actually do anything?

According to multiple Adobe engineers, PF_LOCK_HANDLE and host_lock_handle have been no-op dummy functions since AE CS6 (2011) when the codebase moved to 64-bit. Locking/unlocking handles is redundant in 64-bit address space. However, they still provide a safe way to dereference handles (effectively double pointers) and the SDK samples still call them. The DH macro can be used for direct dereferencing instead.

*Source: adobe-plugin-devs · 2025-04-03 · Tags: `memory-management`, `handles`, `lock-unlock`, `64-bit`, `cs6` · [View in Q&A](../qa/memory-management/)*

---

## How do you handle backward compatibility when adding new parameters to existing plugins?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on new parameters. This flag tells AE to use the parameter's 'value' field (not 'dephault') when loading projects saved before this parameter existed. For more complex migration scenarios, add a hidden parameter storing a version number, and in sequence data setup, parse that version to reset values accordingly.

*Source: adobe-plugin-devs · 2025-01-16 · Tags: `backward-compatibility`, `params`, `versioning`, `pf-param-flag`, `migration` · [View in Q&A](../qa/backward-compatibility/)*

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

## Is 5000+ arb dispose calls on AE shutdown normal?

Yes, this is expected behavior. AE's allocators generally don't dispose until available memory is saturated, thread destruction, or app exit. You can test this by setting AE's available memory artificially low (like 2GB) - you'll see many more destruction calls during normal operation. On exit, AE cleans up everything at once.

*Source: adobe-plugin-devs · 2024-12-22 · Tags: `arb-data`, `memory-management`, `dispose`, `shutdown` · [View in Q&A](../qa/arb-data/)*

---

## How does AE's Custom UI / arb params bug in AE 2025 manifest?

In AE 2025 (25.0.1), custom UI and arb params display glitches: one arb param's UI can corrupt other arb UIs, causing flashes or scrambled/duplicated content. The likely culprit is NewImageFromBuffer in DrawBot - every call may be setting pixels of all retained DRAWBOT_ImageRef within the same context. Adobe acknowledged fixing something in arb handling, but it may have introduced this unintended consequence. The issue affects multiple third-party plugins including Trapcode.

*Source: adobe-plugin-devs · 2024-12-10 · Tags: `custom-ui`, `arb-data`, `drawbot`, `ae-2025`, `bug`, `regression` · [View in Q&A](../qa/custom-ui/)*

---

## Is PF_LayerDef::rowbytes always a multiple of 4 in After Effects?

Yes, you can be 100% confident that rowbytes will always be a multiple of sizeof(PF_Pixel8), i.e., 4 bytes. However, 16-byte alignment is NOT always guaranteed. According to Daniel Wilk from Adobe, rowbytes is technically arbitrary and your code should be prepared to deal with any stride. In practice it is often aligned to 16-byte boundaries for SIMD and GPU performance. Important: a buffer might be a sub-reference to another buffer, so you should never write into bytes outside your image reference frame into the rowbytes gutter. If rowbytes were not a multiple of the pixel type's alignment, you would be dereferencing misaligned pointers, which is undefined behavior per the C/C++ spec (C spec 6.3.2.3 paragraph 7).

*Source: adobe-plugin-devs · 2024-09-23 · Tags: `rowbytes`, `pf-layerdef`, `memory-alignment`, `pixel-buffer`, `simd`, `undefined-behavior` · [View in Q&A](../qa/rowbytes/)*

---

## When is PF_Cmd_GLOBAL_SETUP called - during AE loading or when applying the effect?

In After Effects, GlobalSetup is called the first time the user applies the effect to a layer, not during the AE loading screen (during loading, AE only scans PiPLs). In Premiere Pro, GlobalSetup is called when the app is loading. You can show dialogs during GlobalSetup in AE since it happens after the app is fully loaded.

*Source: adobe-plugin-devs · 2024-05-30 · Tags: `global-setup`, `plugin-lifecycle`, `premiere`, `initialization` · [View in Q&A](../qa/global-setup/)*

---

## Why can't macOS Instruments detect PF_EffectWorld memory leaks?

Instruments won't report PF_EffectWorld leaks because AE legitimately thinks you're holding important data in those worlds. AE allocates the worlds deep in its engine, not directly in your plugin code. While memory allocated with new/malloc within the plugin gets reported accurately (because Instruments has more context), AE-managed allocations are invisible to the leak detector. Creating C++ RAII wrappers around AE SDK objects for automatic memory management is recommended.

*Source: adobe-plugin-devs · 2023-11-22 · Tags: `memory-leak`, `instruments`, `mac`, `debugging`, `effect-world`, `raii` · [View in Q&A](../qa/memory-leak/)*

---

## How does PF_Arbitrary_NEW_FUNC work, and when is it called?

PF_Arbitrary_NEW_FUNC is never called in post-CS6 AE. The way arb data works now: you allocate a default arb handle during ParamSetup. For every new instance, AE sends PF_Arbitrary_COPY_FUNC with your default arb handle. Your arb data won't be zero length because it copies from the default you set up. Follow the ColorGrid SDK example which uses both NEW_FUNC and ParamSetup for this.

*Source: adobe-plugin-devs · 2023-08-22 · Tags: `arb-data`, `arbitrary-data`, `param-setup`, `new-func`, `copy-func` · [View in Q&A](../qa/arb-data/)*

---

## Is Lua or Python a better choice for embedding a scripting language inside an After Effects plugin?

Lua is a significantly better fit for embedding inside an AE plugin. It is very easy to compile in and has a small footprint. Python is much clunkier to embed because it requires installing libraries on the user's system (asking permission to install packages), which is intrusive. While there may be cleaner ways to embed Python, Lua is far simpler to integrate. Both require similarly tedious C binding work without some kind of automation or code-generation tooling. dvb metareal shipped 'omino lua' as a script-driven drawing plugin using this approach.

*Source: adobe-plugin-devs · 2023-08-16 · Tags: `lua`, `python`, `scripting`, `embedding`, `c-bindings`, `plugin-architecture` · [View in Q&A](../qa/lua/)*

---
