# Parameters

> 2 Q&As · source: AE plugin dev community Discord

### What are good naming conventions for checkbox parameters in AE effect plugins?

Verb-based names like "Reduce Noise" work well for checkboxes and sound natural. Noun-based alternatives like "Reducing Noise", "Reduced Noise", or "Noise Reducing" sound more awkward. Parameters are generally nouns, but checkboxes are an exception where verbs are acceptable.

*Tags: `checkbox`, `naming-conventions`, `parameters`, `ui-design`*

---

### What is the challenge in matching After Effects' lighting calculations in a custom 3D plugin?

The main challenge is making all parameters mean the same thing with the same units as AE's native lighting system. This includes matching properties like spotlight falloff softness and the balance between ambient and direct lighting. While many plugins attempt this, it is complex to replicate precisely.

*Tags: `3d`, `effects`, `gpu`, `parameters`*

---
