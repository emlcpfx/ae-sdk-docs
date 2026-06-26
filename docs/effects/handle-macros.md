# Handle Management: PF_NEW_HANDLE, PF_DISPOSE_HANDLE, and Friends

After Effects provides handle-based memory management through macros defined in `AE_EffectCB.h`. Handles are opaque, relocatable memory references managed by the AE host. This document covers all handle macros, their relationship to the underlying callback functions, and best practices for safe handle usage.

---

## Why Handles?

Handles exist for historical reasons (classic Mac OS memory management). In modern AE (CS6+), handles behave much like allocated memory blocks -- locking and unlocking are no-ops -- but the API contract still requires proper lock/unlock discipline. Following the protocol ensures forward compatibility and correct behavior across all host versions.

---

## The Handle Macros

All macros are defined in `AE_EffectCB.h` and assume a variable named `in_data` of type `PF_InData*` is in scope.

### PF_NEW_HANDLE

```cpp
#define PF_NEW_HANDLE(SIZE) \
    (*in_data->utils->host_new_handle)((SIZE))
```

Allocates a new handle of the specified size in bytes. Returns `PF_Handle` (an opaque pointer type). Returns NULL on failure.

**SIZE type:** `A_u_longlong` (64-bit unsigned).

```cpp
PF_Handle h = PF_NEW_HANDLE(sizeof(MyData));
if (!h) {
    return PF_Err_OUT_OF_MEMORY;
}
```

### PF_DISPOSE_HANDLE

```cpp
#define PF_DISPOSE_HANDLE(PF_HANDLE) \
    (*in_data->utils->host_dispose_handle)((PF_Handle)(PF_HANDLE))
```

Frees the handle and its associated memory. After this call, the handle is invalid. Returns void.

```cpp
PF_DISPOSE_HANDLE(h);
h = NULL;  // Good practice -- prevent use-after-free
```

### PF_LOCK_HANDLE

```cpp
#define PF_LOCK_HANDLE(PF_HANDLE) \
    (*in_data->utils->host_lock_handle)((PF_Handle)(PF_HANDLE))
```

Locks the handle and returns a `void*` pointer to the underlying memory. While locked, the memory will not be relocated.

```cpp
MyData* dataP = (MyData*)PF_LOCK_HANDLE(h);
dataP->value = 42;
```

### PF_UNLOCK_HANDLE

```cpp
#define PF_UNLOCK_HANDLE(PF_HANDLE) \
    (*in_data->utils->host_unlock_handle)((PF_Handle)(PF_HANDLE))
```

Unlocks the handle. The pointer obtained from PF_LOCK_HANDLE should not be used after unlocking. Returns void.

```cpp
PF_UNLOCK_HANDLE(h);
dataP = NULL;  // Invalidate the pointer
```

### PF_GET_HANDLE_SIZE

```cpp
#define PF_GET_HANDLE_SIZE(PF_HANDLE) \
    (*in_data->utils->host_get_handle_size)((PF_Handle)(PF_HANDLE))
```

Returns the current size of the handle's data in bytes as `A_u_longlong`.

```cpp
A_u_longlong size = PF_GET_HANDLE_SIZE(h);
```

### PF_RESIZE_HANDLE

```cpp
// Takes a pointer to a handle. Handle may change. 4.1 and later ONLY.
#define PF_RESIZE_HANDLE(NEW_SIZE, PF_HANDLE_P) \
    (*in_data->utils->host_resize_handle)((NEW_SIZE), (PF_Handle*)(PF_HANDLE_P))
```

Resizes an existing handle. Returns `PF_Err`.

> **CRITICAL:** PF_RESIZE_HANDLE takes a **pointer to** the handle, not the handle itself. The handle value may change after resizing (the host may reallocate to a different address). Always pass `&handle`, not `handle`.

```cpp
PF_Handle h = PF_NEW_HANDLE(100);

// CORRECT: pass pointer-to-handle
PF_Err err = PF_RESIZE_HANDLE(200, &h);  // h may now point elsewhere

// WRONG: passing handle directly -- will corrupt memory
// PF_Err err = PF_RESIZE_HANDLE(200, h);  // DO NOT DO THIS
```

---

## Underlying Callback Functions

The macros wrap function pointers in `PF_UtilCallbacks`:

```cpp
typedef struct _PF_UtilCallbacks {
    // ...
    PF_Handle (*host_new_handle)(A_u_longlong size);
    void *    (*host_lock_handle)(PF_Handle pf_handle);
    void      (*host_unlock_handle)(PF_Handle pf_handle);
    void      (*host_dispose_handle)(PF_Handle pf_handle);
    A_u_longlong (*host_get_handle_size)(PF_Handle pf_handle);
    PF_Err    (*host_resize_handle)(A_u_longlong new_sizeL, PF_Handle *handlePH);
    // ...
} PF_UtilCallbacks;
```

If you need to call these without the macros (for example, from a function that does not have `in_data` in scope), you can pass `in_data->utils` and call them directly.

---

## Handle Suite (Alternative)

There is also a PICA suite for handle operations (`PF_HandleSuite1`), which provides the same functionality through a suite interface. The macros are more commonly used in effect code.

---

## Lock/Unlock Discipline

Since CS6 (AE 11.0), lock and unlock are effectively no-ops -- handles are never relocated. However, you should still follow the lock/unlock pattern for these reasons:

1. **Forward compatibility** -- future AE versions could re-introduce relocation
2. **Code clarity** -- makes the lifetime of raw pointers explicit
3. **Correctness** -- some third-party hosts may not treat them as no-ops

### The Pattern

```cpp
PF_Handle h = PF_NEW_HANDLE(sizeof(MyData));
if (!h) return PF_Err_OUT_OF_MEMORY;

MyData* ptr = (MyData*)PF_LOCK_HANDLE(h);

// Use ptr...
ptr->field = value;

PF_UNLOCK_HANDLE(h);
ptr = NULL;  // Do not use ptr after unlock
```

---

## Sequence Data Handle Pattern

The most common use of handles is for sequence data (`out_data->sequence_data`):

### Allocating in PF_Cmd_SEQUENCE_SETUP

```cpp
case PF_Cmd_SEQUENCE_SETUP:
{
    PF_Handle seq_h = PF_NEW_HANDLE(sizeof(MySequenceData));
    if (!seq_h) return PF_Err_OUT_OF_MEMORY;

    MySequenceData* seqP = (MySequenceData*)PF_LOCK_HANDLE(seq_h);
    AEFX_CLR_STRUCT(*seqP);
    seqP->version = 1;
    seqP->initialized = true;
    PF_UNLOCK_HANDLE(seq_h);

    out_data->sequence_data = seq_h;
    break;
}
```

### Reading in PF_Cmd_RENDER / PF_Cmd_SMART_RENDER

```cpp
case PF_Cmd_RENDER:
{
    MySequenceData* seqP = (MySequenceData*)PF_LOCK_HANDLE(in_data->sequence_data);
    if (!seqP) return PF_Err_INTERNAL_STRUCT_DAMAGED;

    // Use seqP->version, seqP->initialized, etc.

    PF_UNLOCK_HANDLE(in_data->sequence_data);
    break;
}
```

### Disposing in PF_Cmd_SEQUENCE_SETDOWN

```cpp
case PF_Cmd_SEQUENCE_SETDOWN:
{
    if (in_data->sequence_data) {
        PF_DISPOSE_HANDLE(in_data->sequence_data);
        out_data->sequence_data = NULL;
    }
    break;
}
```

### Flattening/Unflattening for Serialization

```cpp
case PF_Cmd_SEQUENCE_FLATTEN:
{
    MySequenceData* seqP = (MySequenceData*)PF_LOCK_HANDLE(in_data->sequence_data);

    // Create a flat (portable) version
    PF_Handle flat_h = PF_NEW_HANDLE(sizeof(MyFlatSequenceData));
    MyFlatSequenceData* flatP = (MyFlatSequenceData*)PF_LOCK_HANDLE(flat_h);

    flatP->version = seqP->version;
    // Copy data, convert pointers to offsets, handle endianness...

    PF_UNLOCK_HANDLE(flat_h);
    PF_UNLOCK_HANDLE(in_data->sequence_data);
    PF_DISPOSE_HANDLE(in_data->sequence_data);

    out_data->sequence_data = flat_h;
    break;
}
```

---

## RAII Wrapper Pattern

To prevent handle leaks when exceptions or early returns occur:

```cpp
class HandleScope {
    PF_InData* in_data_;
    PF_Handle handle_;
    void* ptr_;
public:
    HandleScope(PF_InData* in_data, PF_Handle h)
        : in_data_(in_data), handle_(h), ptr_(nullptr)
    {
        if (handle_) {
            ptr_ = PF_LOCK_HANDLE(handle_);
        }
    }

    ~HandleScope() {
        if (handle_) {
            PF_UNLOCK_HANDLE(handle_);
        }
    }

    template<typename T>
    T* as() { return reinterpret_cast<T*>(ptr_); }

    bool valid() const { return ptr_ != nullptr; }

    // Non-copyable
    HandleScope(const HandleScope&) = delete;
    HandleScope& operator=(const HandleScope&) = delete;
};

// Usage:
HandleScope scope(in_data, in_data->sequence_data);
if (!scope.valid()) return PF_Err_INTERNAL_STRUCT_DAMAGED;

MySequenceData* seqP = scope.as<MySequenceData>();
// seqP is valid for the lifetime of 'scope'
// Automatically unlocked when scope exits
```

For handles that should be disposed on scope exit (temporary allocations):

```cpp
class TempHandle {
    PF_InData* in_data_;
    PF_Handle handle_;
public:
    TempHandle(PF_InData* in_data, A_u_longlong size)
        : in_data_(in_data), handle_(PF_NEW_HANDLE(size)) {}

    ~TempHandle() {
        if (handle_) PF_DISPOSE_HANDLE(handle_);
    }

    PF_Handle get() const { return handle_; }
    void* lock() { return handle_ ? PF_LOCK_HANDLE(handle_) : nullptr; }
    void unlock() { if (handle_) PF_UNLOCK_HANDLE(handle_); }
    PF_Handle release() { PF_Handle h = handle_; handle_ = nullptr; return h; }
};
```

---

## Pitfalls

**PF_RESIZE_HANDLE takes a pointer-to-handle.** The most common bug with handles. The handle address itself may change. Always use `&h`.

**Do not use raw pointers after PF_UNLOCK_HANDLE.** Even though modern AE does not relocate, treat the pointer as invalid after unlock.

**Do not dispose handles you do not own.** The `input` world, `in_data`, and `params[]` are owned by AE. Only dispose handles you allocated with PF_NEW_HANDLE.

**AEFX_CLR_STRUCT your data after allocation.** PF_NEW_HANDLE does not zero-initialize memory. Use `AEFX_CLR_STRUCT(*ptr)` or `memset` to initialize.

```cpp
#define AEFX_CLR_STRUCT(STRUCT)  memset(&(STRUCT), 0, sizeof(STRUCT));
```

**Always check for NULL return from PF_NEW_HANDLE.** Return PF_Err_OUT_OF_MEMORY immediately on failure.
