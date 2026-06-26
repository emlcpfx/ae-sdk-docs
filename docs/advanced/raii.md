# Raii

> 2 Q&As · source: AE plugin dev community Discord

### Why can't macOS Instruments detect PF_EffectWorld memory leaks?

Instruments won't report PF_EffectWorld leaks because AE legitimately thinks you're holding important data in those worlds. AE allocates the worlds deep in its engine, not directly in your plugin code. While memory allocated with new/malloc within the plugin gets reported accurately (because Instruments has more context), AE-managed allocations are invisible to the leak detector. Creating C++ RAII wrappers around AE SDK objects for automatic memory management is recommended.

*Tags: `debugging`, `effect-world`, `instruments`, `mac`, `memory-leak`, `raii`*

---

### What are best practices for error handling and memory safety in AE SDK plugins?

Recommendations: (1) Wrap AE handles in C++ RAII classes with proper destructors, use smart pointers. (2) Use scope guards (like the pattern from Dr. Dobb's) for cleanup on any exit path. (3) Use std::expected (C++23) or Result types to chain operations safely. (4) Implement an allocation manager that clears allocations when exiting entry functions, since AE handle pointers are not valid between calls. (5) In debug builds, add a cache validator to detect extra memory still allocated at function exit points. (6) Use custom breakpoint assertions that trigger breakpoints when debugger is attached instead of crashing.

*Tags: `best-practice`, `error-handling`, `memory-safety`, `raii`, `scope-guard`, `smart-pointers`*

---
