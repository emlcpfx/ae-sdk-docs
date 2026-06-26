# Mainthread

> 1 Q&A · source: AE plugin dev community Discord

### How can I safely use threads in an After Effects plugin while calling AEGP suite functions?

Threading is possible in After Effects plugins, but AEGP suite calls must only be made from the main thread. After Effects behaves unpredictably if suite calls are made from other threads, and the application can become unstable even after the worker thread has completed. Use threads only for background computation; marshal all AE API calls back to the main thread. Boost threads have been confirmed to work reliably when this constraint is followed.

```cpp
static void myFunc(){
  for(int i = 0; i<100; i++){
    // Do computation here, not AE API calls
  }
}
// Call AEGP suites only from main thread
ERR(suites.LayerSuite8()->AEGP_GetActiveLayer(&current_layerH));
```

*Tags: `aegp`, `mainthread`, `plugin-development`, `threading`*

---
