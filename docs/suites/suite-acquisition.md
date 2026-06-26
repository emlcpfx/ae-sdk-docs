# Suite Acquisition: SuiteHandler vs SuiteScoper vs Manual

After Effects provides several patterns for acquiring PICA suites. Each has different trade-offs around convenience, safety, and flexibility. This document covers all four acquisition patterns and when to use each.

---

## Background: SPBasicSuite

All suite acquisition ultimately flows through `SPBasicSuite`, which provides two core functions:

```cpp
SPErr (*AcquireSuite)(const char *name, int32_t version, const void **suite);
SPErr (*ReleaseSuite)(const char *name, int32_t version);
```

Every suite you acquire **must** be released. Failure to release causes resource leaks. The patterns below differ in how they automate this lifecycle.

---

## Pattern 1: AEGP_SuiteHandler (Legacy, Convenience)

**Header:** `AEGP_SuiteHandler.h` (in `Examples/Util/`)

**Status:** Deprecated by Adobe as of 2014, but still widely used.

AEGP_SuiteHandler is a large convenience class that wraps dozens of AEGP and PF suites with named accessor methods. It lazily acquires suites on first access and releases them all in the destructor.

### How It Works

```cpp
// Construction requires SPBasicSuite*
AEGP_SuiteHandler suites(in_data->pica_basicP);

// Access suites by name -- acquired lazily, cached
const PF_WorldSuite2* ws = suites.WorldSuite2();
const PF_IterateFloatSuite1* fs = suites.IterateFloatSuite1();
```

### Key Characteristics

- **Throws on missing suite**: If acquisition fails, it calls `MissingSuiteError()` which you must define. This function must throw; it cannot return.
- **Lazy loading**: Suites are only acquired when first accessed.
- **Automatic release**: All acquired suites are released in the destructor.
- **Fixed suite roster**: Only suites explicitly coded into the class are available. Adding new suites requires modifying the source.

### You Must Provide MissingSuiteError

```cpp
// In your .cpp file:
void AEGP_SuiteHandler::MissingSuiteError() const
{
    A_THROW(A_Err_MISSING_SUITE);
}
```

### When to Use

- AEGP plugins that need many suites simultaneously
- Quick prototyping where you want named accessors
- Legacy codebases already using this pattern

### When to Avoid

- New code (Adobe recommends AEFX_SuiteScoper instead)
- When you need optional suite support (no ALLOW_NO_SUITE equivalent)
- When you need suites not in the pre-built roster

---

## Pattern 2: AEFX_SuiteScoper<T> (Recommended RAII)

**Header:** `AEFX_SuiteHandlerTemplate.h`

AEFX_SuiteScoper is a lightweight RAII template that acquires a single suite on construction and releases it on destruction. This is Adobe's recommended pattern for new code.

### Signature

```cpp
template<typename SUITETYPE, bool ALLOW_NO_SUITE = false>
class AEFX_SuiteScoper
{
public:
    AEFX_SuiteScoper(
        const PF_InData *in_data,
        const char *suite_name,
        int32_t suite_versionL,
        PF_OutData *out_dataP0 = 0,         // optional, for error display
        const char *err_stringZ0 = 0         // optional, custom error message
    );

    const SUITETYPE* operator->() const;
    const SUITETYPE* get() const;
};
```

### Basic Usage

```cpp
// Acquire PF_WorldSuite2 -- throws A_Err_MISSING_SUITE on failure
AEFX_SuiteScoper<PF_WorldSuite2> world_suite(
    in_data,
    kPFWorldSuite,
    kPFWorldSuiteVersion2
);

// Use via operator->
PF_EffectWorld temp_world;
world_suite->PF_NewWorld(in_data->effect_ref, 100, 100,
    PF_NewWorldFlag_CLEAR_PIXELS, &temp_world);
```

### With Error Display

```cpp
AEFX_SuiteScoper<PF_WorldSuite2> world_suite(
    in_data,
    kPFWorldSuite,
    kPFWorldSuiteVersion2,
    out_data,                               // enables error display
    "World Suite not available!"            // shown in AE error dialog
);
```

When `out_data` is provided and acquisition fails, the scoper sets `PF_OutFlag_DISPLAY_ERROR_MESSAGE` and formats the error string into `out_data->return_msg` before throwing.

### ALLOW_NO_SUITE for Optional Features

```cpp
// Will NOT throw if suite is missing -- get() returns NULL instead
AEFX_SuiteScoper<PF_IterateFloatSuite2, true> float_suite(
    in_data,
    kPFIterateFloatSuite,
    kPFIterateFloatSuiteVersion2
);

// Always check before use!
if (float_suite.get() != nullptr) {
    float_suite->iterate_origin_float(/* ... */);
} else {
    // Fallback to 16-bit path
}
```

**ALLOW_NO_SUITE defaults to false**. When set to `true`, acquisition failure silently sets the internal pointer to NULL instead of throwing.

### When to Use

- **All new effect plugin code** -- this is the recommended pattern
- When you need a single suite for a specific scope
- When you need optional suite support (ALLOW_NO_SUITE)
- When you want clear error messages shown to the user

### When to Avoid

- AEGP plugins where you lack PF_InData (use SuiteHelper or manual instead)
- When you need to acquire many suites and want named accessors

---

## Pattern 3: SuiteHelper<T> (Traits-Based, SPBasicSuite*)

**Header:** `SuiteHelper.h`

SuiteHelper is a traits-based RAII template that takes `SPBasicSuite*` directly instead of `PF_InData*`. It relies on template specialization of `SuiteTraits<T>` to supply the suite name and version.

### Signature

```cpp
template <typename SuiteType>
struct SuiteTraits
{
    static const A_char* i_name;
    static const int32_t i_version;
};

template<typename SuiteType, class MissingSuiteErrFunc = AssertAndThrowOnMissingSuite<SuiteType>>
class SuiteHelper
{
public:
    SuiteHelper(const SPBasicSuite* const basic_suiteP);
    const SuiteType* operator->() const;
    SuiteType* get() const;
};
```

### Usage

First, specialize `SuiteTraits` for your suite type (typically in a header):

```cpp
template<>
const A_char* SuiteTraits<PF_WorldSuite2>::i_name = kPFWorldSuite;

template<>
const int32_t SuiteTraits<PF_WorldSuite2>::i_version = kPFWorldSuiteVersion2;
```

Then use:

```cpp
SuiteHelper<PF_WorldSuite2> world_suite(in_data->pica_basicP);
world_suite->PF_NewWorld(/* ... */);
```

### Custom Error Handling

The default `MissingSuiteErrFunc` asserts and throws. You can substitute `MissingSuiteErrFunc_NoOp` for optional suites:

```cpp
// Silently swallows missing suite -- get() returns NULL
SuiteHelper<PF_WorldSuite2, MissingSuiteErrFunc_NoOp> world_suite(basic_suiteP);
if (world_suite.get()) {
    // suite available
}
```

### When to Use

- AEGP plugins where you have `SPBasicSuite*` but not `PF_InData*`
- When you want compile-time suite name/version binding via traits
- When you want to define suite mappings once and reuse them

### When to Avoid

- Effect plugins (prefer AEFX_SuiteScoper -- it takes PF_InData directly)
- One-off suite acquisitions where the traits boilerplate is not worth it

---

## Pattern 4: Manual SPBasicSuite Acquisition

For maximum control, acquire and release suites directly.

### Basic Pattern

```cpp
PF_Err err = PF_Err_NONE;
const PF_WorldSuite2* wsP = nullptr;

err = in_data->pica_basicP->AcquireSuite(
    kPFWorldSuite,
    kPFWorldSuiteVersion2,
    reinterpret_cast<const void**>(&wsP)
);

if (err || !wsP) {
    // Handle missing suite
    return PF_Err_INTERNAL_STRUCT_DAMAGED;
}

// Use the suite...
wsP->PF_NewWorld(/* ... */);

// MUST release when done
in_data->pica_basicP->ReleaseSuite(kPFWorldSuite, kPFWorldSuiteVersion2);
```

### RAII Wrapper

If you go manual, always wrap in RAII to guarantee release:

```cpp
class ManualSuiteScope {
    SPBasicSuite* basic_;
    const char* name_;
    int32_t version_;
    const void* suite_;
public:
    ManualSuiteScope(SPBasicSuite* basic, const char* name, int32_t version)
        : basic_(basic), name_(name), version_(version), suite_(nullptr)
    {
        basic_->AcquireSuite(name_, version_, &suite_);
    }
    ~ManualSuiteScope() {
        if (suite_) basic_->ReleaseSuite(name_, version_);
    }
    template<typename T>
    const T* as() const { return reinterpret_cast<const T*>(suite_); }
    bool valid() const { return suite_ != nullptr; }
};
```

### Version Negotiation Pattern

Manual acquisition is ideal when you need to fall back across suite versions:

```cpp
const PF_WorldSuite2* ws2 = nullptr;
SPErr spErr = pica->AcquireSuite(kPFWorldSuite, 2, (const void**)&ws2);
if (spErr || !ws2) {
    // Fall back to version 1
    const PF_WorldSuite1* ws1 = nullptr;
    spErr = pica->AcquireSuite(kPFWorldSuite, 1, (const void**)&ws1);
    if (!spErr && ws1) {
        // Use v1 API...
        pica->ReleaseSuite(kPFWorldSuite, 1);
    }
} else {
    // Use v2 API...
    pica->ReleaseSuite(kPFWorldSuite, 2);
}
```

### When to Use

- C plugins (no C++ templates available)
- Situations where you need version negotiation (try version 2, fall back to version 1)
- Platform or host abstraction layers

---

## Comparison Table

| Feature | AEGP_SuiteHandler | AEFX_SuiteScoper | SuiteHelper | Manual |
|---|---|---|---|---|
| RAII release | Yes | Yes | Yes | No (DIY) |
| Takes PF_InData* | No | Yes | No | No |
| Takes SPBasicSuite* | Yes | No (uses it internally) | Yes | Yes |
| Missing suite handling | Throws (must define) | Throws or NULL (template param) | Throws or no-op (template param) | Your choice |
| Lazy loading | Yes | No (immediate) | No (immediate) | N/A |
| Multiple suites | Yes (one object) | No (one per suite) | No (one per suite) | N/A |
| C compatible | No | No | No | Yes |
| Recommended by Adobe | No (deprecated) | Yes | Neutral | Neutral |

---

## Pitfalls

**Do not cache suite pointers across command selectors.** AE may invalidate suites between calls. Acquire suites fresh each time your entry point is called.

**Do not mix acquire/release across different SPBasicSuite pointers.** Always release through the same `SPBasicSuite*` you acquired from.

**AEFX_SuiteScoper requires PF_InData -- not available in all contexts.** During `PF_Cmd_GLOBAL_SETUP`, `in_data->pica_basicP` is valid, so AEFX_SuiteScoper works. But in standalone AEGP code with no PF_InData, use SuiteHelper or manual acquisition.

**AEGP_SuiteHandler.cpp must be compiled into your project.** It is not a header-only class. You also must provide `MissingSuiteError()`.

---

## FAQ

### What causes the 'Not able to acquire AEFX Suite' error in Premiere?

This error can occur during playback in Premiere when code attempts to acquire an AE-specific suite that is not available in the Premiere host. Check if you have a shared function that conditionally calls AEFX suites based on host detection (e.g., if app_id != 'PrMr' then call AEFX suite). The host detection condition might be returning the wrong result, causing Premiere to try acquiring AE-only suites.
