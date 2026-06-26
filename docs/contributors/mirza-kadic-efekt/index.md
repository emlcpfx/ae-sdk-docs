# Mirza Kadic (EFEKT)

**2 contributions** to AE SDK community knowledge.

Top topics: `smartfx`, `prerender`, `smartrender`, `code-organization`, `cpp`, `dll`, `watermark`, `licensing`, `trial`, `copy-protection`

---

## How can you make a watermark harder to remove from a plugin's trial output?

Several techniques: (1) Make the pixel colors of the watermark vary randomly to prevent color keying. (2) Add alpha blending with the source layer. (3) Make the cross/border wider or vary the shape. (4) Add text/noise. (5) Use a few-color gradient instead of random noise for a prettier but still hard-to-key result. The key is randomizing per-pixel colors with rand() to prevent simple removal via color keying.

*Source: aescripts discord · 2025-10-04 · Tags: `watermark`, `licensing`, `trial`, `copy-protection` · [View in Q&A](../qa/watermark/)*

---

## Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Source: aescripts discord · 2025-07-10 · Tags: `smartfx`, `prerender`, `smartrender`, `code-organization`, `cpp`, `dll` · [View in Q&A](../qa/smartfx/)*

---
