# Effects

> 3 Q&As · source: AE plugin dev community Discord

### How can you leverage After Effects' rendering engine (3D lights, effects) while maintaining control over individual layers in a plugin?

One approach is to use the GPU route with sprites while keeping flexibility for individual layers. This allows you to take advantage of AE's built-in rendering engine capabilities like 3D lights and effects on each layer, rather than trying to replicate these features manually in your own 3D plugin.

*Tags: `3d`, `effects`, `gpu`, `layer-checkout`, `render-loop`*

---

### What is the challenge in matching After Effects' lighting calculations in a custom 3D plugin?

The main challenge is making all parameters mean the same thing with the same units as AE's native lighting system. This includes matching properties like spotlight falloff softness and the balance between ambient and direct lighting. While many plugins attempt this, it is complex to replicate precisely.

*Tags: `3d`, `effects`, `gpu`, `parameters`*

---

### How can I toggle a stopwatch or create keyframes for slider parameters programmatically in an Effects plugin?

You can create keyframes for any parameter using AEGP_KEYFRAMESUITE3. Most AEGP suites work for both AEGP plugins and Effects plugins, including the keyframe suite. The "CheesyCheese" sample project demonstrates how to implement this functionality.

*Tags: `aegp`, `effects`, `keyframe`, `params`*

---
