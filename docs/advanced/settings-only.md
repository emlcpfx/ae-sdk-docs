# Settings Only

> 1 Q&A · source: AE plugin dev community Discord

### What is the simplest way to make a settings-only effect that doesn't modify the layer visually?

Several options: (1) Use AE's utils->Copy() function to copy the input buffer to the output buffer - it's fast. (2) Use PF_OutFlag_AUDIO_EFFECT_ONLY to make AE skip image rendering entirely (but you'll need to implement audio pass-through). (3) Setting the output rect size to 0 or negative during pre-render might cause AE to skip the render, though this isn't fully reliable.

*Tags: `audio-effect-only`, `copy-buffer`, `identity`, `pass-through`, `settings-only`*

---
