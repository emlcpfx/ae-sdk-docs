# Colorspace

> 4 Q&As · source: AE plugin dev community Discord

### How can I use multithreading for pixel iteration in Premiere, since the AE Iterate8Suite doesn't work correctly?

The AE Iterate8Suite may not work correctly in Premiere and can cause rendering issues (e.g., sequence thumbnail updates but program window doesn't). For 8-bit BGRA, Premiere has its own dedicated iteration suite. For 32-bit float, you need to write your own iteration function. Use std::parallel or equivalent for multithreading. The SDK Noise example demonstrates BGRA and VUYA colorspace handling. The iterate suite should work in 8-bit ARGB (Suitev2 since 2022), but for BGRA you need the special Premiere suite.

*Tags: `bgra`, `colorspace`, `iterate-suite`, `multithreading`, `pixel-iteration`, `premiere`*

---

### How should I handle color spaces in Premiere plugins, and can I avoid dealing with multiple color modes?

You can use basic render in 8/16-bit without handling all color spaces -- it works but is not optimized, and you won't get the 32-bit icon next to the plugin. You can choose to support only BGRA and skip VUYA. In GlobalSetup, you select which colorspaces to support. However, 32-bit float is required for color grading workflows to access values outside the 0-1 range. The SDKNoise example in the AE SDK demonstrates both BGRA and VUYA colorspace handling for Premiere.

*Tags: `32-bit`, `bgra`, `color-grading`, `colorspace`, `global-setup`, `premiere`, `vuya`*

---

### How do you handle colorspace differences between After Effects and Premiere Pro plugins?

After Effects uses ARGB colorspace while Premiere Pro uses BGRA_8u. For 8-bit BGRA, Premiere has a special suite available in the AE SDK (see the sdknoise example which demonstrates BGRA and YUVA colorspace support). For 32-bit float, you need to write your own conversion function. You can choose to support only specific colorspaces in global setup rather than all modes Premiere supports. Basic rendering in 8/16-bit works without optimization, though you won't get the 32-bit icon indicator.

*Tags: `argb`, `bgra`, `colorspace`, `premiere`, `render-loop`*

---

### Where can you find example code for handling Premiere Pro BGRA and YUVA colorspaces?

The sdknoise example in the After Effects SDK demonstrates how to properly handle BGRA and YUVA colorspaces for Premiere Pro plugins. This example shows the correct approach for colorspace handling without relying on problematic iterate suites.

*Tags: `bgra`, `colorspace`, `open-source`, `premiere`, `reference`, `yuva`*

---
