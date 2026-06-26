# Iterate Generic

> 4 Q&As · source: AE plugin dev community Discord

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

### How can I debug iterate_generic functions properly in Visual Studio without erratic step-over behavior?

When debugging iterate_generic functions in Visual Studio, hit the breakpoint for the first time, then disable the breakpoint before stepping line by line. This prevents the debugger from jumping between threads (BEE WorkQueue and system DLLs) and allows normal line-by-line stepping.

*Tags: `debugging`, `iterate-generic`, `threading`, `windows`*

---
