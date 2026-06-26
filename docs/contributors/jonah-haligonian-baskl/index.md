# Jonah (Haligonian/Baskl)

**3 contributions** to AE SDK community knowledge.

Top topics: `debugging`, `logging`, `outputdebugstring`, `windows`, `cpp`, `watermark`, `licensing`, `trial`, `copy-protection`, `smartrender`

---

## How do you display a progress banner (like 'Computing: X%') that updates during SmartRender in After Effects?

The technique involves using a hidden parameter to trigger re-renders. When the user presses a button, code changes a hidden parameter which triggers AE to call SmartRender again. Each SmartRender call checks the current state and draws the appropriate banner. Adobe's internal 3D Camera Tracker uses PF_Cmd_RESERVED3 (an undocumented idle command selector that AE calls repeatedly) to check background work status and sets PF_OutFlag_FORCE_RERENDER to trigger SmartRender calls. Parameter values can only be changed during PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_EVENT. An alternative is to use AEGP_RegisterIdleHook with hidden parameter changes, though you don't have access to PF_OutData (and thus PF_OutFlag_FORCE_RERENDER) in an idle hook. In Premiere Pro, PF_OutFlag_FORCE_RERENDER doesn't work, so the hidden parameter trick is the recommended approach.

*Source: aescripts discord · 2026-03-05 · Tags: `smartrender`, `progress-bar`, `re-render`, `hidden-parameter`, `force-rerender`, `pf-cmd-reserved3`, `idle-hook` · [View in Q&A](../qa/smartrender/)*

---

## How can you make a watermark harder to remove from a plugin's trial output?

Several techniques: (1) Make the pixel colors of the watermark vary randomly to prevent color keying. (2) Add alpha blending with the source layer. (3) Make the cross/border wider or vary the shape. (4) Add text/noise. (5) Use a few-color gradient instead of random noise for a prettier but still hard-to-key result. The key is randomizing per-pixel colors with rand() to prevent simple removal via color keying.

*Source: aescripts discord · 2025-10-04 · Tags: `watermark`, `licensing`, `trial`, `copy-protection` · [View in Q&A](../qa/watermark/)*

---

## How do you debug/log variables from a C++ AE plugin on Windows?

Use OutputDebugString to send debug output that can be viewed with tools like DebugView or Visual Studio's Output window.

```cpp
std::string log1 = "variable: " + std::to_string(variable) + "\n"; OutputDebugString(log1.c_str());
```

*Source: aescripts discord · 2024-05-17 · Tags: `debugging`, `logging`, `outputdebugstring`, `windows`, `cpp` · [View in Q&A](../qa/debugging/)*

---
