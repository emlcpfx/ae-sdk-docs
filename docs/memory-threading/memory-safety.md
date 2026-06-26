# Memory Safety

> 1 Q&A · source: AE plugin dev community Discord

### What are best practices for error handling and memory safety in AE SDK plugins?

Recommendations: (1) Wrap AE handles in C++ RAII classes with proper destructors, use smart pointers. (2) Use scope guards (like the pattern from Dr. Dobb's) for cleanup on any exit path. (3) Use std::expected (C++23) or Result types to chain operations safely. (4) Implement an allocation manager that clears allocations when exiting entry functions, since AE handle pointers are not valid between calls. (5) In debug builds, add a cache validator to detect extra memory still allocated at function exit points. (6) Use custom breakpoint assertions that trigger breakpoints when debugger is attached instead of crashing.

*Tags: `best-practice`, `error-handling`, `memory-safety`, `raii`, `scope-guard`, `smart-pointers`*

---
