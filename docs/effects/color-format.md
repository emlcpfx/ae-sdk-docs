# Color Format

> 3 Q&As · source: AE plugin dev community Discord

### What is the pixel format difference between After Effects and OpenCV Mat?

After Effects uses ARGB format by default in dumb render mode, while OpenCV Mat uses BGRA format by default. The color channels are in different order, so you need to convert between them when copying data between the two formats.

*Tags: `color-format`, `pixel-data`*

---

### Why do colors shift between After Effects and Premiere Pro, especially on saturated reds and blues?

Premiere Pro delivers pixels in `VUYA_4444_32f` format — float `{V(Cr), U(Cb), Y(luma), A}` — using **BT.709** (ITU-R BT.709-6) color space. If you convert using BT.601 coefficients the luma weights and chroma scaling differ, producing a visible tint across the image.

BT.601 (wrong for Premiere):
```
R = Y + 1.402 * V
G = Y - 0.344 * U - 0.714 * V
B = Y + 1.772 * U
```

BT.709 (correct for Premiere HD content):
```
R = Y + 1.5748 * V
G = Y - 0.1873 * U - 0.4681 * V
B = Y + 1.8556 * U
```

*Tags: `color-format`, `color-space`, `premiere`, `vuya`, `bt709`*

---

### What is Premiere Pro's VUYA_4444_32f pixel layout, and how do I convert it to RGBA?

Premiere always delivers 32-bit float pixels in VUYA order — **not** RGBA or ARGB:

| Channel index | Name | Meaning |
|---|---|---|
| `float[0]` | V | Cr chroma (BT.709) |
| `float[1]` | U | Cb chroma (BT.709) |
| `float[2]` | Y | luma |
| `float[3]` | A | alpha |

Full BT.709 conversion (full-range):
```cpp
// VUYA → RGBA
float v = vuya[0], u = vuya[1], y = vuya[2], a = vuya[3];
float r = y + 1.5748f * v;
float g = y - 0.1873f * u - 0.4681f * v;
float b = y + 1.8556f * u;

// RGBA → VUYA
float y_out =  0.2126f * r + 0.7152f * g + 0.0722f * b;
float u_out = -0.1146f * r - 0.3854f * g + 0.5000f * b;
float v_out =  0.5000f * r - 0.4542f * g - 0.0458f * b;
```

Register VUYA support in `GlobalSetup` via `PF_PixelFormatSuite1::ClearSupportedPixelFormats` / `AddSupportedPixelFormat` with `PrPixelFormat_VUYA_4444_32f`. After Effects never uses VUYA — it always sends ARGB.

*Tags: `color-format`, `premiere`, `vuya`, `pixel-conversion`, `bt709`*

---
