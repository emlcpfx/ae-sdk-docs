# Getting Started

New to AE plugin development? Start here.

## Your First Plugin

1. **Get the SDK** — Download the [After Effects SDK](https://developer.adobe.com/after-effects/) from Adobe
2. **Pick a build system** — [CMake](../platform/cmake.md) is the modern choice, or use the included Visual Studio/Xcode projects
3. **Start from Skeleton** — The SDK includes a `Skeleton` example that's the minimal working plugin
4. **Understand the basics** — Read [AE Plugin Basics](../guides/ae-plugin-basics.md) for how plugins work

## Key Concepts

- **Effects** are the most common plugin type — they process pixels on a layer
- **PiPL** (Plug-In Property List) is a resource that tells AE what your plugin does before loading it
- **SmartFX** is the modern render path — use `PF_Cmd_SMART_PRE_RENDER` and `PF_Cmd_SMART_RENDER`
- **Bit depths**: AE supports 8-bit, 16-bit, and 32-bit float per channel

## Bit Depth Support

| Depth | Type | Range | When to Use |
|-------|------|-------|-------------|
| 8 bpc | `PF_Pixel8` / `A_u_char` | 0–255 | Legacy, basic effects |
| 16 bpc | `PF_Pixel16` / `A_u_short` | 0–32768 | HDR-aware effects |
| 32 bpc | `PF_PixelFloat` | 0.0–1.0+ | Modern, GPU, HDR |

Set `PF_OutFlag_DEEP_COLOR_AWARE` in your PiPL to support 16/32-bit.

---

*28 documents in this section. Use **Search** (top of page) to find what you need.*
