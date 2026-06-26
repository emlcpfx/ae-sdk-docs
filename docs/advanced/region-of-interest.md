# Region Of Interest

> 1 Q&A · source: AE plugin dev community Discord

### Does Premiere support SmartFX for requesting only a portion of the input buffer?

Premiere does not support SmartFX. You cannot officially request only a specific region of the buffer like you can in AE. As a workaround, you can manually iterate through the data using row bytes to delimit the part you need. Be careful with negative row byte values in Premiere.

*Tags: `buffer`, `premiere`, `region-of-interest`, `row-bytes`, `smartfx`, `transition`*

---
