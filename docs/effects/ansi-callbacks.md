# ANSI Math Callbacks: PF_COS, PF_SQRT, PF_SPRINTF and Friends

## Overview

The After Effects SDK provides a set of callback macros that wrap standard C math and string functions. These are defined in `AE_EffectCB.h` and route through the `PF_ANSICallbacks` structure embedded in `in_data->utils->ansi`. Rather than calling `cos()` or `sqrt()` directly from the C runtime library, plugins call `PF_COS()` or `PF_SQRT()`, which invoke function pointers provided by the host application.

## Why These Exist

These callbacks were introduced for two practical reasons:

1. **Cross-platform portability.** In the early days of the SDK (Classic Mac OS and Windows), C runtime libraries differed in behavior, precision, and availability. By routing math calls through the host, Adobe guaranteed consistent results across platforms.

2. **Avoiding CRT linkage issues.** On some platforms and build configurations, linking against the C runtime library from a plugin could cause symbol conflicts or require distributing additional DLLs. The host-provided callbacks eliminated this dependency entirely.

Both concerns are largely historical on modern systems (macOS and Windows both ship reliable CRTs), but the callbacks remain in the SDK and are still used by many existing plugins.

## The PF_ANSICallbacks Structure

All callbacks are stored in a structure defined in `AE_EffectCB.h`:

```c
typedef struct {
    A_FpLong  (*atan)(A_FpLong);
    A_FpLong  (*atan2)(A_FpLong y, A_FpLong x);   /* returns atan(y/x) - note param order! */
    A_FpLong  (*ceil)(A_FpLong);                    /* returns next int above x */
    A_FpLong  (*cos)(A_FpLong);
    A_FpLong  (*exp)(A_FpLong);                     /* returns e to the x power */
    A_FpLong  (*fabs)(A_FpLong);                    /* returns absolute value of x */
    A_FpLong  (*floor)(A_FpLong);                   /* returns closest int below x */
    A_FpLong  (*fmod)(A_FpLong x, A_FpLong y);     /* returns x mod y */
    A_FpLong  (*hypot)(A_FpLong x, A_FpLong y);    /* returns sqrt(x*x + y*y) */
    A_FpLong  (*log)(A_FpLong);                     /* returns natural log of x */
    A_FpLong  (*log10)(A_FpLong);                   /* returns log base 10 of x */
    A_FpLong  (*pow)(A_FpLong x, A_FpLong y);      /* returns x to the y power */
    A_FpLong  (*sin)(A_FpLong);
    A_FpLong  (*sqrt)(A_FpLong);
    A_FpLong  (*tan)(A_FpLong);

    int       (*sprintf)(A_char *, const A_char *, ...);
    A_char *  (*strcpy)(A_char *, const A_char *);

    A_FpLong  (*asin)(A_FpLong);
    A_FpLong  (*acos)(A_FpLong);

    A_long    ansi_procs[1];
} PF_ANSICallbacks;
```

Note that `asin` and `acos` appear at the end of the structure, after `sprintf` and `strcpy`. They were added later than the original math functions.

The trailing `ansi_procs[1]` field is a common SDK pattern for future expandability -- it provides a slot for additional function pointers without changing the structure size seen by older plugins.

## Complete Macro Reference

All macros require `in_data` to be in scope (which it always is inside any `PF_Cmd_*` handler).

### Math Callbacks

| Macro | Signature | C Equivalent | Notes |
|-------|-----------|--------------|-------|
| `PF_ACOS(X)` | `A_FpLong -> A_FpLong` | `acos(x)` | Arc cosine, result in radians |
| `PF_ASIN(X)` | `A_FpLong -> A_FpLong` | `asin(x)` | Arc sine, result in radians |
| `PF_ATAN(X)` | `A_FpLong -> A_FpLong` | `atan(x)` | Arc tangent, result in radians |
| `PF_ATAN2(Y, X)` | `(A_FpLong, A_FpLong) -> A_FpLong` | `atan2(y, x)` | Note: Y comes first, matching C convention |
| `PF_CEIL(X)` | `A_FpLong -> A_FpLong` | `ceil(x)` | Smallest integer >= x |
| `PF_COS(X)` | `A_FpLong -> A_FpLong` | `cos(x)` | Cosine, argument in radians |
| `PF_EXP(X)` | `A_FpLong -> A_FpLong` | `exp(x)` | e raised to the power x |
| `PF_FABS(X)` | `A_FpLong -> A_FpLong` | `fabs(x)` | Absolute value |
| `PF_FLOOR(X)` | `A_FpLong -> A_FpLong` | `floor(x)` | Largest integer <= x |
| `PF_FMOD(X, Y)` | `(A_FpLong, A_FpLong) -> A_FpLong` | `fmod(x, y)` | Floating-point remainder of x/y |
| `PF_HYPOT(X, Y)` | `(A_FpLong, A_FpLong) -> A_FpLong` | `hypot(x, y)` | sqrt(x*x + y*y), avoids overflow |
| `PF_LOG(X)` | `A_FpLong -> A_FpLong` | `log(x)` | Natural logarithm (base e) |
| `PF_LOG10(X)` | `A_FpLong -> A_FpLong` | `log10(x)` | Logarithm base 10 |
| `PF_POW(X, Y)` | `(A_FpLong, A_FpLong) -> A_FpLong` | `pow(x, y)` | x raised to the power y |
| `PF_SIN(X)` | `A_FpLong -> A_FpLong` | `sin(x)` | Sine, argument in radians |
| `PF_SQRT(X)` | `A_FpLong -> A_FpLong` | `sqrt(x)` | Square root |
| `PF_TAN(X)` | `A_FpLong -> A_FpLong` | `tan(x)` | Tangent, argument in radians |

### String Callbacks

| Macro | Signature | C Equivalent | Notes |
|-------|-----------|--------------|-------|
| `PF_SPRINTF` | `(A_char *, const A_char *, ...) -> int` | `sprintf` | See special definition note below |
| `PF_STRCPY(DST, SRC)` | `(A_char *, const A_char *) -> A_char *` | `strcpy` | Simple string copy |

### Related Constant

The header also defines a convenience constant:

```c
#define PF_SQRT2  1.41421356237309504880
```

## PF_SPRINTF: The Special Case

All of the math macros follow the same pattern -- they wrap their argument(s) in parentheses:

```c
#define PF_COS(X)     (*in_data->utils->ansi.cos)(X)
#define PF_ATAN2(Y,X) (*in_data->utils->ansi.atan2)((Y), (X))
```

`PF_SPRINTF` is defined differently because it is a variadic function. The C preprocessor cannot forward variadic arguments through a macro that takes parenthesized parameters. Instead, it is defined as a bare function pointer dereference:

```c
/* This is kind of a hack to deal with the varargs params to sprintf */
#define PF_SPRINTF  (*in_data->utils->ansi.sprintf)
```

This means you call it exactly like `sprintf`, with the parentheses and arguments supplied by you at the call site:

```c
A_char buf[256];
PF_SPRINTF(buf, "Frame %d: value = %.2f", frame_num, some_value);
```

If `PF_SPRINTF` were defined with macro arguments like `PF_SPRINTF(DST, FMT, ...)`, it would not work on older compilers that lack `__VA_ARGS__` support, and even with `__VA_ARGS__` the behavior has edge cases around zero variadic arguments. The "hack" comment in the header acknowledges this design choice.

## Macro Expansion Details

Each math macro dereferences a function pointer stored in the `PF_ANSICallbacks` struct. For example, `PF_COS(angle)` expands to:

```c
(*in_data->utils->ansi.cos)(angle)
```

This path is:
1. `in_data` -- the `PF_InData *` passed to every command handler
2. `->utils` -- pointer to the `PF_UtilCallbacks` structure
3. `->ansi` -- the embedded `PF_ANSICallbacks` struct (not a pointer)
4. `.cos` -- the function pointer for cosine
5. Dereference and call with the argument

## Should Modern Plugins Still Use Them?

**Short answer:** You do not have to, but there are reasons you might.

**Arguments for using them:**
- They are guaranteed to work. Adobe tests them.
- Some developers prefer the consistency of routing everything through the host.
- If you ever need to support an unusual platform or build environment, the callbacks avoid any CRT dependency questions.

**Arguments for using standard C math directly:**
- Modern compilers on Windows and macOS have reliable, well-tested math libraries.
- Direct calls can be inlined by the compiler, while callback function pointers cannot.
- In performance-critical inner loops (per-pixel processing), the function pointer indirection adds measurable overhead compared to a direct `sin()` or `sqrt()` call.
- C++ code using `<cmath>` functions integrates more naturally with templates, constexpr, and compiler optimizations.

**Practical recommendation:** Use standard `<cmath>` functions in performance-critical rendering code. Use `PF_SPRINTF` when you want the convenience and it is available (i.e., you have `in_data` in scope). There is no correctness risk in using standard library functions on modern platforms.

> **Warning:** If you use `PF_SPRINTF` or any ANSI callback, `in_data` must be valid and non-null. These macros will crash immediately if called when `in_data` is not available, such as in standalone utility functions that do not receive the `PF_InData` pointer. In those cases, use standard library functions.

## Code Examples

### Basic Math Operations

```c
static PF_Err
RenderPixel(
    PF_InData     *in_data,
    A_FpLong      angle_radians,
    A_FpLong      *sin_out,
    A_FpLong      *cos_out)
{
    // Using ANSI callbacks -- in_data must be in scope
    *sin_out = PF_SIN(angle_radians);
    *cos_out = PF_COS(angle_radians);
    return PF_Err_NONE;
}
```

### Distance Calculation with PF_HYPOT

```c
static A_FpLong
DistanceFromCenter(
    PF_InData   *in_data,
    A_FpLong    x,
    A_FpLong    y,
    A_FpLong    center_x,
    A_FpLong    center_y)
{
    A_FpLong dx = x - center_x;
    A_FpLong dy = y - center_y;

    // PF_HYPOT computes sqrt(dx*dx + dy*dy) without overflow risk
    return PF_HYPOT(dx, dy);
}
```

### PF_SPRINTF for Debug and UI Strings

```c
static PF_Err
FormatParamName(
    PF_InData   *in_data,
    A_long      index,
    A_FpLong    value,
    A_char      *name_buf,
    A_long      buf_size)
{
    // PF_SPRINTF works exactly like sprintf
    PF_SPRINTF(name_buf, "Channel %d (%.1f%%)", index, value * 100.0);
    return PF_Err_NONE;
}
```

### Gamma Correction Using PF_POW

```c
static A_FpLong
ApplyGamma(
    PF_InData   *in_data,
    A_FpLong    linear_value,
    A_FpLong    gamma)
{
    if (linear_value <= 0.0)
        return 0.0;

    return PF_POW(linear_value, 1.0 / gamma);
}
```

### Angle Conversion with PF_ATAN2

```c
static void
CartesianToPolar(
    PF_InData   *in_data,
    A_FpLong    x,
    A_FpLong    y,
    A_FpLong    *radius,
    A_FpLong    *angle)
{
    *radius = PF_HYPOT(x, y);
    *angle  = PF_ATAN2(y, x);  // Note: Y first, X second
}
```

### Mixing Callbacks with Standard Math

```c
#include <cmath>

static PF_Err
RenderRow(
    PF_InData       *in_data,
    PF_Pixel8       *row,
    A_long          width,
    A_FpLong        phase)
{
    // Performance-critical inner loop: use standard math
    for (A_long x = 0; x < width; x++) {
        double t = (double)x / (double)width;
        double wave = sin(t * 3.14159265 * 2.0 + phase);  // direct call, inlineable
        A_u_char val = (A_u_char)((wave * 0.5 + 0.5) * 255.0);
        row[x].red = row[x].green = row[x].blue = val;
        row[x].alpha = 255;
    }

    // Non-critical logging: PF_SPRINTF is fine
    A_char msg[64];
    PF_SPRINTF(msg, "Rendered %d pixels", width);

    return PF_Err_NONE;
}
```

## Pitfalls and Warnings

> **Pitfall: PF_SPRINTF buffer overflows.** `PF_SPRINTF` is `sprintf`, not `snprintf`. There is no length-checked variant in the ANSI callbacks. Always size your buffers conservatively, or use `snprintf` from the standard library when buffer safety matters.

> **Pitfall: Calling without in_data.** Every ANSI callback macro dereferences `in_data->utils->ansi.*`. If `in_data` is null or you are in a context where it is not provided (e.g., a standalone thread not processing a command), the macro will crash with a null pointer dereference.

> **Pitfall: PF_ATAN2 parameter order.** The macro follows the C convention: `PF_ATAN2(Y, X)`, not `(X, Y)`. Getting this wrong produces silently incorrect angles.

> **Pitfall: Thread safety with PF_SPRINTF.** `PF_SPRINTF` writes to a user-provided buffer and is safe for concurrent use from multiple threads as long as each thread uses its own destination buffer. However, do not assume the underlying implementation is reentrant on all platforms -- avoid calling ANSI callbacks from threads that do not originate from AE's rendering pipeline.

## See Also

- `AE_EffectCB.h` -- Full callback definitions
- `AE_EffectCBSuites.h` -- Suite-based alternative: `PF_ANSICallbacksSuite1`
- Color conversion callbacks (`PF_RGB_TO_HLS`, `PF_HLS_TO_RGB`, etc.) are defined immediately after the ANSI macros in the same header
