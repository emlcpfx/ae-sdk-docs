# Aegp Memory Suite

> 1 Q&A · source: AE plugin dev community Discord

### What is the AEGP memory leak issue on Mac during export with Compute Cache?

When using AEGP_MemorySuite to create memHandles during ComputeCache threads, the memory may not be properly freed during export on Mac (works fine on Windows). The used memory grows past the RAM limit. Using new/delete instead of AEGP memHandles avoids the leak. The issue is that memHandles allocated in one thread and freed in another may not actually release memory during the render thread. The AEGP tools report the memory as freed, but virtual memory keeps growing.

*Tags: `aegp-memory-suite`, `compute-cache`, `export`, `mac`, `memory-leak`, `threading`*

---
