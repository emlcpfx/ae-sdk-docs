# Ccu

> 2 Q&As · source: AE plugin dev community Discord

### How can I efficiently access neighboring pixel values during CCU iteration without using sampleIntegral32 on every pixel?

You can use pointer math to access adjacent pixels directly from the input buffer pointer. If inP is a pointer to the current pixel, you can get the next pixel on the same line with `PF_Pixel *next = inP + 1;`. However, for better performance when needing multiple integer samples of other pixels, use iterate_generic instead of the regular iteration suites, as it allows much more optimization potential than acquiring and releasing the sample suite on every pixel.

```cpp
PF_Pixel *next = inP + 1; // gives you the next pixel on the same line
```

*Tags: `aegp`, `ccu`, `iteration`, `memory`, `mfr`, `performance`*

---

### What is the performance impact of acquiring and releasing the sample suite on every pixel during iteration?

Acquiring and releasing the sample suite on every pixel significantly weighs down performance. For optimal results when you need to sample many adjacent pixels, it's recommended to access the input and output buffers directly and use iterate_generic instead, which allows each thread to handle a single buffer line and eliminates the overhead of repeated suite acquisition/release operations.

*Tags: `ccu`, `memory`, `optimization`, `performance`, `threading`*

---
