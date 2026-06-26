# Memory Leak

> 4 Q&As · source: AE plugin dev community Discord

### Why can't macOS Instruments detect PF_EffectWorld memory leaks?

Instruments won't report PF_EffectWorld leaks because AE legitimately thinks you're holding important data in those worlds. AE allocates the worlds deep in its engine, not directly in your plugin code. While memory allocated with new/malloc within the plugin gets reported accurately (because Instruments has more context), AE-managed allocations are invisible to the leak detector. Creating C++ RAII wrappers around AE SDK objects for automatic memory management is recommended.

*Tags: `debugging`, `effect-world`, `instruments`, `mac`, `memory-leak`, `raii`*

---

### What is the AEGP memory leak issue on Mac during export with Compute Cache?

When using AEGP_MemorySuite to create memHandles during ComputeCache threads, the memory may not be properly freed during export on Mac (works fine on Windows). The used memory grows past the RAM limit. Using new/delete instead of AEGP memHandles avoids the leak. The issue is that memHandles allocated in one thread and freed in another may not actually release memory during the render thread. The AEGP tools report the memory as freed, but virtual memory keeps growing.

*Tags: `aegp-memory-suite`, `compute-cache`, `export`, `mac`, `memory-leak`, `threading`*

---

### How does GLator (deprecated OpenGL sample) have memory leaks?

The GLator sample wraps GL rendering in try/catch. The suites.IterateFloatSuite1()->iterate call can throw PF_Interrupt_CANCEL in the download texture function, causing bufferH to never be deallocated. This leaks an entire output buffer of 32bpc pixels per frame from CPU, plus GPU FBO resources. Using scope guards (C++ RAII cleanup on scope exit) prevents this pattern - they handle cleanup regardless of how the scope is exited (exception, return, etc.).

*Tags: `exception-handling`, `glator`, `memory-leak`, `opengl`, `scope-guard`*

---

### Is AEGP_RenderAndCheckoutLayerFrame_Async safe to use for rendering multiple frames?

There is a known bug with the async version of RenderAndCheckout related to memory releasing. The issue was discussed between several developers and the AE team. All developers reverted back to the synchronous version, doing their best to render one frame at a time on each idle cycle to avoid making the UI laggy. The frame receipt cannot be properly checked in when using the async version.

*Tags: `async-render`, `bug`, `frame-checkout`, `memory-leak`, `render-suite`*

---
