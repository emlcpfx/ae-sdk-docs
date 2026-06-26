# Displacement

> 5 Q&As · source: AE plugin dev community Discord

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

### Why must X and Y position arguments be cast to FIX when using subpixel_sample_float, and is there a better sampling method for smoother displacement?

The subpixel_sample_float function requires FIX format for coordinate arguments even though it returns float samples. This is part of AE's API design. For smoother displacement results comparable to vanilla AE or other displacement plugins, you should review sampling best practices in the Warbler plugin tutorial, which covers proper displacement sampling techniques.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
         INT2FIX(newX),
         INT2FIX(newY),
         &giP->inSamp_pb,
         &samplePixel));
```

*Tags: `debugging`, `displacement`, `gpu`, `mfr`, `sampling`*

---

### Why must X and Y coordinates be cast to FIX when using subpixel_sample_float, and what sampling method should be used for smoother displacement results?

When using the SamplingFloatSuite1()->subpixel_sample_float() function, coordinates are cast to FIX format (fixed-point representation) rather than remaining as floats. This is part of the After Effects API's internal coordinate system. The FIX format provides the necessary precision for the sampling algorithm while maintaining compatibility with AE's rendering pipeline. For smoother displacement results comparable to After Effects' native displacement or professional plugins, ensure you're using subpixel_sample_float with proper coordinate conversion, and consider whether your displacement values are being calculated with sufficient precision before being passed to the sampling function. The quality difference may also stem from how displacement offsets are computed rather than the sampling function itself.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
        INT2FIX(newX),
        INT2FIX(newY),
        &giP->inSamp_pb,
        &samplePixel));
```

*Tags: `aegp`, `displacement`, `mfr`, `output-rect`, `sampling`*

---

### Why must X and Y positions be cast to FIX when using subpixel_sample_float for displacement sampling?

When using the subpixel_sample_float function from the Sampling Float Suite, coordinates must be converted to FIX format (fixed-point notation) as required by the AE API, even though you're working with float values. This is a requirement of the sampling function signature, not a rounding operation that degrades quality. The issue with displacement quality may relate to the sampling method chosen rather than the FIX conversion itself. Consulting displacement plugin implementations and tutorials can help identify if a different sampling approach would yield better results.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref, INT2FIX(newX), INT2FIX(newY), &giP->inSamp_pb, &samplePixel));
```

*Tags: `displacement`, `params`, `reference`, `sampling`*

---

### Is there a tutorial or reference for building displacement plugins in After Effects?

The Warbler plugin tutorial from MacTech Magazine (Vol. 15, Issue 9) provides guidance on building displacement effects in After Effects plugins. It covers the fundamentals of displacement plugin development and can help understand proper sampling techniques and implementation practices.

*Tags: `displacement`, `reference`, `tutorial`*

---
