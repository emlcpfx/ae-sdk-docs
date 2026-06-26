# Compiler

> 1 Q&A · source: AE plugin dev community Discord

### Why does Premiere give a NULL input_world in the Render call when compiled with fast optimizations (-Ofast), but works without optimizations?

This is a known issue that occurs in Premiere but not AE when using -Ofast compiler optimizations. A workaround is to add __attribute__((optnone)) before the render function to disable optimization for that specific function while keeping the rest of the plugin optimized. Another suggestion is to check if the input world is null and return PF_Err_OUT_OF_MEMORY (not bad param), which may cause Premiere to retry the render. Also verify your colorspace setup in GlobalSetup and consider trying BGRA instead of ARGB.

```cpp
__attribute__ ((optnone))
static PF_Err Render(...) {
    // render code here
}
```

*Tags: `compiler`, `null-input-world`, `ofast`, `optimization`, `premiere`, `render`, `workaround`*

---
