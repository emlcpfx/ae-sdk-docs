# Multithreading

> 7 Q&As · source: AE plugin dev community Discord

### How can I use multithreading for pixel iteration in Premiere, since the AE Iterate8Suite doesn't work correctly?

The AE Iterate8Suite may not work correctly in Premiere and can cause rendering issues (e.g., sequence thumbnail updates but program window doesn't). For 8-bit BGRA, Premiere has its own dedicated iteration suite. For 32-bit float, you need to write your own iteration function. Use std::parallel or equivalent for multithreading. The SDK Noise example demonstrates BGRA and VUYA colorspace handling. The iterate suite should work in 8-bit ARGB (Suitev2 since 2022), but for BGRA you need the special Premiere suite.

*Tags: `bgra`, `colorspace`, `iterate-suite`, `multithreading`, `pixel-iteration`, `premiere`*

---

### What is the difference between thread_indexL and i in iterate_generic?

The 'i' parameter is the iteration index (not sequential - may be reordered to minimize CPU cache conflicts), while thread_indexL is the thread index. With 4 threads and 8 iterations, i values may arrive as 6,0,2,7,4,5,1,3 on threads 3,0,1,3,2,2,0,1. They only match when iterations equals threads (e.g., PF_Iterations_ONCE_PER_PROCESSOR). The thread count doesn't change during an AE session. For image processing, it's better NOT to use PF_Iterations_ONCE_PER_PROCESSOR because some threads will finish early and sit idle. Using N iterations ensures minimal CPU power loss. Never call iterate_generic from within iterate_generic, and don't call suites from non-thread-0 threads.

*Tags: `iterate-generic`, `iteration`, `multithreading`, `performance`, `thread-index`*

---

### Can I use iterate_generic to parallelize transform_world calls?

No, transform_world is not safe to call from utility threads. AE has main threads (UI and render) and utility threads (used by iteration suite). Interaction with AE is only allowed from main threads. transform_world is already internally multithreaded, so calling it from parallel utility threads won't help and may cause bugs/crashes. Very few API callbacks are safe from non-main threads (e.g., subpixel sample callbacks, but suites must be acquired in advance from main threads). If you need parallel image transformations, write your own transform function with nearest-neighbor sampling that can safely run on multiple threads.

*Tags: `iterate-generic`, `multithreading`, `performance`, `thread-safety`, `transform-world`*

---

### Should I use multithreading (iterate_generic) or MFR (Multi-Frame Rendering) in my plugin, or both?

Use both. MFR and multithreading can coexist because not all steps in the single-frame rendering pipeline are multi-threadable. MFR shines for sequential parts, while multithreaded parts (via iterate callbacks) are critical for performance. Mixing MFR with MT doesn't necessarily cause competition because MT-friendly code tends to be CPU cache friendly, while other parts may leave the CPU waiting for RAM. When using iteration callbacks, AE may prioritize between MFR and MT threads. The AE engineering team tested these scenarios extensively.

*Tags: `iterate-generic`, `mfr`, `multi-frame-rendering`, `multithreading`, `performance`*

---

### What is the recommended approach for multithreading in Premiere plugins when the iterate suite doesn't work?

If the Iterate8Suite doesn't work reliably in Premiere (particularly with certain colorspace configurations), you should implement your own multithreading code instead of relying on the suite. Standard approaches include using std::parallel on Windows and equivalent threading libraries for Mac (like Grand Central Dispatch or pthreads). The iterate suite should theoretically work for 8-bit ARGB, but if you encounter issues, custom multithreading implementations are a stable fallback.

*Tags: `bgra`, `macos`, `multithreading`, `performance`, `premiere`, `threading`, `windows`*

---

### Can you use the Iterate8Suite in Premiere Pro plugins, and what are the alternatives for multithreading?

The Iterate8Suite has issues in Premiere Pro and doesn't work reliably with multithreading. The suite should work in 8-bit ARGB colorspace (SuiteV2 since 2022), but for BGRA you must use the special BGRA suite. If you need multithreading, implement your own using standard libraries like std::parallel for cross-platform compatibility.

*Tags: `aegp`, `multithreading`, `premiere`, `threading`*

---

### Should I use the Iterate Suite with many iterations containing inner loops, or fewer iterations with larger inner loops?

When using the Iterate Suite with multithreading, the best approach depends on how well tasks can be split between cores rather than mathematical efficiency. If you have 4-75 outer iterations with 10,000 inner loop iterations, fewer large iterations would have less overhead. However, the optimal strategy is to break the work into smaller tasks that can run in parallel across available threads. With variable inner loop sizes (10-10,000), consider distributing tasks evenly across cores to avoid situations where some threads finish early and wait for others. Testing different configurations is recommended to find the optimal thread count for your specific workstation.

*Tags: `iterate-suite`, `multithreading`, `optimization`, `performance`, `threading`*

---
