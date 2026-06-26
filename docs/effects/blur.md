# Blur

> 1 Q&A · source: AE plugin dev community Discord

### How can I use AEGP_FastBlur from WorldSuite2 within an effect plugin instead of an AEGP?

If you need to blur the input layer, you must copy its content into an AEGP_WorldH. However, for intermediate processing, you can create an AEGP_WorldH and use AEGP_FillOutPFEffectWorld to wrap the AEGP world into an effect world, allowing you to access the same world in both forms.

*Tags: `aegp`, `blur`, `effectworld`, `sdk`, `worldsuite`*

---
