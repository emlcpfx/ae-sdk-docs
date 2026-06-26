# Multiple Instances

> 1 Q&A · source: AE plugin dev community Discord

### How do you generate unique checkout IDs for checkout_layer/checkout_layer_pixels in SmartFX with multiple layers and instances?

This is a known challenge when dealing with multiple layers, multiple instances, and nested loops in SmartFX PreRender/SmartRender. The checkout ID must be unique across all checkouts. No definitive solution was provided in the discussion, but the issue typically arises with many cloned plugin instances resulting in 'checkout id is not unique' errors.

*Tags: `checkout-layer`, `multiple-instances`, `prerender`, `smartfx`, `smartrender`, `unique-id`*

---
