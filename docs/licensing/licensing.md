# Licensing

> 4 Q&As · source: AE plugin dev community Discord

### What floating license server options exist for OFX plugins (e.g., for Flame)?

RLM (Reprise License Manager) is one option used by companies like Maxon for their plugins. However, the options for floating license servers tend to be pricey. The aescripts licensing system does not support floating licenses. For Flame-specific plugins with Linux floating license requirements, RLM is the most commonly referenced solution in the community.

*Tags: `flame`, `floating-license`, `licensing`, `linux`, `ofx`, `rlm`*

---

### How can you make a watermark harder to remove from a plugin's trial output?

Several techniques: (1) Make the pixel colors of the watermark vary randomly to prevent color keying. (2) Add alpha blending with the source layer. (3) Make the cross/border wider or vary the shape. (4) Add text/noise. (5) Use a few-color gradient instead of random noise for a prettier but still hard-to-key result. The key is randomizing per-pixel colors with rand() to prevent simple removal via color keying.

*Tags: `copy-protection`, `licensing`, `trial`, `watermark`*

---

### Are developers allowed to redistribute the Adobe SDKs publicly?

Most likely no. You should check with Adobe directly, but the general expectation is that redistribution is not permitted.

*Tags: `adobe`, `licensing`, `redistribution`, `sdk`*

---

### Are there licensing restrictions around Blender plugins or their marketplace?

There are some licensing restrictions around Blender plugins, particularly if you use their marketplace.

*Tags: `deployment`, `licensing`*

---
