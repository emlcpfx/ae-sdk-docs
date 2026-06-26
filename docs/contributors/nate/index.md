# Nate

**2 contributions** to AE SDK community knowledge.

Top topics: `premiere`, `popup`, `dropdown`, `parameter`, `transition`, `cache-bug`, `apple-silicon`, `m1`, `arm64`, `pipl`

---

## Why can I only select the first two items in a PF_ADD_POPUPX dropdown with 3 elements in Premiere, and the third snaps back to 2?

This can be caused by opening a Premiere project with the transition already applied (stale cached state). Applying the transition plugin fresh resolves the issue and the dropdowns work normally. Also ensure that each popup choice string has the pipe separator '|' at the end of each line (except the last), and that you have a proper AEFX_CLR_STRUCT(def) before the popup definition.

```cpp
AEFX_CLR_STRUCT(def);

PF_ADD_POPUPX("Color", 3, 2, 
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Source: adobe-plugin-devs · 2022-12-17 · Tags: `premiere`, `popup`, `dropdown`, `parameter`, `transition`, `cache-bug` · [View in Q&A](../qa/premiere/)*

---

## How do I build an After Effects plugin for Apple Silicon (M1/ARM64)?

You need to do two things: (1) Set the Xcode build architecture to include arm64 (ensure the architecture setting targets ARM), and (2) Add 'CodeMacARM64 {"EffectMain"}' to your PiPL resource file. Without the PiPL entry, AE won't recognize the plugin as ARM-native even if the binary is compiled for arm64.

*Source: adobe-plugin-devs · 2022-04-28 · Tags: `apple-silicon`, `m1`, `arm64`, `pipl`, `xcode`, `mac`, `build-configuration` · [View in Q&A](../qa/apple-silicon/)*

---
