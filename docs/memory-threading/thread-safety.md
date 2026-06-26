# Thread Safety

> 3 Q&As · source: AE plugin dev community Discord

### How much work is required to update a simple AE plugin for Multi-Frame Rendering (MFR)?

For simple plugins that don't use sequence data or caching, there is not much to do besides rebuilding. The complexity comes when you have GPU plugins, custom caching, or sequence data which require more significant refactoring for thread safety.

*Tags: `mfr`, `multi-frame-rendering`, `plugin-update`, `thread-safety`*

---

### What is the Compute Cache API and how does it differ from sequence data and arb data?

Compute Cache replaces sequence data for MFR-compatible caching. Key differences: (1) Compute Cache can be read/written from any thread (render or UI), (2) it is NOT saved with the project (unlike arb data), (3) it's designed for per-frame computed values. Register it in GlobalSetup with AEGP_ClassRegister. The key function should include effect ref, layer ID, effect position, current time, and time scale. Pass in_data->pica_basicP through the options refcon pointer to access suites inside callback functions. Must be registered during GlobalSetup, first access possible during ParamSetup.

```cpp
#include "AE_ComputeCacheSuite.h"

struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
};

A_Err MyGenerateKeyFunc(AEGP_CCComputeOptionsRefconP optionsP, AEGP_CCComputeKeyP out_keyP) {
    access_cache_data* accessCacheData = static_cast<access_cache_data*>(optionsP);
    SPBasicSuite* bsuite = accessCacheData->in_data->pica_basicP;
    // Generate key using AEGP_HashSuite1...
    return A_Err_NONE;
}

static PF_Err GlobalSetup(...) {
    AEFX_SuiteScoper<AEGP_ComputeCacheSuite1> compute_suite(in_data, kAEGPComputeCacheSuite, kAEGPComputeCacheSuiteVersion1);
    AEGP_ComputeCacheCallbacks callbacks = { MyGenerateKeyFunc, MyComputeFunc, MyApproxSizeValueFunc, MyDeleteComputeValueFunc };
    compute_suite->AEGP_ClassRegister("com.mycompany.effect.myComputeCacheClass", &callbacks);
}
```

*Tags: `arb-data`, `caching`, `compute-cache`, `mfr`, `sequence-data`, `thread-safety`*

---

### Can I use iterate_generic to parallelize transform_world calls?

No, transform_world is not safe to call from utility threads. AE has main threads (UI and render) and utility threads (used by iteration suite). Interaction with AE is only allowed from main threads. transform_world is already internally multithreaded, so calling it from parallel utility threads won't help and may cause bugs/crashes. Very few API callbacks are safe from non-main threads (e.g., subpixel sample callbacks, but suites must be acquired in advance from main threads). If you need parallel image transformations, write your own transform function with nearest-neighbor sampling that can safely run on multiple threads.

*Tags: `iterate-generic`, `multithreading`, `performance`, `thread-safety`, `transform-world`*

---
