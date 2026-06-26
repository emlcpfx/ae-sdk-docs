# Parameter

> 2 Q&As · source: AE plugin dev community Discord

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

### How can I hide an effect parameter in After Effects SDK?

Set the AEGP_DynStreamFlag_HIDDEN flag during PF_Cmd_UPDATE_PARAMS_UI. It should get called when a project loads and the effect is shown for the first time. Note that PF_PUI_NO_ECW_UI in ui_flags only removes it from the effect window but not from the timeline. There is a known AE bug where duplicating an effect may cause the hidden parameter's owner-drawn portion to still appear in some cases.

*Tags: `dynamic-stream`, `hidden`, `parameter`, `ui-flags`, `update-params-ui`*

---
