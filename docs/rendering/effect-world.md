# Effect World

> 2 Q&As · source: AE plugin dev community Discord

### Why is checkout_layer_pixels returning data == NULL for a layer parameter in SmartRender?

In GPU mode (CUDA, Metal, OpenCL), the SDK documentation states that you must assume the data pointer is NULL in PF_EffectWorld. The pixel data lives on the GPU, not in CPU memory. This is expected behavior when GPU rendering is active.

*Tags: `checkout-layer`, `effect-world`, `gpu`, `null-data`, `smart-render`*

---

### Why can't macOS Instruments detect PF_EffectWorld memory leaks?

Instruments won't report PF_EffectWorld leaks because AE legitimately thinks you're holding important data in those worlds. AE allocates the worlds deep in its engine, not directly in your plugin code. While memory allocated with new/malloc within the plugin gets reported accurately (because Instruments has more context), AE-managed allocations are invisible to the leak detector. Creating C++ RAII wrappers around AE SDK objects for automatic memory management is recommended.

*Tags: `debugging`, `effect-world`, `instruments`, `mac`, `memory-leak`, `raii`*

---
