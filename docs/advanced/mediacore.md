# Mediacore

> 3 Q&As · sources: AE plugin dev community Discord; PluginPort field debugging

### How do you prevent an AE plugin (.aex) from being detected and loaded in Premiere Pro?

Two main approaches: (1) Place the .aex file only in the AE plugins folder and not in the MediaCore folder. (2) Return an error from the EffectMain callback when the host identifier in in_data is "PrMr". You can check the host in EffectMain (which has access to in_data) rather than PluginDataEntryFunction.

*Tags: `effectmain`, `host-detection`, `mediacore`, `plugin-loading`, `premiere-pro`*

---

### Plugins are not loading via MediaCore folder on macOS Tahoe / AE 2026. What's happening?

Multiple reports have been received about plugins failing to load from the MediaCore folder on macOS Tahoe (or with the AE 2026 upgrade). Plugins only work when placed in the Plug-ins folder. This appears to be a newly emerging issue with the OS/AE update.

**Confirmed cause + fix in at least one case:** the `.plugin` bundle was not *typed as an After Effects effect*. AE's own `Plug-ins/` folder is lenient — it Rez-scans every bundle's PiPL directly and loads it regardless of package type — so a mis-typed bundle still loads there. The shared **MediaCore** host instead classifies bundles by their package type / signature and **silently skips** anything that isn't an effect (`eFKT` / `FXTC`). Recent macOS / AE appear to enforce this more strictly than before, which is why bundles that "always worked" suddenly stopped loading from MediaCore. See the next entry for the exact bundle requirements.

*Tags: `ae-2026`, `macos-tahoe`, `mediacore`, `plugin-loading`, `regression`, `bundle-type`*

---

### macOS: my effect loads from the Plug-ins folder but is invisible in the MediaCore folder. How do I fix it?

The `.plugin` bundle must be **typed as an AE effect**. MediaCore reads the bundle's package type and signature (from `Info.plist` and `Contents/PkgInfo`) to decide what kind of plug-in it is; a generic `BNDL` bundle is not recognized as an effect and is dropped without any error or log. AE's own `Plug-ins/` folder doesn't care (it reads the PiPL directly), which is exactly why the symptom is "works in Plug-ins, missing in MediaCore."

Set, to match the AE SDK reference effect bundles (e.g. `SDK_Noise.plugin`):

```xml
<!-- Contents/Info.plist -->
<key>CFBundlePackageType</key><string>eFKT</string>
<key>CFBundleSignature</key><string>FXTC</string>
```
```
# Contents/PkgInfo  — exactly 8 raw bytes, NO trailing newline
eFKTFXTC
```

`eFKT` is the AE effect package type; `FXTC` is the Adobe effect/transcode signature; `PkgInfo` is just those two 4-char codes concatenated. Verify a built bundle:

```bash
plutil -p MyPlugin.plugin/Contents/Info.plist | grep -E "PackageType|Signature"
xxd     MyPlugin.plugin/Contents/PkgInfo        # expect: eFKTFXTC
```

Reinstall the rebuilt bundle to `/Library/Application Support/Adobe/Common/Plug-ins/<version>/MediaCore/` (replacing any stale copy) and it will appear in AE, Premiere, and Media Encoder. (The PiPL must also be Rez-compiled into `Contents/Resources/<name>.rsrc` — a separate requirement for the effect to be discovered at all on macOS.)

*Tags: `macos`, `mediacore`, `plugin-loading`, `bundle-type`, `efkt`, `fxtc`, `pkginfo`, `info-plist`, `premiere-pro`*

---
