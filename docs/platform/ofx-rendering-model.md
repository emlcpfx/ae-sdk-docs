# OFX Rendering Differs From After Effects: Buffer, Offset, and GPU Model

> If you build one effect for both After Effects and an OFX host (DaVinci Resolve,
> Nuke), do not assume the OFX render path mirrors the AE one. The intermediate
> buffer, the offset sign, the GPU device handshake, and even bundle discovery all
> differ. Copying the AE patterns into OFX is a reliable way to get black output.

## Symptom

OFX plugins render black or transparent in DaVinci Resolve while the AE build of
the same effect works. The issues cascade -- fixing one reveals the next -- and
the fastest way to diagnose them is to compare against a known-working OFX plugin.

## Differences and fixes

### 1. No intermediate buffer needed for 32-bit float

AE works in ARGB channel order internally, so an AE render path often needs an
intermediate buffer to convert to/from the plugin's working layout. OFX images
for 32-bit float are RGBA natively. Write directly to the destination image's
pixel data (`dst_img_->getPixelData()`) rather than staging through an
intermediate copy -- the copy path is an extra place for bugs (and was the source
of black output here). Direct write is the proven OFX pattern.

### 2. `output()` must not re-clear the buffer

If your render helper clears the destination to transparent black each time it is
called, a *second* call (for example a watermark or overlay pass after the main
render) wipes the render and leaves only what was written after the clear. Clear
once -- track first-call with a flag -- not on every call.

### 3. The OFX offset model is the opposite of AE

The expanded buffer is on opposite sides in the two hosts:

| Host | Which buffer is larger | Offset formula |
|------|------------------------|----------------|
| AE   | Output (larger)        | `(output_w - input_w) / 2` |
| OFX  | Input (larger, via Region of Interest) | `dst_bounds.x1 - src_bounds.x1` |

Using the AE formula in OFX produces *negative* offsets (because input > output
there). Compute the offset from the bounds origins instead.

### 4. OFX GPU requires per-host validation

OFX hosts hand you only a command queue/stream, not the full device or context:

- **Metal** -- no `MTLDevice` directly; extract it from `queue.device`.
- **OpenCL** -- no `cl_context`/`cl_device_id`; extract via
  `clGetCommandQueueInfo`.
- **CUDA** -- works with just the stream.

Advertising GPU support (e.g. `setSupportsCudaRender(true)`) makes the host send
GPU device pointers instead of CPU pointers. If the GPU path then fails silently,
the output is black with **no CPU fallback**. Some shipping OFX plugins disable
GPU entirely with a note that GPU flags crash without a complete implementation.
Enable OFX GPU only after testing in each target host.

### 5. `Info.plist` is required in the OFX bundle

DaVinci Resolve requires `Contents/Info.plist` inside the `.ofx.bundle` to
discover plugins -- without it the plugin silently does not load. Nuke is more
lenient. Generate the `Info.plist` as part of the build for every OFX bundle.

## How to avoid it

- Don't copy the AE render pattern blindly into OFX -- the buffer model, offset
  sign, and GPU handshake all differ.
- When an OFX render goes black, compare against a known-working OFX plugin
  rather than your AE path.
- Test in Resolve specifically -- it is stricter than Nuke about bundle metadata
  and GPU capabilities.
- Never advertise OFX GPU support without validating it in each host you ship to.

*Tags: `ofx`, `resolve`, `nuke`, `rendering`, `gpu`, `cross-host`, `bundle`*
