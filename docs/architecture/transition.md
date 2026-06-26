# Transition

> 3 Q&As · source: AE plugin dev community Discord

### Why can I only select the first two items in a PF_ADD_POPUPX dropdown with 3 elements in Premiere, and the third snaps back to 2?

This can be caused by opening a Premiere project with the transition already applied (stale cached state). Applying the transition plugin fresh resolves the issue and the dropdowns work normally. Also ensure that each popup choice string has the pipe separator '|' at the end of each line (except the last), and that you have a proper AEFX_CLR_STRUCT(def) before the popup definition.

```cpp
AEFX_CLR_STRUCT(def);

PF_ADD_POPUPX("Color", 3, 2, 
    "Medidata Green|"
    "Navy Blue|"
    "3DS Steel Blue"
    , NULL, SDK_CROSSDISSOLVE_COLOUR);
```

*Tags: `cache-bug`, `dropdown`, `parameter`, `popup`, `premiere`, `transition`*

---

### Does Premiere support SmartFX for requesting only a portion of the input buffer?

Premiere does not support SmartFX. You cannot officially request only a specific region of the buffer like you can in AE. As a workaround, you can manually iterate through the data using row bytes to delimit the part you need. Be careful with negative row byte values in Premiere.

*Tags: `buffer`, `premiere`, `region-of-interest`, `row-bytes`, `smartfx`, `transition`*

---

### How can you limit the output buffer to a specific region when Premiere doesn't support smartFX for transitions?

Since Premiere Pro does not support smartFX like After Effects does, you cannot use the smartFX output_rect mechanism to specify a smaller region of interest. When building transitions for Premiere, the plugin receives frames at the full sequence resolution even if the source footage is smaller. You need to work with the full buffer dimensions provided by Premiere's API rather than requesting a subset.

*Tags: `output-rect`, `premiere`, `smartfx`, `transition`*

---
