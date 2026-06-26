# Q&A: mac

**7 entries** tagged with `mac`.

---

## How do you build AE plugins for Apple Silicon (ARM64)?

You can build on an M1 Mac (even a cheap Mac Mini works as a build machine). It's not strictly required to build on M1, but it helps for testing. The key thing many people miss is that you must add ARM64 to your PiPL resource file - just building as Universal Binary in Xcode is not enough. Without the ARM64 entry in the PiPL, AE will show the 'not yet compatible' warning.

*Contributors: [**gabgren**](../contributors/gabgren/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/), [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2023-08-18 · Tags: `apple-silicon`, `arm64`, `pipl`, `xcode`, `mac`, `universal-binary`*

---

## What causes the 'Invalid Filter 25::3' error on macOS?

This error is typically caused by the Deployment Target being set too high in Xcode. For example, setting it to 12.3 will cause this error on older macOS versions. Try lowering it to support older versions. For Intel builds, supporting back to macOS 10.10 is common.

*Contributors: [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2023-08-16 · Tags: `mac`, `xcode`, `deployment-target`, `invalid-filter`, `compatibility`*

---

## Why can't macOS Instruments detect PF_EffectWorld memory leaks?

Instruments won't report PF_EffectWorld leaks because AE legitimately thinks you're holding important data in those worlds. AE allocates the worlds deep in its engine, not directly in your plugin code. While memory allocated with new/malloc within the plugin gets reported accurately (because Instruments has more context), AE-managed allocations are invisible to the leak detector. Creating C++ RAII wrappers around AE SDK objects for automatic memory management is recommended.

*Contributors: [**rowbyte**](../contributors/rowbyte/), [**tlafo**](../contributors/tlafo/), [**fad**](../contributors/fad/) · Source: adobe-plugin-devs · 2023-11-22 · Tags: `memory-leak`, `instruments`, `mac`, `debugging`, `effect-world`, `raii`*

---

## How do you run an older version of Xcode on a newer macOS?

Download older Xcode versions from https://developer.apple.com/download/all/. Run the binary directly from Terminal: '/Applications/[Xcode 14.3.1.app]/Contents/MacOS/Xcode'. The .app bundle won't open normally on newer macOS, but running the binary directly bypasses this restriction. For quick access, add the /Contents/MacOS folder to Finder sidebar.

*Contributors: [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2023-11-28 · Tags: `xcode`, `mac`, `debugging`, `development-environment`*

---

## What is the AEGP memory leak issue on Mac during export with Compute Cache?

When using AEGP_MemorySuite to create memHandles during ComputeCache threads, the memory may not be properly freed during export on Mac (works fine on Windows). The used memory grows past the RAM limit. Using new/delete instead of AEGP memHandles avoids the leak. The issue is that memHandles allocated in one thread and freed in another may not actually release memory during the render thread. The AEGP tools report the memory as freed, but virtual memory keeps growing.

*Contributors: [**tlafo**](../contributors/tlafo/), [**dvb metareal**](../contributors/dvb-metareal/) · Source: adobe-plugin-devs · 2023-09-07 · Tags: `memory-leak`, `compute-cache`, `mac`, `export`, `aegp-memory-suite`, `threading`*

---

## How do I build an After Effects plugin for Apple Silicon (M1/ARM64)?

You need to do two things: (1) Set the Xcode build architecture to include arm64 (ensure the architecture setting targets ARM), and (2) Add 'CodeMacARM64 {"EffectMain"}' to your PiPL resource file. Without the PiPL entry, AE won't recognize the plugin as ARM-native even if the binary is compiled for arm64.

*Contributors: [**Nate**](../contributors/nate/) · Source: adobe-plugin-devs · 2022-04-28 · Tags: `apple-silicon`, `m1`, `arm64`, `pipl`, `xcode`, `mac`, `build-configuration`*

---

## How to compile unicode characters (e.g., Chinese) in C++ plugin parameter names and category names for Mac AE plugins?

The name field is defined as A_char[32]. Convert your string to UTF-8 and feed the param macro the resulting string. Make sure the length of the resulting string is no more than 31 chars long (including multibyte characters), as the last char must be used for null termination.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2025-06-01 · Tags: `unicode`, `utf8`, `mac`, `xcode`, `parameter-name`, `localization`*

---
