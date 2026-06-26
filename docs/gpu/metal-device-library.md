# Metal Plugins Must Load Their OWN Bundle's Library -- Never newDefaultLibrary

> On macOS, `[device newDefaultLibrary]` and `[NSBundle mainBundle]` resolve to
> the **host application** (After Effects / a host OFX app), not your plugin.
> They will never find your kernels, and the failure is silent.

## Symptom

A Metal GPU render produces a **black frame with glitch/garbage artifacts** on
macOS. The CPU (software) path works on Mac, and other GPU backends (e.g. CUDA on
Windows) work -- only the macOS Metal path is broken. Nothing is logged; it fails
silently and looks like a render bug rather than a load failure.

## Root cause

Pipeline setup tried to obtain the kernel library with `newDefaultLibrary`
*first*:

```objc
library = [device newDefaultLibrary];          // WRONG for a plugin
if (!library) { /* fall back to compiling our .metal source */ }
```

`newDefaultLibrary` loads `default.metallib` from `[NSBundle mainBundle]`, which
for a *loaded plugin* is the **host application**, not the plugin. The host
(After Effects, Resolve, etc.) is itself a Metal app, so this returns a non-`nil`
library -- the host's. Because it was non-`nil`, the fallback that would compile
your own `.metal` source never ran. Then every
`[library newFunctionWithName:@"YourKernel"]` returns `nil` (the host's library
has none of your kernels), so all `MTLComputePipelineState`s are `nil`, pipeline
setup returns failure, and the render returns early -- leaving the output buffer
holding **uninitialized GPU memory** (the black-with-glitch symptom).

Two compounding factors make this hard to diagnose:

- The failure path logged nothing, so it looked like a render bug.
- The build copied the `.metal` source into one bundle layout but not another
  (e.g. into the AE plugin bundle but not an OFX bundle), so even the runtime
  source-compile fallback had no source to find under that host.

## The fix

1. **Resolve your OWN bundle**, not the host's. Use `dladdr()` on a function
   inside the plugin binary to get the plugin's path, then read from its
   `Contents/Resources`. This works for any host because it keys off the plugin
   binary, not the running app:

   ```objc
   Dl_info info;
   dladdr((const void*)&FindPluginResource, &info);
   // info.dli_fname = .../<YourPlugin>/Contents/MacOS/<binary>
   // walk up to Contents/Resources
   ```

   If you have an Objective-C class in the plugin,
   `[NSBundle bundleForClass:[YourClass class]]` is the equivalent.

2. **Load your library from there.** Prefer a build-time
   `YourKernel.metallib` via `newLibraryWithURL:` if present; otherwise compile
   `YourKernel.metal` with `newLibraryWithSource:` (the OS Metal runtime
   compiler, which ships in the OS and needs no Xcode Metal toolchain on the
   build machine).

3. **Never call `newDefaultLibrary`** in a plugin.

4. **Log on every failure path.** A `nil` `MTLLibrary` silently no-ops the whole
   GPU render to a black frame; surface the error (`NSLog` / host log) so a
   load/compile failure is visible.

5. **Ship the resources to every bundle layout** you build. Copy `.metal`
   sources (or the `.metallib`) into each bundle's `Contents/Resources`.

## How to avoid it

In **any** macOS plugin, `[NSBundle mainBundle]` and
`[device newDefaultLibrary]` resolve to the host app -- they cannot find your
kernels. Always resolve your own bundle (`dladdr()` or
`[NSBundle bundleForClass:]`) and load your library/resources from there. Prefer
a build-time `.metallib` so MSL syntax errors are caught at build time, with
runtime `newLibraryWithSource:` as the portable fallback. And always make a load
or compile failure *visible* -- a clean compile does not catch this; only a
runtime render does.

## See also

- [Metal Compute Shaders](metal-compute.md) -- full device-setup, dispatch, and
  CMake patterns for loading kernels from the plugin bundle.

*Tags: `metal`, `gpu`, `macos`, `plugin-bundle`, `library-loading`, `dladdr`*
