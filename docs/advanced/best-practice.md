# Best Practice

> 2 Q&As · source: AE plugin dev community Discord

### What are best practices for error handling and memory safety in AE SDK plugins?

Recommendations: (1) Wrap AE handles in C++ RAII classes with proper destructors, use smart pointers. (2) Use scope guards (like the pattern from Dr. Dobb's) for cleanup on any exit path. (3) Use std::expected (C++23) or Result types to chain operations safely. (4) Implement an allocation manager that clears allocations when exiting entry functions, since AE handle pointers are not valid between calls. (5) In debug builds, add a cache validator to detect extra memory still allocated at function exit points. (6) Use custom breakpoint assertions that trigger breakpoints when debugger is attached instead of crashing.

*Tags: `best-practice`, `error-handling`, `memory-safety`, `raii`, `scope-guard`, `smart-pointers`*

---

### What is a best practice for managing PiPL outflags in plugin source code?

Define outflags as preprocessor macros in your plugin header file, then reference these macros in your PiPL resource. This way the flags are centrally defined and automatically updated during compilation, eliminating the need to manually touch the PiPL values. Example: define FX_OUT_FLAGS with all necessary flag constants (like PF_OutFlag_FORCE_RERENDER, PF_OutFlag_DEEP_COLOR_AWARE, etc.) and use them in AE_Effect_Global_OutFlags and AE_Effect_Global_OutFlags_2 sections.

```cpp
#define FX_OUT_FLAGS (  PF_OutFlag_FORCE_RERENDER + \
                        PF_OutFlag_USE_OUTPUT_EXTENT + \
                        PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                        PF_OutFlag_DEEP_COLOR_AWARE + \
                        PF_OutFlag_CUSTOM_UI + \
                        PF_OutFlag_I_DO_DIALOG )

AE_Effect_Global_OutFlags {
    FX_OUT_FLAGS
},
AE_Effect_Global_OutFlags_2 {
    FX_OUT_FLAGS2
}
```

*Tags: `best-practice`, `build`, `params`, `pipl`*

---
