# OFX Pitfalls in DaVinci Resolve: createInstance Failures, Optional Clips, and Channel Order

> Resolve is the strictest common OFX host, and several of its failure modes are
> silent: the plugin compiles, the bundle installs, but the effect either never
> appears, reports "No parameter is exposed to user," or renders flat/black/blue.
> Every fix below also keeps the plugin working in Nuke, so apply them regardless
> of target host.

This page assumes the standard OFX C++ structure from
[The OpenFX C++ Plugin Model](ofx-plugin-structure.md). For the broader
AE-vs-OFX render model see
[OFX Rendering Differs From After Effects](ofx-rendering-model.md).

## Do not mutate parameter UI in the constructor

The single most common cause of `createInstance` failing in Resolve -- surfaced
as **"No parameter is exposed to user"** or the effect simply not loading -- is
calling parameter-UI mutators from the `ImageEffect` constructor.

Resolve constructs an effect instance for every parallel render as well as for
the visible UI instance. Helpers that walk your parameter set and call
`setIsSecret()` / `setEnabled()` (or any wrapper that does so) will hit
**group-end / page markers and collapsed group headers that have no fetchable
parameter object on a render instance**, and the resulting throw aborts
construction.

```cpp
// WRONG -- in the constructor:
MyEffect::MyEffect(OfxImageEffectHandle h) : OFX::ImageEffect(h) {
    fetchClips();
    updateParamVisibility();   // <-- iterates params, calls setIsSecret() on a
                               //     group marker -> throws -> instance dies
}
```

Move all visibility/enabled logic to `changedParam`, which only runs for the
interactive UI instance after the parameter set is fully realized:

```cpp
// RIGHT:
MyEffect::MyEffect(OfxImageEffectHandle h) : OFX::ImageEffect(h) {
    fetchClips();             // handles only -- no UI mutation
    fetchParams();
}

void MyEffect::changedParam(const OFX::InstanceChangedArgs&,
                            const std::string& name) {
    updateParamVisibility();  // safe here
}
```

If you also want the UI in the correct state the first time the panel opens, do
it from the host's "create UI" hook if available, or trigger one pass from a
known-safe param change -- never from the constructor.

## Wrap fetchClip for optional clips in try/catch

`fetchClip()` on an **optional** clip can throw in `eContextFilter` -- the clip
was declared optional in `describeInContext`, but in the filter context Resolve
may not have registered a fetchable handle for it. An unguarded throw in the
constructor kills the instance.

```cpp
// Required clips: fetch directly.
m_dst = fetchClip(kOfxImageEffectOutputClipName);
m_src = fetchClip(kOfxImageEffectSimpleSourceClipName);

// Optional clips: always guard.
try   { m_aux = fetchClip("Aux"); }
catch (...) { m_aux = nullptr; }
```

Before *using* an optional clip at render time, check both the pointer and the
connection state:

```cpp
std::unique_ptr<OFX::Image> aux;
if (m_aux && m_aux->isConnected())
    aux.reset(m_aux->fetchImage(args.time));
// ... then fall back to a default if aux is null ...
```

## Never throw from the constructor

`OFX::throwSuiteStatusException` (and any other exception) escaping the
`ImageEffect` constructor causes Resolve to mark the plugin **unavailable** --
sometimes permanently until the host restarts, displayed as "OFX Plugin [id] is
not available." Engine/GPU init failures are common at construction time, so
treat construction as best-effort:

```cpp
// WRONG -- a failed init takes the whole plugin down:
MyEffect::MyEffect(OfxImageEffectHandle h) : OFX::ImageEffect(h) {
    getEngine();   // throws if the GPU device can't be created
}

// RIGHT -- log and continue; fail gracefully later at render time:
MyEffect::MyEffect(OfxImageEffectHandle h) : OFX::ImageEffect(h) {
    fetchClips();
    fetchParams();
    try { getEngine(); }
    catch (const std::exception& e) {
        log("engine init failed: %s", e.what());
        // leave engine null; render() detects it and falls back or no-ops
    }
}
```

Throwing from `render()`/`setupAndProcess()` is fine and expected (that's what
`throwSuiteStatusException(kOfxStatFailed)` is for) -- it only fails *that
frame*, not the whole plugin.

## RGBA vs ARGB: alpha landing in the blue channel

OFX images are **RGBA** (channel 0 = red, channel 3 = alpha). If you share a
render core with an After Effects build -- which is **ARGB** on the CPU (alpha
first; see [ARGB pixel order](../advanced/argb.md)) -- and you reuse the AE
channel-order read/write path for OFX, the constant-alpha byte (255 / 1.0) lands
in the **blue** channel and every pixel reads full blue.

Symptom: OFX output is solid blue (or strongly blue-shifted) while the AE build
is correct.

Fix: select the channel layout from a host/format flag, not by assuming one
order everywhere. If your shared core carries an "is ARGB" flag, clear it on the
OFX path and branch the byte-shuffle accordingly:

```cpp
// One shared cast helper, two layouts:
if (layout.is_argb) {
    iterateCast<ArgbPix, CorePix>(...);   // AE: A,R,G,B
} else {
    iterateCast<RgbaPix, CorePix>(...);   // OFX: R,G,B,A
}
```

For GPU paths, do not infer lane order from the format enum -- prove it with a
pure-red pixel probe. See
[Verify GPU Channel Order Empirically](../gpu/pixel-layout-rgba.md).

## Pass a real image where the host expects one -- avoid 1x1 black fallbacks

When an effect has an *optional* secondary input that drives geometry, depth, or
displacement, a common bug is passing `nullptr` to the engine when that input is
absent and the effect should fall back to deriving the data from the source.
Many engines treat a null input as a 1x1 black texture, which collapses the
derived signal (all depth at z=0, all displacement zero) and produces flat or
empty output.

The fix is to pass a **real** image -- usually the source itself -- as the
stand-in input rather than null:

```cpp
// WRONG -- null gives the engine a 1x1 black fallback -> flat result:
engine->render(srcImg, /*auxImg=*/nullptr, dstImg, params);

// RIGHT -- when the aux input is absent, feed the source as the fallback so the
// engine derives the signal (e.g. luma) from real pixels:
OFX::Image* auxInput =
    (useExternalAux && aux && m_aux && m_aux->isConnected())
        ? aux.get()
        : src.get();           // fallback: real image, not null
engine->render(src.get(), auxInput, dstImg, params);
```

This mirrors the AE side, where the source world is passed as the depth/guide
world in the derive-from-luma case. The rule is general: if a host or engine
contract expects a valid image, hand it a valid image -- never a null that
silently degrades to a 1x1 placeholder.

## A few more Resolve-specific traps

- **GPU support is opt-in and unforgiving.** Advertising
  `setSupportsCudaRender`/`setSupportsMetalRender`/`setSupportsOpenCLRender`
  makes the host pass GPU device pointers with **no CPU fallback** if your GPU
  path fails -- output goes black silently. See
  [OFX Rendering Differs From After Effects](ofx-rendering-model.md).
- **bytesPerRow alignment.** Resolve renders thumbnails at odd sizes (e.g.
  208x117); if you upload to a GPU API that requires 256-byte row alignment,
  pad each row before the texture write or you'll get validation errors.
- **Force a rescan after install.** Resolve caches its OFX plugin list; delete
  its plugin cache (or restart) after replacing a bundle, or you'll keep
  testing the old binary.

## Checklist

- [ ] No parameter-UI mutation in the constructor -- only in `changedParam`
- [ ] Every optional `fetchClip` wrapped in try/catch; checked with
      `isConnected()` before use
- [ ] No exceptions escape the constructor -- log init failures, fail later
- [ ] OFX channel layout treated as RGBA; AE ARGB path not reused blindly
- [ ] Absent optional inputs fall back to a real image (source), never null
- [ ] GPU support advertised only after validating it in Resolve specifically

## See also

- [The OpenFX C++ Plugin Model](ofx-plugin-structure.md)
- [OFX Rendering Differs From After Effects](ofx-rendering-model.md)
- [Verify GPU Channel Order Empirically](../gpu/pixel-layout-rgba.md)
- [ARGB pixel order](../advanced/argb.md)

*Tags: `ofx`, `openfx`, `resolve`, `nuke`, `rgba`, `argb`, `createinstance`, `clips`, `cross-host`*
