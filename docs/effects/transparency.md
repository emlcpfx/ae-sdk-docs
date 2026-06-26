# Transparency

> 2 Q&As · source: AE plugin dev community Discord

### Why does a plugin produce black edges around precomposed layers when the same effect works fine on masked/keyed layers with transparency?

This is typically a compositing or alpha channel handling issue. When a layer is precomposed, the alpha channel behavior changes - the precomp creates a new composition with its own alpha, which can cause edge artifacts if the plugin isn't properly accounting for premultiplied vs. straight alpha, or if it's not respecting the layer's transparency correctly during rendering. The plugin may need to verify it's handling alpha channels consistently and that output rectangles are being computed correctly for precomposed material.

*Tags: `alpha`, `compositing`, `precomp`, `render-loop`, `transparency`*

---

### Why does a plugin produce black edges around results when applied to a precomposed layer that already has the effect applied internally?

This is typically caused by alpha channel handling in the output rect and how the plugin processes transparency across precomposition boundaries. When a layer is precomposed with effects already applied, the alpha channel information and output boundaries may not be properly respected by the plugin, causing black pixels to appear at edges where transparency should exist. The issue often relates to how the plugin reads and writes the output rectangle and handles premultiplied alpha in the composition pipeline.

*Tags: `layer-checkout`, `memory`, `output-rect`, `render-loop`, `transparency`*

---
