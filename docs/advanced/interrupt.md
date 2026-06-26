# Interrupt

> 1 Q&A · source: AE plugin dev community Discord

### Why does transform_world give bad results when dragging a slider, and what is the correct first argument?

The SDK documentation is wrong about putting in_data as the first argument to transform_world. The first argument is actually the effect_ref (in_data->effect_ref), but passing NULL instead works better. When using effect_ref, transform_world appears to check for interrupts, which causes it to halt the transformation during slider dragging for smoother interactive experience, but this results in incorrect output. Passing NULL avoids this interrupt check. Many suite callbacks accept NULL instead of effect_ref.

*Tags: `documentation-bug`, `effect-ref`, `interrupt`, `rendering`, `transform-world`*

---
