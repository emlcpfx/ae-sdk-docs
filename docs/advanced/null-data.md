# Null Data

> 1 Q&A · source: AE plugin dev community Discord

### Why is checkout_layer_pixels returning data == NULL for a layer parameter in SmartRender?

In GPU mode (CUDA, Metal, OpenCL), the SDK documentation states that you must assume the data pointer is NULL in PF_EffectWorld. The pixel data lives on the GPU, not in CPU memory. This is expected behavior when GPU rendering is active.

*Tags: `checkout-layer`, `effect-world`, `gpu`, `null-data`, `smart-render`*

---
