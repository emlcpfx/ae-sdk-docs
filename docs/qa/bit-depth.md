# Q&A: bit-depth

**8 entries** tagged with `bit-depth`.

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

*Contributors: [**fad**](../contributors/fad/), [**rowbyte**](../contributors/rowbyte/), [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: adobe-plugin-devs · 2024-12-26 · Tags: `bit-depth`, `world-flags`, `pf-layer-def`, `undocumented`, `pixel-format`*

---

## What causes artifacts in 16/32-bit mode when a Track Matte is applied?

The root causes are typically: (1) Incorrect bit depth detection causing memory corruption - using wrong pixel sizes for memory operations. (2) Improper stride/rowbytes handling for padded buffers - when a track matte is applied, the layer gets cropped to a different size, changing the relationship between width and rowbytes. Fix bit depth detection to use rowbytes calculation and implement proper stride handling throughout your processing algorithm. The effect works in 8-bit because the rowbytes happen to align, but in 16/32-bit the padding differences cause corruption.

*Contributors: [**Eric CPFX**](../contributors/eric-cpfx/), [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2025-10-15 · Tags: `track-matte`, `bit-depth`, `16-bit`, `32-bit`, `rowbytes`, `memory-corruption`, `artifacts`*

---

## How can I make a plugin for 8-bit, 16-bit, and 32-bit projects share the same internal codebase?

Use C++ templates to create 3 instances of your algorithm with different data types and range constants. This allows you to write the core logic once and instantiate it for PF_Pixel8, PF_Pixel16, and PF_PixelFloat with appropriate range constants for each bit depth.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-11-01 · Tags: `bit-depth`, `templates`, `pixel-format`, `smart-render`, `code-reuse`*

---

## How to avoid noise artifacts when AE automatically converts 16/32-bit project input to 8-bit for an 8-bit-only plugin?

AE's automatic bit-depth conversion can introduce dithering/noise artifacts. The recommended approach is to let your plugin accept 16 and 32 bpc inputs as well and do the conversion to 8-bit yourself internally. This avoids the warning sign next to your plugin and gives users the benefit of having the output back in 16/32 bpc. You could also report the noise issue as a bug to Adobe.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-11-01 · Tags: `bit-depth`, `dithering`, `noise`, `pixel-conversion`, `8-bit`*

---

## What is the minimal code needed for SmartFX PreRender and SmartRender?

In PreRender: checkout the layer with channel_mask set to PF_ChannelMask_ARGB, then union the result rects. In SmartRender: checkout layer pixels and output, check the bit depth via extra->input->bitdepth (8, 16, or 32) and process accordingly. If you don't need to process anything, use PF_COPY(inputP, outputP, NULL, NULL) which is bit-depth independent. Always check in params with ERR2 regardless of error state. Don't forget to set the correct flags in global setup.

```cpp
// PreRender
PF_RenderRequest req = extra->input->output_request;
PF_CheckoutResult in_result;
req.channel_mask = PF_ChannelMask_ARGB;
ERR(extra->cb->checkout_layer(in_data->effect_ref, EFFECT_INPUT, EFFECT_INPUT, &req,
    in_data->current_time, in_data->time_step, in_data->time_scale, &in_result));
UnionLRect(&in_result.result_rect, &extra->output->result_rect);
UnionLRect(&in_result.max_result_rect, &extra->output->max_result_rect);

// SmartRender
ERR(extra->cb->checkout_layer_pixels(in_data->effect_ref, EFFECT_INPUT, &input_worldP));
ERR(extra->cb->checkout_output(in_data->effect_ref, &output_worldP));
switch (extra->input->bitdepth) {
    case 8: /* process 8 bit */ break;
    case 16: /* process 16 bit */ break;
    case 32: /* process 32 bit */ break;
}
ERR2(PF_CHECKIN_PARAM(in_data, &paramWhatever));
```

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2010-01-01 · Tags: `smartfx`, `smart-render`, `pre-render`, `minimal`, `bit-depth`, `boilerplate`*

---

## How do you view the current pixel bit depth in Premiere Pro?

Use the DogEars feature. You can look up the keyboard shortcut online, or edit the debug database to enable it. This feature is only available in Premiere Pro, not in After Effects.

*Contributors: [**Antoine_Autokroma**](../contributors/antoine-autokroma/) · Source: aescripts discord · 2024-10-02 · Tags: `premiere-pro`, `bit-depth`, `dogears`, `debugging`*

---

## What is the optimal way to make an AE effect plugin compatible with all bit depths (8, 16, and 32 bpc)?

The simplest approach is to write wrapper/converter functions to convert to 32bpc and back, then write your processing code once in 32bpc. These converter functions can also use the multi-threaded iteration suites. The alternative is branching with switch statements for each bit depth, but that results in three separate pieces of code that only differ in type names.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2025-05-23 · Tags: `bit-depth`, `8bpc`, `16bpc`, `32bpc`, `pixel-format`, `conversion`*

---

## How can I make a plugin support 8-bit, 16-bit, and 32-bit color depths without duplicating code for each bit depth?

Use C++ templates to create a single implementation that can be instantiated for different data types and range constants. This allows you to share the same core logic while supporting multiple bit depths. You can also manually convert higher bit-depth input to 8-bit within the plugin itself if needed, rather than relying on After Effects' automatic conversion which may introduce unwanted color noise.

*Tags: `templates`, `bit-depth`, `color-processing`, `codebase-reuse`, `sdk`, `c++`*

---
