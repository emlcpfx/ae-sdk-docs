# Iteration

> 2 Q&As · source: AE plugin dev community Discord

### What is the difference between thread_indexL and i in iterate_generic?

The 'i' parameter is the iteration index (not sequential - may be reordered to minimize CPU cache conflicts), while thread_indexL is the thread index. With 4 threads and 8 iterations, i values may arrive as 6,0,2,7,4,5,1,3 on threads 3,0,1,3,2,2,0,1. They only match when iterations equals threads (e.g., PF_Iterations_ONCE_PER_PROCESSOR). The thread count doesn't change during an AE session. For image processing, it's better NOT to use PF_Iterations_ONCE_PER_PROCESSOR because some threads will finish early and sit idle. Using N iterations ensures minimal CPU power loss. Never call iterate_generic from within iterate_generic, and don't call suites from non-thread-0 threads.

*Tags: `iterate-generic`, `iteration`, `multithreading`, `performance`, `thread-index`*

---

### How can I efficiently access neighboring pixel values during CCU iteration without using sampleIntegral32 on every pixel?

You can use pointer math to access adjacent pixels directly from the input buffer pointer. If inP is a pointer to the current pixel, you can get the next pixel on the same line with `PF_Pixel *next = inP + 1;`. However, for better performance when needing multiple integer samples of other pixels, use iterate_generic instead of the regular iteration suites, as it allows much more optimization potential than acquiring and releasing the sample suite on every pixel.

```cpp
PF_Pixel *next = inP + 1; // gives you the next pixel on the same line
```

*Tags: `aegp`, `ccu`, `iteration`, `memory`, `mfr`, `performance`*

---
