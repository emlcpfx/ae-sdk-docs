# The OpenFX C++ Plugin Model: Factory, Effect, and Processor

> If you ship an effect to DaVinci Resolve or Nuke, you write to the OpenFX (OFX)
> API rather than the After Effects SDK. The OFX C++ Support library
> (`OFX::...`) gives you a three-class skeleton -- a plugin *factory*, an
> *image effect* instance, and an *image processor* -- that maps cleanly onto the
> AE `GlobalSetup` / `ParamsSetup` / `SmartRender` lifecycle. This page documents
> the standard support-library structure that works for any OFX host.

This is the OFX counterpart to the AE plugin lifecycle in
[Plugin Architecture](../architecture/index.md). For the AE-vs-OFX render
differences (buffer model, offset sign, GPU handshake) see
[OFX Rendering Differs From After Effects](ofx-rendering-model.md).

## The three classes

The OFX Support library (`ofxsImageEffect.h`) splits a plugin into three roles:

| Class | Base | Role | AE analogue |
|-------|------|------|-------------|
| Factory | `OFX::PluginFactoryHelper<T>` | Declares the plugin, its contexts, bit depths, clips, and parameters | `GlobalSetup` + `ParamsSetup` |
| Effect | `OFX::ImageEffect` | One live instance: holds clip handles, handles param changes, drives a render | Sequence/instance + `SmartRender` |
| Processor | `OFX::ImageProcessor` | Does the actual per-pixel (or per-tile) work, optionally multi-threaded | the render inner loop / iterate callback |

The host calls the factory once to learn what the plugin is, then creates one
`ImageEffect` instance per applied effect, and the instance spins up a processor
for each frame it renders.

## Factory: describe, describeInContext, createInstance

```cpp
class MyPluginFactory : public OFX::PluginFactoryHelper<MyPluginFactory> {
public:
    MyPluginFactory()
        : OFX::PluginFactoryHelper<MyPluginFactory>(
              "com.example.myplugin",   // unique plugin identifier (reverse-DNS)
              1,                        // major version
              0) {}                     // minor version

    void describe(OFX::ImageEffectDescriptor& desc) override;
    void describeInContext(OFX::ImageEffectDescriptor& desc,
                           OFX::ContextEnum context) override;
    OFX::ImageEffect* createInstance(OfxImageEffectHandle handle,
                                     OFX::ContextEnum context) override;
    // Optional:
    void load() override;     // process-wide one-time setup
    void unload() override;   // process-wide teardown
};
```

### describe -- global capabilities

`describe()` runs once per plugin and declares what the effect can do. It does
**not** define clips or parameters; those depend on the context.

```cpp
void MyPluginFactory::describe(OFX::ImageEffectDescriptor& desc) {
    desc.setLabels("My Plugin", "My Plugin", "My Plugin"); // short, label, long
    desc.setPluginGrouping("Example");                     // host menu category

    // Contexts the plugin supports (see "Contexts" below):
    desc.addSupportedContext(OFX::eContextFilter);
    desc.addSupportedContext(OFX::eContextGeneral);

    // Bit depths the plugin can be handed:
    desc.addSupportedBitDepth(OFX::eBitDepthUByte);
    desc.addSupportedBitDepth(OFX::eBitDepthUShort);
    desc.addSupportedBitDepth(OFX::eBitDepthHalf);
    desc.addSupportedBitDepth(OFX::eBitDepthFloat);

    desc.setSupportsTiles(false);            // request whole-frame images
    desc.setTemporalClipAccess(false);       // no access to other frames
    desc.setHostFrameThreading(false);       // you manage your own threading
    desc.setRenderThreadSafety(OFX::eRenderFullySafe);
    desc.setSupportsMultipleClipPARs(false);
}
```

### describeInContext -- clips and parameters

`describeInContext()` runs once per supported context and is where you define
input/output clips and all parameters. The standard clip names are constants
from the SDK, not literals you invent:

```cpp
void MyPluginFactory::describeInContext(OFX::ImageEffectDescriptor& desc,
                                        OFX::ContextEnum /*context*/) {
    // --- Source clip (required input) -------------------------------------
    OFX::ClipDescriptor* src =
        desc.defineClip(kOfxImageEffectSimpleSourceClipName);
    src->addSupportedComponent(OFX::ePixelComponentRGBA);
    src->setSupportsTiles(false);

    // --- An optional secondary input (e.g. a matte or guide layer) ---------
    OFX::ClipDescriptor* aux = desc.defineClip("Aux");
    aux->addSupportedComponent(OFX::ePixelComponentRGBA);
    aux->setOptional(true);              // host may leave it unconnected
    aux->setSupportsTiles(false);

    // --- Output clip (required) -------------------------------------------
    OFX::ClipDescriptor* dst =
        desc.defineClip(kOfxImageEffectOutputClipName);
    dst->addSupportedComponent(OFX::ePixelComponentRGBA);
    dst->setSupportsTiles(false);

    // --- Parameters -------------------------------------------------------
    OFX::PageParamDescriptor* page = desc.definePageParam("Controls");

    OFX::DoubleParamDescriptor* amount = desc.defineDoubleParam("amount");
    amount->setLabel("Amount");
    amount->setDefault(1.0);
    amount->setRange(0.0, 10.0);
    amount->setDisplayRange(0.0, 2.0);
    page->addChild(*amount);

    OFX::BooleanParamDescriptor* invert = desc.defineBooleanParam("invert");
    invert->setLabel("Invert");
    invert->setDefault(false);
    page->addChild(*invert);

    OFX::ChoiceParamDescriptor* mode = desc.defineChoiceParam("mode");
    mode->setLabel("Mode");
    mode->appendOption("Normal");
    mode->appendOption("Screen");
    mode->appendOption("Multiply");
    mode->setDefault(0);
    page->addChild(*mode);

    // Groups nest parameters under a collapsible header:
    OFX::GroupParamDescriptor* grp = desc.defineGroupParam("advanced");
    grp->setLabel("Advanced");
    OFX::DoubleParamDescriptor* gamma = desc.defineDoubleParam("gamma");
    gamma->setLabel("Gamma");
    gamma->setDefault(1.0);
    gamma->setParent(*grp);
    page->addChild(*grp);
    page->addChild(*gamma);
}
```

Parameter descriptor types parallel the AE param set: `defineDoubleParam`,
`defineInt2DParam`, `defineBooleanParam`, `defineChoiceParam` (popup),
`defineRGBAParam` / `defineRGBParam` (color), `defineDouble2DParam`
(position), `definePushButtonParam`, and `defineGroupParam`. Each parameter is
identified by a string name that is the stable serialization key -- treat
parameter names like AE disk IDs: never rename or repurpose one once shipped, or
saved projects will lose their values. (See the AE equivalent in
[Disk ID stability](../parameters/index.md).)

### createInstance

```cpp
OFX::ImageEffect* MyPluginFactory::createInstance(
        OfxImageEffectHandle handle, OFX::ContextEnum /*context*/) {
    return new MyPluginEffect(handle);
}
```

The host owns the returned pointer and deletes it when the effect is removed.

### Registering the factory

The host discovers your plugin through one well-known entry point:

```cpp
void OFX::Plugin::getPluginIDs(OFX::PluginFactoryArray& ids) {
    static MyPluginFactory factory;
    ids.push_back(&factory);
}
```

## Contexts

OFX hosts apply an effect in a *context* that constrains which clips exist:

| Context | Meaning | Typical host surface |
|---------|---------|----------------------|
| `eContextFilter` | One source clip in, one out | Resolve Edit page, Nuke node |
| `eContextGeneral` | Arbitrary clips and params; most flexible | Resolve Color page |
| `eContextTransition` | Two sources (`SourceFrom` / `SourceTo`) plus a transition param | Cross-dissolve-style effects |
| `eContextGenerator` | No source; produces an image | Pattern/noise generators |

A filter-style effect usually advertises both `eContextFilter` and
`eContextGeneral` so it works on both Resolve pages. Optional clips may not be
present in every context -- handle that defensively (see
[OFX Resolve Pitfalls](ofx-resolve-pitfalls.md)).

## ImageEffect: the live instance

```cpp
class MyPluginEffect : public OFX::ImageEffect {
public:
    explicit MyPluginEffect(OfxImageEffectHandle handle);

    void render(const OFX::RenderArguments& args) override;
    void changedParam(const OFX::InstanceChangedArgs& args,
                      const std::string& name) override;

private:
    void setupAndProcess(MyPluginProcessor& proc,
                         const OFX::RenderArguments& args);

    OFX::Clip* m_dstClip = nullptr;
    OFX::Clip* m_srcClip = nullptr;
    OFX::Clip* m_auxClip = nullptr;   // optional
    OFX::DoubleParam*  m_amount = nullptr;
    OFX::BooleanParam* m_invert = nullptr;
};
```

The constructor fetches the clip and parameter handles defined in
`describeInContext`:

```cpp
MyPluginEffect::MyPluginEffect(OfxImageEffectHandle handle)
    : OFX::ImageEffect(handle) {
    m_dstClip = fetchClip(kOfxImageEffectOutputClipName);
    m_srcClip = fetchClip(kOfxImageEffectSimpleSourceClipName);

    // Optional clips must be fetched defensively -- see the Resolve pitfalls
    // page; fetchClip can throw for an unconnected optional clip in some hosts.
    try { m_auxClip = fetchClip("Aux"); } catch (...) { m_auxClip = nullptr; }

    m_amount = fetchDoubleParam("amount");
    m_invert = fetchBooleanParam("invert");
}
```

### render and setupAndProcess

`render()` is the per-frame entry point. The idiomatic pattern is a thin
`render()` that hands off to a `setupAndProcess()` helper, which fetches images,
reads parameters, configures the processor, and runs it:

```cpp
void MyPluginEffect::render(const OFX::RenderArguments& args) {
    MyPluginProcessor proc(*this);
    setupAndProcess(proc, args);
}

void MyPluginEffect::setupAndProcess(MyPluginProcessor& proc,
                                     const OFX::RenderArguments& args) {
    // RAII: std::unique_ptr releases the OFX image when it goes out of scope.
    std::unique_ptr<OFX::Image> dst(m_dstClip->fetchImage(args.time));
    std::unique_ptr<OFX::Image> src(m_srcClip->fetchImage(args.time));
    if (!dst || !src) { OFX::throwSuiteStatusException(kOfxStatFailed); return; }

    // Bit depth / components come off the fetched image, not a global:
    OFX::BitDepthEnum       depth = dst->getPixelDepth();
    OFX::PixelComponentEnum comps = dst->getPixelComponents();

    // Read current parameter values at this time:
    double amount = m_amount->getValueAtTime(args.time);
    bool   invert = m_invert->getValueAtTime(args.time);

    proc.setDstImg(dst.get());
    proc.setSrcImg(src.get());
    proc.setParams(amount, invert);
    proc.setRenderWindow(args.renderWindow);
    proc.process();      // runs multiThreadProcessImages, possibly threaded
}
```

### changedParam

Parameter-driven UI updates and side effects belong in `changedParam`, **not**
the constructor:

```cpp
void MyPluginEffect::changedParam(const OFX::InstanceChangedArgs& /*args*/,
                                  const std::string& name) {
    if (name == "mode") {
        // e.g. show/hide dependent params here
    }
}
```

Mutating parameter UI (visibility, enabled state) from the constructor is a
classic Resolve failure -- see [OFX Resolve Pitfalls](ofx-resolve-pitfalls.md).

## ImageProcessor: the work

`OFX::ImageProcessor` gives you setup plumbing and a `process()` entry that the
support library may call on multiple threads, partitioning the render window for
you. You override `multiThreadProcessImages`:

```cpp
class MyPluginProcessor : public OFX::ImageProcessor {
public:
    explicit MyPluginProcessor(OFX::ImageEffect& inst)
        : OFX::ImageProcessor(inst) {}

    void setSrcImg(OFX::Image* v) { _srcImg = v; }
    void setParams(double amount, bool invert) {
        _amount = amount; _invert = invert;
    }

    void multiThreadProcessImages(OfxRectI window) override {
        for (int y = window.y1; y < window.y2; ++y) {
            if (_effect.abort()) break;   // honor host cancellation
            for (int x = window.x1; x < window.x2; ++x) {
                // getPixelAddress returns the right type for the bit depth;
                // here shown for float (eBitDepthFloat) RGBA:
                auto* dst = (float*)_dstImg->getPixelAddress(x, y);
                auto* src = (float*)(_srcImg ? _srcImg->getPixelAddress(x, y)
                                             : nullptr);
                if (!src) { dst[0]=dst[1]=dst[2]=dst[3]=0.f; continue; }
                for (int c = 0; c < 3; ++c) {
                    float v = src[c] * (float)_amount;
                    dst[c] = _invert ? 1.f - v : v;
                }
                dst[3] = src[3];          // pass alpha through
            }
        }
    }

private:
    OFX::Image* _srcImg = nullptr;
    double _amount = 1.0;
    bool   _invert = false;
};
```

`_dstImg`, `_effect`, the render window, and `process()` come from the base
class; you only supply the inner loop and any extra inputs.

## Pixel format and bit depth on the OFX side

OFX images are described by a `BitDepthEnum` and a `PixelComponentEnum`,
queried from the fetched image (not from a global):

| `OFX::BitDepthEnum` | Sample type | Notes |
|---------------------|-------------|-------|
| `eBitDepthUByte`    | `unsigned char` | 8-bit integer |
| `eBitDepthUShort`   | `unsigned short` | 16-bit integer |
| `eBitDepthHalf`     | 16-bit float | not all hosts |
| `eBitDepthFloat`    | `float` | the common Resolve path |

Components are usually `ePixelComponentRGBA`. Two facts make OFX simpler than AE
here:

- **OFX is RGBA**, not ARGB. Channel 0 is red, channel 3 is alpha. (AE is ARGB
  on the CPU side -- alpha first; see [ARGB pixel order](../advanced/argb.md).)
  Reusing an AE ARGB read path on an OFX RGBA buffer lands alpha in blue.
- **Bit depth is per-image at render time.** You branch on
  `image->getPixelDepth()` inside `setupAndProcess`/the processor rather than
  detecting a format constant in a setup callback. Advertise every depth you
  support in `describe()`, then handle each one in the processor's pixel loop
  (or by templating the processor on the sample type).

A robust processor either templates on the sample type or switches on the depth
once and casts `getPixelAddress` accordingly. Keep the per-pixel math identical
across depths so 8/16/float stay visually consistent.

## Memory ownership

- Images fetched with `fetchImage()` are owned by you for the call -- wrap them
  in `std::unique_ptr<OFX::Image>` (RAII) or call `releaseImage()` explicitly.
  Never `delete` an OFX image with `delete`.
- Parameter and clip handles fetched in the constructor are owned by the
  `ImageEffect` base and need no manual release.
- The `ImageEffect` instance itself is owned by the host (it deletes what your
  `createInstance` returned).

## Minimal checklist

- [ ] Factory subclasses `OFX::PluginFactoryHelper<T>` with a unique
      reverse-DNS identifier and version
- [ ] `describe()` advertises contexts, bit depths, tile/threading flags
- [ ] `describeInContext()` defines source/output (and any optional) clips and
      every parameter, each with a stable string name
- [ ] `createInstance()` returns a new `ImageEffect` subclass
- [ ] `OFX::Plugin::getPluginIDs` pushes the factory
- [ ] Effect constructor fetches clip + param handles (optional clips
      defensively)
- [ ] `render()` -> `setupAndProcess()` -> processor; read params with
      `getValueAtTime(args.time)`
- [ ] Processor overrides `multiThreadProcessImages`, honors `abort()`, treats
      buffers as RGBA, branches on the fetched image's bit depth

## See also

- [OFX Rendering Differs From After Effects](ofx-rendering-model.md) -- buffer,
  offset, and GPU model differences vs the AE path.
- [OFX Resolve Pitfalls](ofx-resolve-pitfalls.md) -- createInstance failures,
  optional clips, channel order.
- [Building Against the OpenFX SDK](ofx-build.md) -- SDK layout, linking, and
  bundle structure.
- [ARGB pixel order](../advanced/argb.md) -- why AE is ARGB and OFX is RGBA.

*Tags: `ofx`, `openfx`, `resolve`, `nuke`, `plugin-structure`, `image-effect`, `cross-host`*
