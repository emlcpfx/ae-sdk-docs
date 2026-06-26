# Q&A: sampling

**7 entries** tagged with `sampling`.

---

## Why must X and Y positions be cast to FIX when using subpixel_sample_float, and is there a better sampling method for smoother displacement?

The subpixel_sample_float function requires FIX format for coordinate input despite being a float sampling function. FIX is a fixed-point format used by After Effects for precise sub-pixel positioning, not a simple integer rounding. For smoother displacement results comparable to vanilla AE or professional displacement plugins, ensure you are using the correct sampling suite and that your displacement values are calculated with sufficient precision before conversion. Consider reviewing the sampling parameters and whether alternative sampling methods or higher precision displacement calculations might improve quality.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
        INT2FIX(newX),
        INT2FIX(newY),
        &giP->inSamp_pb,
        &samplePixel));
```

*Tags: `sampling`, `displacement`, `mfr`, `quality`, `debugging`*

---

## Why must X and Y position arguments be cast to FIX when using subpixel_sample_float, and is there a better sampling method for smoother displacement?

The subpixel_sample_float function requires FIX format for coordinate arguments even though it returns float samples. This is part of AE's API design. For smoother displacement results comparable to vanilla AE or other displacement plugins, you should review sampling best practices in the Warbler plugin tutorial, which covers proper displacement sampling techniques.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
         INT2FIX(newX),
         INT2FIX(newY),
         &giP->inSamp_pb,
         &samplePixel));
```

*Tags: `mfr`, `gpu`, `sampling`, `displacement`, `debugging`*

---

## Why must X and Y coordinates be cast to FIX when using subpixel_sample_float, and what sampling method should be used for smoother displacement results?

When using the SamplingFloatSuite1()->subpixel_sample_float() function, coordinates are cast to FIX format (fixed-point representation) rather than remaining as floats. This is part of the After Effects API's internal coordinate system. The FIX format provides the necessary precision for the sampling algorithm while maintaining compatibility with AE's rendering pipeline. For smoother displacement results comparable to After Effects' native displacement or professional plugins, ensure you're using subpixel_sample_float with proper coordinate conversion, and consider whether your displacement values are being calculated with sufficient precision before being passed to the sampling function. The quality difference may also stem from how displacement offsets are computed rather than the sampling function itself.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref,
        INT2FIX(newX),
        INT2FIX(newY),
        &giP->inSamp_pb,
        &samplePixel));
```

*Tags: `sampling`, `displacement`, `mfr`, `output-rect`, `aegp`*

---

## Why must X and Y positions be cast to FIX when using subpixel_sample_float for displacement sampling?

When using the subpixel_sample_float function from the Sampling Float Suite, coordinates must be converted to FIX format (fixed-point notation) as required by the AE API, even though you're working with float values. This is a requirement of the sampling function signature, not a rounding operation that degrades quality. The issue with displacement quality may relate to the sampling method chosen rather than the FIX conversion itself. Consulting displacement plugin implementations and tutorials can help identify if a different sampling approach would yield better results.

```cpp
ERR(suites.SamplingFloatSuite1()->subpixel_sample_float(giP->inIn_data.effect_ref, INT2FIX(newX), INT2FIX(newY), &giP->inSamp_pb, &samplePixel));
```

*Tags: `sampling`, `displacement`, `params`, `reference`*

---

## How can I optimize subpixel sampling performance in After Effects plugins?

To optimize subpixel sampling, acquire the sampling suite once before your rendering loop using AEFX_AcquireSuite, get a direct pointer to it, and release it afterwards. Avoid acquiring and releasing the suite for each sample operation. Alternatively, define the AEGP_SuitesHandler before the loop and pass a pointer to it to the sampling function, though acquiring the suite directly is likely faster since it reduces function call overhead.

*Tags: `mfr`, `aegp`, `performance`, `sampling`, `render-loop`*

---

## What is the difference between PF_SUBPIXEL_SAMPLE and subpixel_sample in After Effects plugins?

PF_SUBPIXEL_SAMPLE is a macro that wraps the in_data->utils->subpixel_sample() function. Both provide the same sampling functionality. The macro approach can be used even though some documentation may mark it as unsupported. The key performance difference is not in which function you use, but in how you manage suite acquisition—acquire the sampling suite once before your loop rather than repeatedly inside it.

*Tags: `mfr`, `aegp`, `sampling`, `params`*

---

## How can I convert Pixel Bender code that samples and transforms pixel coordinates to a native After Effects plugin?

You can accomplish coordinate transformation and pixel sampling using the C++ API. The recommended approach is to pull values from input layers rather than push to arbitrary pixels. Iterate through output samples and pull input values from transformed locations. For direct buffer pixel access in RAM, study the CCU sample project. For iterating through output and pulling from other locations, examine the Shifter sample. Note that subpixel interpolation to 4 adjacent pixels is not supported by the standard suite.

*Tags: `pixel_bender`, `effect_api`, `c++`, `sampling`, `reference`*

---
