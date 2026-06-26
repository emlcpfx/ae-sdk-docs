# Layers & Compositions

Working with layers, compositions, masks, and track mattes in AE plugins.

## Layer Checkout

To read pixels from another layer (or the same layer at a different time):

```cpp
PF_ParamDef checkout;
ERR(PF_CHECKOUT_PARAM(in_data, PARAM_LAYER,
    in_data->current_time, in_data->time_step, in_data->time_scale, &checkout));
// Use checkout.u.ld for the layer pixels
ERR(PF_CHECKIN_PARAM(in_data, &checkout));  // Always check back in!
```

## Masks & Track Mattes

- Masks are applied by AE **after** your plugin renders (unless you request otherwise)
- `PF_OutFlag_I_USE_SHUTTER_ANGLE` — needed if you need motion blur data
- Track matte issues? See [Track Matte Fix](../guides/track-matte-fix.md)

---

*10 documents in this section. Use **Search** (top of page) to find what you need.*
