# Effects & Processing

Image processing operations — color, compositing, blending, alpha, masks.

## Alpha and Premultiplication

This is the #1 source of visual bugs in AE plugins.

- **AE pixel buffers are premultiplied** — RGB values are pre-multiplied by alpha
- **Straight alpha** (unpremultiplied) must be converted before writing to output
- **Black edges/halos** = you're treating premultiplied data as straight, or vice versa

See [Black Edges & Premultiplication](../guides/black-edges-premult.md) for the full guide.

## Compositing

- **`transfer_rect`** — SDK function for compositing one world onto another with blend modes
- **`PF_TransferMode`** — Controls blend mode (Normal, Add, Multiply, etc.)
- See [Compositing Operations](../guides/compositing-operations.md) and [Transfer Modes](../guides/sdk-transfer-modes.md)

## Color Processing

- Always handle all three bit depths (8/16/32) or set appropriate PiPL flags
- 32-bit float values can exceed 0.0–1.0 (HDR) — don't clamp unless intentional
- See [Float Color Handling](../guides/float-color-handling.md)

## 3D Lighting

- [AE Lights](ae-lights.md) — fetch sequence, stream units, and the **empirically-derived falloff math** for SMOOTH (`1 - smoothstep`) and INVERSE_SQUARE_CLAMPED (`(R/d)²`, distance ignored). Includes the propagation-direction-in-row-2 fact, additive composition, and downsample-invariance.
- [Lighting](lighting.md) — Q&A on extracting AE camera + light matrices for Vulkan / WebGPU / Metal shaders.

For the camera side of the same problem, see
[AE Camera Matrix Conventions](../coordinates/ae-camera-matrix-conventions.md)
and [AE Camera Pipeline](../coordinates/ae-camera-pipeline.md).

---

*44 documents in this section. Use **Search** (top of page) to find what you need.*
