# C Plus Plus

> 1 Q&A · source: AE plugin dev community Discord

### What causes uninitialized PF_Err variables and subtle bugs?

A common C++ syntax error: 'PF_Err err, err2 = PF_Err_NONE;' only initializes err2, leaving err undefined (often ends up as 1). The correct form is 'PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;'. This can cause if(!err) checks to fail unexpectedly and lead to hard-to-track bugs like AEGP_GetNewEffectStreamByIndex returning 0x0.

```cpp
// WRONG - err is uninitialized:
PF_Err err, err2 = PF_Err_NONE;

// CORRECT - both initialized:
PF_Err err = PF_Err_NONE, err2 = PF_Err_NONE;
```

*Tags: `c-plus-plus`, `common-mistake`, `debugging`, `initialization`, `pf-err`*

---
