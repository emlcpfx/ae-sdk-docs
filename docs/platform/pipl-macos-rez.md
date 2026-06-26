# macOS AE Plugins Need the PiPL Rez-Compiled Into the Bundle

> A Windows-style `.rc`-embedded PiPL does not carry over to macOS. On Mac an AE
> `.plugin` must contain a Rez-compiled PiPL resource in `Contents/Resources/`,
> or After Effects silently ignores the plugin -- it builds and links cleanly,
> then never appears in Effects & Presets.

## Symptom

An AE plugin built on macOS does not appear in After Effects at all -- nothing in
Effects & Presets. The `.plugin` bundle's `Contents/Resources/` is empty and the
binary has no PiPL section, yet it compiled and linked without error. Nothing
flags the problem until you try to load it.

## Root cause

After Effects discovers effects via their **PiPL**, read at plugin-scan time. On
Windows the PiPL is produced by the PiPLtool pipeline
(`.r` -> `.rr` -> `.rrc` -> `.rc`) and the resulting resource is linked into the
binary. That entire path is Windows-only. A build set up only for the Windows
embed will generate and validate the PiPL `.r` on macOS but never compile it into
the bundle. No PiPL in the bundle means AE silently skips the plugin.

This is invisible if Mac work has been OFX-only (OFX discovers plugins
differently), which is exactly why it goes unnoticed until the first Mac AE build.

## The fix

On macOS, Rez-compile the configured PiPL `.r` into
`Contents/Resources/<name>.rsrc` as a post-build step (the AE SDK Mac
convention):

```sh
Rez -o <bundle>/Contents/Resources/<name>.rsrc \
    -d SystemSevenOrLater=1 -useDF -script Roman -d __MACH__ -d AE_OS_MAC=1 \
    -arch arm64 -arch x86_64 \
    -i <AE_SDK>/Headers -i <AE_SDK>/Util -i <AE_SDK>/Headers/SP -i <AE_SDK>/Resources \
    <name>_PiPL.r
```

Notes:

- `Rez` is found via `xcrun -f Rez`.
- `-useDF` writes a **data-fork** `.rsrc` (resource forks are deprecated).
- The PiPL's `CodeMacARM64 { "EffectMain" }` (and the x86_64 equivalent) must
  match the binary's exported entry symbol (`_EffectMain`).
- Verify with `strings <name>.rsrc` -- it should show the match name, `PiPL`, and
  the entry-point name.

## How to avoid it

On macOS an AE `.plugin` **must** carry a Rez-compiled PiPL `.rsrc` in
`Contents/Resources/`; the Windows `.rc`-embed path does not apply. An empty
`Contents/Resources/` means the effect won't show up. Whenever an AE plugin
"builds fine but doesn't appear in the menu" on Mac, check for the PiPL `.rsrc`
first. A runtime load test catches this instantly; a clean compile never will.

*Tags: `ae`, `macos`, `pipl`, `build`, `rez`, `bundle`*
