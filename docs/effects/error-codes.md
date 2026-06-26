# Error Codes: Complete PF_Err Reference

All After Effects effect plugin error codes are defined in `AE_Effect.h`. This document covers every PF_Err value, what triggers each, how to handle them, and the ERR/ERR2 macro system for error propagation.

---

## The PF_Err Enumeration

```cpp
#define PF_FIRST_ERR  512

enum {
    PF_Err_NONE                     = 0,
    PF_Err_OUT_OF_MEMORY            = 4,
    PF_Err_INTERNAL_STRUCT_DAMAGED  = PF_FIRST_ERR,  // = 512
    PF_Err_INVALID_INDEX,                             // = 513
    PF_Err_UNRECOGNIZED_PARAM_TYPE,                   // = 514
    PF_Err_INVALID_CALLBACK,                          // = 515
    PF_Err_BAD_CALLBACK_PARAM,                        // = 516
    PF_Interrupt_CANCEL,                              // = 517
    PF_Err_CANNOT_PARSE_KEYFRAME_TEXT                 // = 518
};
typedef A_long PF_Err;
```

Note that `PF_Err` is `A_long` (a signed 32-bit integer), not an enum class. Values from 512 onward are AE-specific; value 4 maps to the system out-of-memory code.

---

## Error Code Reference

### PF_Err_NONE (0)

**Meaning:** No error. Operation succeeded.

**Usage:** Return this from every command selector handler when everything goes well. This is also the initial value you should assign to your `err` variable.

```cpp
PF_Err err = PF_Err_NONE;
```

---

### PF_Err_OUT_OF_MEMORY (4)

**Meaning:** Memory allocation failed.

**What triggers it:**
- `PF_NEW_HANDLE` when the system cannot allocate the requested size
- `PF_NEW_WORLD` / `PF_WorldSuite::PF_NewWorld` when there is insufficient memory for pixel buffers
- `PF_RESIZE_HANDLE` when the new size cannot be accommodated
- Any suite call that internally allocates memory

**How to handle:** Return this error immediately. Do not attempt fallback allocations in a loop. AE will display a memory warning to the user. Clean up any partially allocated resources before returning.

```cpp
PF_Handle h = PF_NEW_HANDLE(big_size);
if (!h) {
    return PF_Err_OUT_OF_MEMORY;
}
```

---

### PF_Err_INTERNAL_STRUCT_DAMAGED (512)

**Meaning:** An internal data structure is corrupted or invalid.

**What triggers it:**
- Passing a NULL or invalid `PF_EffectWorld` to a callback
- Corrupted `in_data` or `out_data` pointers
- Using a disposed handle
- Internal AE errors when processing your plugin's data

**How to handle:** This is a serious error. Log diagnostic information if possible and return immediately. This often indicates a programming error in the plugin (use-after-free, bad pointer arithmetic, etc.).

---

### PF_Err_INVALID_INDEX (513)

**Meaning:** An index is out of range, or the requested action is not allowed on this index.

**What triggers it:**
- Checking out a parameter with an index beyond `num_params`
- Accessing a layer parameter index that does not exist
- Channel suite operations with an out-of-range channel index

**How to handle:** Verify your parameter indices match your `PF_ADD_*` calls during `PF_Cmd_PARAMS_SETUP`. Remember that parameter index 0 is always the input layer.

---

### PF_Err_UNRECOGNIZED_PARAM_TYPE (514)

**Meaning:** The parameter type is not recognized by AE.

**What triggers it:**
- Checking out a parameter and treating it as the wrong type
- Plugin compiled against a newer SDK than the host supports
- Corrupted parameter data

**How to handle:** Verify you are accessing the correct union member of `PF_ParamDef` for the parameter type you registered. For example, do not read `.u.fd` (float slider) from a parameter registered as `PF_Param_CHECKBOX`.

---

### PF_Err_INVALID_CALLBACK (515)

**Meaning:** The requested callback function is NULL or invalid.

**What triggers it:**
- Calling a callback through `in_data->utils` when the utils pointer is NULL
- Calling a callback that does not exist in the current AE version
- Using `get_callback_addr` with an invalid `PF_CallbackID`

**How to handle:** Verify `in_data` and `in_data->utils` are non-NULL before calling callbacks. If targeting older AE versions, check for NULL function pointers before calling.

---

### PF_Err_BAD_CALLBACK_PARAM (516)

**Meaning:** A parameter passed to a callback is invalid.

**What triggers it:**
- Passing a NULL world pointer to sampling/iterate callbacks
- Passing invalid rectangles (right < left, bottom < top)
- Passing a source world that has been disposed
- `PF_SampPB` with invalid or uninitialized fields

**How to handle:** Validate all parameters before passing them to callbacks. Use `AEFX_CLR_STRUCT` to zero-initialize structures before filling them.

---

### PF_Interrupt_CANCEL (517)

**Meaning:** The user pressed Escape or Cmd+Period (Mac) to cancel rendering.

**What triggers it:**
- AE's progress/abort mechanism during `PF_ITERATE`, `PF_ITERATE16`, or any iterate variant
- Returned by `PF_ABORT` / `PF_PROGRESS` when the user has requested cancellation
- Internally during long-running suite calls

**How to handle:** This is **not** an error in the traditional sense. When you receive this value, you must propagate it up the call chain without modification. Do not overwrite it with `PF_Err_NONE`. Do not display error messages. Simply clean up and return.

```cpp
PF_Err err = PF_Err_NONE;

// iterate may return PF_Interrupt_CANCEL
err = PF_ITERATE(0, height, &input, NULL, &refcon, MyPixelFunc, &output);

// DO NOT do this:
// if (err == PF_Interrupt_CANCEL) err = PF_Err_NONE;  // WRONG!

// Just return whatever err is -- AE handles cancellation display
return err;
```

> **PITFALL:** The ERR() macro correctly handles PF_Interrupt_CANCEL -- once any error is set (including cancel), subsequent ERR() calls are skipped. But if you manually reset `err` to NONE after a cancel, subsequent operations will run on invalid/partial data.

---

### PF_Err_CANNOT_PARSE_KEYFRAME_TEXT (518)

**Meaning:** The effect's arbitrary data scan function could not parse the provided text.

**What triggers it:**
- Returned from your `PF_Arbitrary_SCAN_FUNC` when the text representation of arbitrary/custom parameter data cannot be parsed back into binary form
- Typically occurs during paste operations or preset loading

**How to handle:** Return this from your scan function when the input text is malformed. AE will report the error to the user. Ensure your `PRINT_FUNC` and `SCAN_FUNC` are symmetric -- anything PRINT produces, SCAN must be able to consume.

---

## The ERR and ERR2 Macros

Defined in `AE_Macros.h`, these macros implement a simple error-chaining pattern that is pervasive throughout SDK example code.

### ERR -- Execute if No Prior Error

```cpp
#define ERR(FUNC)  do { if (!err) { err = (FUNC); } } while (0)
```

If `err` is already non-zero, the function is **not called**. This lets you write sequential operations that bail out on first failure:

```cpp
PF_Err err = PF_Err_NONE;

ERR(PF_CHECKOUT_PARAM(in_data, PARAM_INPUT, ...));
ERR(PF_ITERATE(0, height, src, NULL, refcon, MyFunc, dst));
ERR(PF_CHECKIN_PARAM(in_data, &param));

return err;
```

### ERR2 -- Execute Always, Preserve First Error

```cpp
#define ERR2(FUNC)  do { if (((err2 = (FUNC)) != A_Err_NONE) && !err) err = err2; } while (0)
```

ERR2 **always** executes the function, even if `err` is already set. If `err` is zero and the function fails, `err` is updated. If `err` is already set, the new error is discarded.

Use ERR2 for **cleanup operations** that must run regardless of prior errors:

```cpp
PF_Err err = PF_Err_NONE;
PF_Err err2 = PF_Err_NONE;  // REQUIRED -- ERR2 writes to err2

ERR(PF_CHECKOUT_PARAM(in_data, PARAM_LAYER, ...));
ERR(do_something_with(param));

// Must check in even if do_something_with() failed
ERR2(PF_CHECKIN_PARAM(in_data, &param));

return err;  // returns the FIRST error that occurred
```

> **PITFALL:** You must declare `PF_Err err2 = PF_Err_NONE;` in the same scope. Forgetting this declaration causes a compile error.

---

## Error Handling Best Practices

### 1. Always Initialize err

```cpp
PF_Err err = PF_Err_NONE;
```

### 2. Use ERR for Sequential Operations

Chain dependent operations with ERR so that failure at any step skips the rest.

### 3. Use ERR2 for Mandatory Cleanup

Parameter check-in, handle disposal, and world disposal must happen even after errors.

### 4. Never Swallow PF_Interrupt_CANCEL

Always propagate cancellation. Hiding it causes AE to hang waiting for a response.

### 5. Return PF_Err from Your Entry Point

Your main `EffectMain` / `EntryPointFunc` must return `PF_Err`. AE inspects this value.

### 6. Do Not Invent Custom Error Codes

Return only the documented PF_Err values. AE does not understand arbitrary error numbers and may behave unpredictably.

---

## Error Code Summary Table

| Code | Name | Value | Severity | Action |
|---|---|---|---|---|
| `PF_Err_NONE` | Success | 0 | None | Continue |
| `PF_Err_OUT_OF_MEMORY` | Out of memory | 4 | Fatal | Clean up, return |
| `PF_Err_INTERNAL_STRUCT_DAMAGED` | Struct damaged | 512 | Fatal | Log, return |
| `PF_Err_INVALID_INDEX` | Bad index | 513 | Error | Fix index, return |
| `PF_Err_UNRECOGNIZED_PARAM_TYPE` | Bad param type | 514 | Error | Fix type access |
| `PF_Err_INVALID_CALLBACK` | Bad callback | 515 | Error | Check NULL |
| `PF_Err_BAD_CALLBACK_PARAM` | Bad param to CB | 516 | Error | Validate inputs |
| `PF_Interrupt_CANCEL` | User cancel | 517 | Normal | Propagate, do not suppress |
| `PF_Err_CANNOT_PARSE_KEYFRAME_TEXT` | Parse failure | 518 | Error | Return from SCAN_FUNC |
