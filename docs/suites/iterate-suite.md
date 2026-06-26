# Iterate Suite

> 2 Q&As · source: AE plugin dev community Discord

### How can I use multithreading for pixel iteration in Premiere, since the AE Iterate8Suite doesn't work correctly?

The AE Iterate8Suite may not work correctly in Premiere and can cause rendering issues (e.g., sequence thumbnail updates but program window doesn't). For 8-bit BGRA, Premiere has its own dedicated iteration suite. For 32-bit float, you need to write your own iteration function. Use std::parallel or equivalent for multithreading. The SDK Noise example demonstrates BGRA and VUYA colorspace handling. The iterate suite should work in 8-bit ARGB (Suitev2 since 2022), but for BGRA you need the special Premiere suite.

*Tags: `bgra`, `colorspace`, `iterate-suite`, `multithreading`, `pixel-iteration`, `premiere`*

---

### Should I use the Iterate Suite with many iterations containing inner loops, or fewer iterations with larger inner loops?

When using the Iterate Suite with multithreading, the best approach depends on how well tasks can be split between cores rather than mathematical efficiency. If you have 4-75 outer iterations with 10,000 inner loop iterations, fewer large iterations would have less overhead. However, the optimal strategy is to break the work into smaller tasks that can run in parallel across available threads. With variable inner loop sizes (10-10,000), consider distributing tasks evenly across cores to avoid situations where some threads finish early and wait for others. Testing different configurations is recommended to find the optimal thread count for your specific workstation.

*Tags: `iterate-suite`, `multithreading`, `optimization`, `performance`, `threading`*

---
