# Thread Index

> 1 Q&A · source: AE plugin dev community Discord

### What is the difference between thread_indexL and i in iterate_generic?

The 'i' parameter is the iteration index (not sequential - may be reordered to minimize CPU cache conflicts), while thread_indexL is the thread index. With 4 threads and 8 iterations, i values may arrive as 6,0,2,7,4,5,1,3 on threads 3,0,1,3,2,2,0,1. They only match when iterations equals threads (e.g., PF_Iterations_ONCE_PER_PROCESSOR). The thread count doesn't change during an AE session. For image processing, it's better NOT to use PF_Iterations_ONCE_PER_PROCESSOR because some threads will finish early and sit idle. Using N iterations ensures minimal CPU power loss. Never call iterate_generic from within iterate_generic, and don't call suites from non-thread-0 threads.

*Tags: `iterate-generic`, `iteration`, `multithreading`, `performance`, `thread-index`*

---
