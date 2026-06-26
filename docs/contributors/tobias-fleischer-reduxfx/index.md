# Tobias Fleischer (reduxFX)

**29 contributions** to AE SDK community knowledge.

Top topics: `pipl`, `macos`, `smartrender`, `premiere-pro`, `force-rerender`, `bit-depth`, `pixel-format`, `aegp`, `regression`, `cs6`

---

## How do you display a progress banner (like 'Computing: X%') that updates during SmartRender in After Effects?

The technique involves using a hidden parameter to trigger re-renders. When the user presses a button, code changes a hidden parameter which triggers AE to call SmartRender again. Each SmartRender call checks the current state and draws the appropriate banner. Adobe's internal 3D Camera Tracker uses PF_Cmd_RESERVED3 (an undocumented idle command selector that AE calls repeatedly) to check background work status and sets PF_OutFlag_FORCE_RERENDER to trigger SmartRender calls. Parameter values can only be changed during PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_EVENT. An alternative is to use AEGP_RegisterIdleHook with hidden parameter changes, though you don't have access to PF_OutData (and thus PF_OutFlag_FORCE_RERENDER) in an idle hook. In Premiere Pro, PF_OutFlag_FORCE_RERENDER doesn't work, so the hidden parameter trick is the recommended approach.

*Source: aescripts discord · 2026-03-05 · Tags: `smartrender`, `progress-bar`, `re-render`, `hidden-parameter`, `force-rerender`, `pf-cmd-reserved3`, `idle-hook` · [View in Q&A](../qa/smartrender/)*

---

## Plugins are not loading via MediaCore folder on macOS Tahoe / AE 2026. What's happening?

Multiple reports have been received about plugins failing to load from the MediaCore folder on macOS Tahoe (or with the AE 2026 upgrade). Plugins only work when placed in the Plug-ins folder. This appears to be a newly emerging issue with the OS/AE update.

*Source: aescripts discord · 2026-03-02 · Tags: `macos-tahoe`, `mediacore`, `plugin-loading`, `ae-2026`, `regression` · [View in Q&A](../qa/macos-tahoe/)*

---

## How do you set up separate debug and release plugin names in Visual Studio for AE plugins?

Use preprocessor conditionals in the PiPL .r file for both the Name and AE_Effect_Match_Name fields. Add _DEBUG to the PiPL custom build command line. You'll likely need different output filenames too. Watch out for rebuilds cleaning the previous output folder -- add a post-build copy step that copies the generated file from the local folder to the MediaCore folder.

```cpp
Name {
    #ifdef _DEBUG
    "plugin_DEBUG"
    #else
    "plugin"
    #endif
},
AE_Effect_Match_Name {
    #ifdef _DEBUG
    "ADBE plugin_DEBUG"
    #else
    "ADBE plugin"
    #endif
},
```

*Source: aescripts discord · 2026-02-25 · Tags: `debug`, `release`, `pipl`, `visual-studio`, `build-configuration`, `match-name` · [View in Q&A](../qa/debug/)*

---

## How does AE_Effect_Version work in the PiPL resource file?

AE_Effect_Version encodes version information using bit shifting. The formula is: (MAJOR_VERSION << 19) + (MINOR_VERSION << 15) + (BUG_VERSION << 11) + BUILD_VERSION. Note that MAJOR_VERSION may not be larger than 8 due to bit width constraints. The full PF_VERSION macro in AE_Effect.h provides more detail with additional fields for stage and build. If you get a PiPL version mismatch error, output the version you have in code for global setup and compare it to the one in the PiPL using a hex editor.

```cpp
#define PF_VERSION(vers, subvers, bugvers, stage, build)    \
    (PFVersionInfo)(    \
         ((((A_u_long)PF_Vers_VERS_HIGH(vers)) & PF_Vers_VERS_HIGH_BITS) << PF_Vers_VERS_HIGH_SHIFT) |   \
        ((((A_u_long)(vers)) & PF_Vers_VERS_BITS) << PF_Vers_VERS_SHIFT) |    \
        ((((A_u_long)(subvers)) & PF_Vers_SUBVERS_BITS)<<PF_Vers_SUBVERS_SHIFT) |\
        ((((A_u_long)(bugvers)) & PF_Vers_BUGFIX_BITS) << PF_Vers_BUGFIX_SHIFT) |\
        ((((A_u_long)(stage)) & PF_Vers_STAGE_BITS) << PF_Vers_STAGE_SHIFT) |    \
        ((((A_u_long)(build)) & PF_Vers_BUILD_BITS) << PF_Vers_BUILD_SHIFT)    \
    )
```

*Source: aescripts discord · 2026-02-23 · Tags: `pipl`, `version`, `ae-effect-version`, `pf-version`, `bit-shifting` · [View in Q&A](../qa/pipl/)*

---

## How do you prevent an AE plugin (.aex) from being detected and loaded in Premiere Pro?

Two main approaches: (1) Place the .aex file only in the AE plugins folder and not in the MediaCore folder. (2) Return an error from the EffectMain callback when the host identifier in in_data is "PrMr". You can check the host in EffectMain (which has access to in_data) rather than PluginDataEntryFunction.

*Source: aescripts discord · 2026-02-19 · Tags: `premiere-pro`, `plugin-loading`, `host-detection`, `mediacore`, `effectmain` · [View in Q&A](../qa/premiere-pro/)*

---

## Do Mac plugins need to be notarized in addition to codesigning for distribution via the aescripts app?

Yes, codesigning alone is not enough. The plugins also need to be notarized by Apple for distribution.

*Source: aescripts discord · 2026-02-16 · Tags: `macos`, `notarization`, `code-signing`, `distribution`, `aescripts` · [View in Q&A](../qa/macos/)*

---

## Where can I find resources and community support for developing OFX plugins for Flame on Linux?

There are several resources: (1) The official OpenFX Slack channel (now under ASWF - Academy Software Foundation) is the recommended place, though access may be restricted to specific email domains like linuxfoundation.org or aswf.io. (2) There is an unofficial OFX Discord server with a dedicated Flame channel, though it's not very active. (3) OpenFX also has a mailing list you can join. (4) You can request a free developer license from Autodesk for Flame testing. For DaVinci Resolve OFX debugging, Maxon has published documentation that may also be helpful.

*Source: adobe-plugin-devs · 2026-02-09 · Tags: `ofx`, `flame`, `linux`, `openfx`, `autodesk`, `resolve`, `community` · [View in Q&A](../qa/ofx/)*

---

## Is there a way to show a progress bar in Premiere Pro for video filter / AE effect rendering?

In the AE SDK, there is a function for reporting render progress back to the host, which results in the standard AE progress bar being displayed/updated. It's unclear whether Premiere supports this. There is also an unofficial way used by some of Adobe's native effects to display custom progress, but the details may not be publicly disclosed.

*Source: aescripts discord · 2026-02-09 · Tags: `progress-bar`, `rendering`, `premiere-pro`, `ui-feedback` · [View in Q&A](../qa/progress-bar/)*

---

## How does PF_Cmd_SEQUENCE_RESETUP work and when is sequence data flattened?

PF_Cmd_SEQUENCE_RESETUP is called either when opening a project with saved (flat) sequence data, or after AE has just asked to flatten sequence data (for a save) and now needs to unflatten it. The in_data->sequence_data pointer will always point to flattened data at this point, and your task is to unflatten it. There is a flag in the input data that tells you if data is flat or not. The documentation phrase 're-create (usually unflatten) sequence data' means that for simple plugins that only use regular parameters and don't need flattening/unflattening, the unflatten step is unnecessary. The input data should always be flat and the output data should always be unflat for this function.

```cpp
// We got here because we're either opening a project w/saved (flat) sequence data,
// or we've just been asked to flatten our sequence data (for a save) and now we're blowing it back up.
```

*Source: aescripts discord · 2026-02-04 · Tags: `sequence-data`, `flatten`, `unflatten`, `sequence-resetup`, `project-save` · [View in Q&A](../qa/sequence-data/)*

---

## What are the parts of the AE PiPL version system that are legacy and how old is some of the AE SDK code?

Some parts of the AE codebase that developers still struggle with, like PiPL data, date back to 1990 -- over 35 years old. Adobe has been reluctant to modernize these legacy systems or listen to developer suggestions. The sequence data handling hasn't changed for 15+ years either.

*Source: aescripts discord · 2026-02-04 · Tags: `pipl`, `legacy`, `sdk-history`, `adobe` · [View in Q&A](../qa/pipl/)*

---

## Does the AE Windows ARM beta require native ARM64 plugins or can x64 plugins run in emulation?

Adobe's AE for ARM is not EC (Emulation Compatible), meaning x64 plugins will NOT run emulated on ARM, unlike Resolve and Nuke which currently work in EC mode. You must rebuild your Windows plugins as native ARM64. The latest VS2022 is required for ARM64 compilation, and all third-party libraries (static or dynamic) must also be available in ARM64 format. The C++ licensing library from aescripts is not yet available for Windows ARM64 but internal builds are working.

*Source: aescripts discord · 2026-01-29 · Tags: `windows-arm`, `arm64`, `emulation`, `native`, `visual-studio-2022` · [View in Q&A](../qa/windows-arm/)*

---

## Can After Effects CS6 be installed and run on Windows 11?

Yes, AE CS6 runs fine on Windows 11. If you have activation issues with the license key, you may need to run it in compatibility mode. Windows 10 is a fallback option if Windows 11 doesn't cooperate with activation.

*Source: adobe-plugin-devs · 2026-01-28 · Tags: `cs6`, `windows-11`, `compatibility`, `installation`, `legacy` · [View in Q&A](../qa/cs6/)*

---

## Can you disable the crosshair/visual representation of point parameters in the AE comp window?

No, the crosshairs are the visual representation of point parameters and cannot be disabled. As a workaround, you could use a different parameter type like an int or float param pair for x and y values instead of a point parameter.

*Source: aescripts discord · 2025-12-29 · Tags: `point-parameters`, `crosshair`, `comp-window`, `ui`, `workaround` · [View in Q&A](../qa/point-parameters/)*

---

## Can the same .aex plugin work in both After Effects and Premiere Pro?

Yes, but it requires correct coding. Premiere does many things differently than AE and not all features are supported in both hosts. The AE SDK documentation has a dedicated section for Premiere Pro compatibility (https://ae-plugins.docsforadobe.dev/). Notably, Premiere does not support SmartRender, which is a significant difference. There are also long-standing bugs in Premiere's implementation of AE plugin APIs (present since at least 2006) that are regularly reported to but largely ignored by Adobe.

*Source: aescripts discord · 2025-12-21 · Tags: `premiere-pro`, `aex`, `cross-host`, `compatibility`, `smartrender` · [View in Q&A](../qa/premiere-pro/)*

---

## Why might a plugin fail to load on some Macs with 'couldn't find main entry point' error, even without missing dependencies?

While 'couldn't find main entry point' usually means missing dependencies, it can also be caused by library collisions. For example, if your plugin links against curl, check whether you're providing your own dylib or linking against Apple's system version. On macOS, curl is normally relatively unproblematic -- you can usually safely use the lib provided by Apple. Also check whether the issue appears only on Silicon or Intel Macs.

*Source: aescripts discord · 2025-12-12 · Tags: `macos`, `loading-error`, `entry-point`, `library-collision`, `curl`, `dylib` · [View in Q&A](../qa/macos/)*

---

## How can you make a watermark harder to remove from a plugin's trial output?

Several techniques: (1) Make the pixel colors of the watermark vary randomly to prevent color keying. (2) Add alpha blending with the source layer. (3) Make the cross/border wider or vary the shape. (4) Add text/noise. (5) Use a few-color gradient instead of random noise for a prettier but still hard-to-key result. The key is randomizing per-pixel colors with rand() to prevent simple removal via color keying.

*Source: aescripts discord · 2025-10-04 · Tags: `watermark`, `licensing`, `trial`, `copy-protection` · [View in Q&A](../qa/watermark/)*

---

## Is there a way to create a universal .aex binary for both x64 and Windows ARM, similar to macOS universal binaries?

No, there is no universal binary format for Windows like macOS has. You need to compile separate binaries for each architecture. However, you can cross-compile for both x64 and ARM64 platforms on one system using the latest Visual Studio 2022 (after installing the respective tool sets). Note that AE on ARM is not EC (Emulation Compatible), so unlike Resolve and Nuke which run x64 code emulated on ARM, AE requires native ARM64 plugins. All third-party libraries you link to must also be available in ARM64 format.

*Source: aescripts discord · 2025-09-25 · Tags: `windows-arm`, `arm64`, `cross-compile`, `visual-studio`, `architecture` · [View in Q&A](../qa/windows-arm/)*

---

## Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Source: aescripts discord · 2025-07-10 · Tags: `smartfx`, `prerender`, `smartrender`, `code-organization`, `cpp`, `dll` · [View in Q&A](../qa/smartfx/)*

---

## What causes AE error code 1397908844?

Error 1397908844 is a generic exception handler/catch-all in Adobe's suites manager, dating back to CS2/CS3. It commonly appears when double-freeing/releasing a suite pointer. It appears more often in newer AE versions due to MFR with global suite handles shared between threads. Known triggers include: problems with arb params, crashes after exporting RAM previews, and double-releasing suites. The underlying issue is usually in the plugin's suite management code.

*Source: adobe-plugin-devs · 2025-07-05 · Tags: `error-code`, `suite-manager`, `double-free`, `mfr`, `debugging` · [View in Q&A](../qa/error-code/)*

---

## How do you write cross-host plugin code that works for AE, Premiere, OFX, and other hosts?

Write host/format-independent processing code and create thin adapter/wrapper layers per host. This approach has been used successfully for AE/PPro/OFX/Nuke/Frei0r formats. The OFX SDK (partially based on AE SDK) is cleaner in some areas - for example, OFX supports thousands of plugins in a single binary (useful for suites like GMIC with ~2000 plugins), while AE requires separate PiPL/loader for each. When porting, the processing core stays the same and only the parameter declaration and pixel access patterns need per-host adaptation.

*Source: adobe-plugin-devs · 2025-06-06 · Tags: `cross-platform`, `ofx`, `architecture`, `code-reuse`, `plugin-framework` · [View in Q&A](../qa/cross-platform/)*

---

## What is PF_REGISTER_EFFECT and will it replace PiPL?

AE's internal plugins now use two entry points: EffectMainExtra and PluginDataEntryFunction, defined via PF_REGISTER_EFFECT / PF_Register_effect_ext2. This appears in SDK samples like SDK_Invert_ProcAmp. While it seems like it could eventually replace PiPL, as of now PiPL is still required for AE - attempting to build without a PiPL/rsrc still results in AE not finding the effect. Premiere Pro no longer requires PiPL and uses this newer system. No official announcement about PiPL deprecation has been made.

*Source: adobe-plugin-devs · 2025-06-05 · Tags: `pipl`, `pf-register-effect`, `plugin-entry-point`, `future`, `premiere` · [View in Q&A](../qa/pipl/)*

---

## What is the optimal way to make an AE effect plugin compatible with all bit depths (8, 16, and 32 bpc)?

The simplest approach is to write wrapper/converter functions to convert to 32bpc and back, then write your processing code once in 32bpc. These converter functions can also use the multi-threaded iteration suites. The alternative is branching with switch statements for each bit depth, but that results in three separate pieces of code that only differ in type names.

*Source: aescripts discord · 2025-05-23 · Tags: `bit-depth`, `8bpc`, `16bpc`, `32bpc`, `pixel-format`, `conversion` · [View in Q&A](../qa/bit-depth/)*

---

## Do PF_LOCK_HANDLE / host_lock_handle functions actually do anything?

According to multiple Adobe engineers, PF_LOCK_HANDLE and host_lock_handle have been no-op dummy functions since AE CS6 (2011) when the codebase moved to 64-bit. Locking/unlocking handles is redundant in 64-bit address space. However, they still provide a safe way to dereference handles (effectively double pointers) and the SDK samples still call them. The DH macro can be used for direct dereferencing instead.

*Source: adobe-plugin-devs · 2025-04-03 · Tags: `memory-management`, `handles`, `lock-unlock`, `64-bit`, `cs6` · [View in Q&A](../qa/memory-management/)*

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

## What suites should you avoid calling from Death Hook?

You shouldn't rely on any suite being available in the Death Hook. Attempting to use suites like UtilitySuite's ReportInfo from Death Hook can cause unhandled exceptions. By that point in the shutdown process, many suites may already be torn down.

*Source: adobe-plugin-devs · 2024-03-30 · Tags: `death-hook`, `suites`, `shutdown`, `aegp` · [View in Q&A](../qa/death-hook/)*

---

## How do you use GuidMixInPtr to force re-rendering of cached frames?

Set the flag PF_OutFlag2_I_MIX_GUID_DEPENDENCIES, then in PreRender call extraP->cb->GuidMixInPtr with changed data. When the mixed-in data changes, AE invalidates the cache for that frame. Check for null pointers on extraP, cb, and GuidMixInPtr. Also check against 0xabababababababab (uninitialized memory pattern). You can mix in timestamps, parameter values, or any data that should trigger re-rendering.

```cpp
static PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data, PF_PreRenderExtra* extraP) {
    if (in_data->version.major >= PF_AE130_PLUG_IN_VERSION &&
        extraP && extraP->cb && extraP->cb->GuidMixInPtr &&
        (unsigned long)(extraP->cb->GuidMixInPtr) != 0xabababababababab) {
        int r = time(0) + in_data->current_time;
        extraP->cb->GuidMixInPtr(in_data->effect_ref, sizeof(r), reinterpret_cast<void*>(&r));
    }
    // ...
}
```

*Source: adobe-plugin-devs · 2024-03-28 · Tags: `guid-mixin`, `cache-invalidation`, `pre-render`, `force-rerender` · [View in Q&A](../qa/guid-mixin/)*

---

## How do you keep PiPL out_flags in sync with GlobalSetup flags?

Define the flags as macros in your plugin header file (e.g., #define FX_OUT_FLAGS (...)) and use those same macros in both your PiPL .r file and GlobalSetup code. The PiPL should include the header with all bit flags defined so it auto-updates when compiling. However, note that this approach may fail with higher bits/newest flags due to overflow in Adobe's PiPL compiler. An online tool for calculating flag bitmasks is available at https://reduxfx.com/aeoutflags.htm.

```cpp
#define FX_OUT_FLAGS (PF_OutFlag_FORCE_RERENDER + \
                       PF_OutFlag_USE_OUTPUT_EXTENT + \
                       PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                       PF_OutFlag_DEEP_COLOR_AWARE + \
                       PF_OutFlag_CUSTOM_UI + \
                       PF_OutFlag_I_DO_DIALOG)

// In PiPL:
AE_Effect_Global_OutFlags { FX_OUT_FLAGS },
AE_Effect_Global_OutFlags_2 { FX_OUT_FLAGS2 },
```

*Source: adobe-plugin-devs · 2024-03-27 · Tags: `pipl`, `out-flags`, `build-system`, `macros` · [View in Q&A](../qa/pipl/)*

---

## Should .aex plugins be code signed on Windows?

.aex plugins are regular DLLs (just renamed), so they can be codesigned. Microsoft currently allows unsigned DLLs to be loaded from a signed process, but they have hinted this might change in the future (was supposed to happen with Win11). On macOS, a plugin is a bundle (renamed folder), and you don't need to sign the actual plugin binary inside it as long as the bundle itself is signed.

*Source: aescripts discord · 2023-12-13 · Tags: `code-signing`, `aex`, `dll`, `windows`, `macos`, `security` · [View in Q&A](../qa/code-signing/)*

---
