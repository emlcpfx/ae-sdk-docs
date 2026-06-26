# Slider

> 1 Q&A · source: AE plugin dev community Discord

### How can I create a parameter slider with a negative range and percentage display in After Effects?

Use PF_ADD_FLOAT_SLIDERX (or PF_ADD_FIXED for integer values) with negative minimum and maximum values, combined with the PF_ValueDisplayFlag_PERCENT flag. Example: PF_ADD_FLOAT_SLIDERX("Length", -100, 100, -100, 100, 100, PF_Precision_TENTHS, PF_ValueDisplayFlag_PERCENT, NULL, PARAM_LENGTH_ID); This allows creating sliders with ranges like -100% to 100%, similar to Element 3D sliders.

```cpp
PF_ADD_FLOAT_SLIDERX("Length", -100, 100, -100, 100, 100, PF_Precision_TENTHS, PF_ValueDisplayFlag_PERCENT, NULL, PARAM_LENGTH_ID);
```

*Tags: `params`, `slider`, `ui`*

---
