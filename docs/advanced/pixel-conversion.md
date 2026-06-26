# Pixel Conversion

> 1 Q&A · source: AE plugin dev community Discord

### How to avoid noise artifacts when AE automatically converts 16/32-bit project input to 8-bit for an 8-bit-only plugin?

AE's automatic bit-depth conversion can introduce dithering/noise artifacts. The recommended approach is to let your plugin accept 16 and 32 bpc inputs as well and do the conversion to 8-bit yourself internally. This avoids the warning sign next to your plugin and gives users the benefit of having the output back in 16/32 bpc. You could also report the noise issue as a bug to Adobe.

*Tags: `8-bit`, `bit-depth`, `dithering`, `noise`, `pixel-conversion`*

---
