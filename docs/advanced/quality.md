# Quality

> 1 Q&A · source: AE plugin dev community Discord

### Why must X and Y positions be cast to FIX when using subpixel_sample_float, and is there a better sampling method for smoother displacement?

The subpixel_sample_float function requires FIX format for coordinate input despite being a float sampling function. FIX is a fixed-point format used by After Effects for precise sub-pixel positioning, not a simple integer rounding. For smoother displacement results comparable to vanilla AE or professional displacement plugins, ensure you are using the correct sampling suite and that your displacement values are calculated with sufficient precision before conversion. Consider reviewing the sampling parameters and whether alternative sampling methods or higher precision displacement calculations might improve quality.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
        INT2FIX(newX),
        INT2FIX(newY),
        &giP->inSamp_pb,
        &samplePixel));
```

*Tags: `debugging`, `displacement`, `mfr`, `quality`, `sampling`*

---
