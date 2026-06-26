# Copy Protection

> 1 Q&A · source: AE plugin dev community Discord

### How can you make a watermark harder to remove from a plugin's trial output?

Several techniques: (1) Make the pixel colors of the watermark vary randomly to prevent color keying. (2) Add alpha blending with the source layer. (3) Make the cross/border wider or vary the shape. (4) Add text/noise. (5) Use a few-color gradient instead of random noise for a prettier but still hard-to-key result. The key is randomizing per-pixel colors with rand() to prevent simple removal via color keying.

*Tags: `copy-protection`, `licensing`, `trial`, `watermark`*

---
