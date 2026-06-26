# Bit Depth Conversion: CONVERT8TO16 and Cross-Depth Patterns

After Effects supports three pixel bit depths: 8-bit, 16-bit, and 32-bit float. Each uses different value ranges, and converting between them correctly is essential. This document covers the SDK-provided macros, manual conversion formulas, and the critical pitfall of AE's non-standard 16-bit range.

---

## Channel Value Ranges

Defined in `AE_Effect.h`:

```cpp
#define PF_MAX_CHAN8    255
#define PF_HALF_CHAN8   128
#define PF_MAX_CHAN16   32768
#define PF_HALF_CHAN16  16384
```

32-bit float has no macro -- the range is `0.0` to `1.0`, with values outside that range representing over-bright or negative (HDR) data.

### Summary Table

| Depth | Type | Min | Half | Max | Notes |
|---|---|---|---|---|---|
| 8-bit | `A_u_char` | 0 | 128 | 255 | Standard byte range |
| 16-bit | `A_u_short` | 0 | 16384 | 32768 | NOT 32767, NOT 65535 |
| 32-bit | `PF_FpShort` (float) | 0.0 | 0.5 | 1.0 | Can exceed [0,1] for HDR |

> **CRITICAL:** AE's 16-bit range is 0-32768, not 0-65535. The maximum value is 32768 (0x8000), which fits in an unsigned short but is not the full 16-bit range. This is a constant source of bugs when porting code from other image processing libraries.

---

## Pixel Structures

```cpp
// 8-bit: components are A_u_char (0-255)
typedef struct {
    A_u_char  alpha, red, green, blue;
} PF_Pixel;       // also PF_Pixel8

// 16-bit: components are A_u_short (0-32768)
typedef struct {
    A_u_short  alpha, red, green, blue;
} PF_Pixel16;

// 32-bit float: components are PF_FpShort (float, nominally 0.0-1.0)
typedef struct {
    PF_FpShort  alpha, red, green, blue;
} PF_PixelFloat;  // also PF_Pixel32
```

Component order in all structures is **ARGB** (alpha first).

---

## SDK-Provided Macro: CONVERT8TO16

Defined in `AE_Macros.h`:

```cpp
#define CONVERT8TO16(A)  ( (((long)(A) * PF_MAX_CHAN16) + PF_HALF_CHAN8) / PF_MAX_CHAN8 )
```

Expanded:

```cpp
#define CONVERT8TO16(A)  ( (((long)(A) * 32768) + 128) / 255 )
```

This maps 0 to 0 and 255 to 32768, with rounding via the `+ 128` (half of 255, rounded down).

### Example

```cpp
A_u_char val8 = 200;
A_u_short val16 = CONVERT8TO16(val8);  // = (200 * 32768 + 128) / 255 = 25701
```

---

## All Conversion Formulas

### 8-bit to 16-bit

```cpp
// SDK macro
A_u_short val16 = CONVERT8TO16(val8);

// Equivalent formula
A_u_short val16 = (A_u_short)(((long)val8 * PF_MAX_CHAN16 + PF_HALF_CHAN8) / PF_MAX_CHAN8);
```

### 16-bit to 8-bit

No SDK macro is provided. Manual conversion:

```cpp
A_u_char val8 = (A_u_char)(((long)val16 * PF_MAX_CHAN8 + PF_HALF_CHAN16) / PF_MAX_CHAN16);
```

Expanded:

```cpp
A_u_char val8 = (A_u_char)(((long)val16 * 255 + 16384) / 32768);
```

### 8-bit to Float

```cpp
PF_FpShort valf = (PF_FpShort)val8 / (PF_FpShort)PF_MAX_CHAN8;
```

Expanded: `valf = val8 / 255.0f`

### Float to 8-bit

```cpp
// Clamp, then scale
PF_FpShort clamped = MIN(MAX(valf, 0.0f), 1.0f);
A_u_char val8 = (A_u_char)(clamped * PF_MAX_CHAN8 + 0.5f);
```

> **Note:** The `+ 0.5f` provides proper rounding. Without it, `1.0f * 255 = 255.0f` truncates correctly, but intermediate values may lose precision.

### 16-bit to Float

```cpp
PF_FpShort valf = (PF_FpShort)val16 / (PF_FpShort)PF_MAX_CHAN16;
```

Expanded: `valf = val16 / 32768.0f`

### Float to 16-bit

```cpp
PF_FpShort clamped = MIN(MAX(valf, 0.0f), 1.0f);
A_u_short val16 = (A_u_short)(clamped * PF_MAX_CHAN16 + 0.5f);
```

---

## Complete Conversion Table

| From | To | Formula |
|---|---|---|
| 8 | 16 | `(val8 * 32768 + 128) / 255` |
| 16 | 8 | `(val16 * 255 + 16384) / 32768` |
| 8 | float | `val8 / 255.0f` |
| float | 8 | `clamp(valf, 0, 1) * 255 + 0.5` |
| 16 | float | `val16 / 32768.0f` |
| float | 16 | `clamp(valf, 0, 1) * 32768 + 0.5` |

---

## Working With All Three Depths

A common pattern is to write a templated or macro-driven pixel function, or to write three separate iterate callbacks:

```cpp
// Generic conversion helpers
static inline PF_FpShort to_float_8(A_u_char v) {
    return (PF_FpShort)v / 255.0f;
}

static inline PF_FpShort to_float_16(A_u_short v) {
    return (PF_FpShort)v / 32768.0f;
}

static inline A_u_char from_float_8(PF_FpShort v) {
    v = MIN(MAX(v, 0.0f), 1.0f);
    return (A_u_char)(v * 255.0f + 0.5f);
}

static inline A_u_short from_float_16(PF_FpShort v) {
    v = MIN(MAX(v, 0.0f), 1.0f);
    return (A_u_short)(v * 32768.0f + 0.5f);
}
```

### Rendering Dispatch Pattern

```cpp
if (PF_WORLD_IS_DEEP(output)) {
    // 16-bit path
    ERR(PF_ITERATE16(0, output->height, &input, NULL, &refcon, MyFunc16, &output));
} else {
    // 8-bit path
    ERR(PF_ITERATE(0, output->height, &input, NULL, &refcon, MyFunc8, &output));
}
```

For 32-bit float, use the iterate float suite:

```cpp
AEFX_SuiteScoper<PF_IterateFloatSuite1> float_suite(
    in_data, kPFIterateFloatSuite, kPFIterateFloatSuiteVersion1);

ERR(float_suite->iterate_origin_float(in_data, 0, output->height,
    &input, NULL, NULL, &refcon, MyFuncFloat, &output));
```

---

## Common Bugs

### Bug: Assuming 16-bit Max Is 65535

```cpp
// WRONG -- will produce washed-out images in 16-bit mode
A_u_short val16 = (A_u_short)((float)val8 / 255.0f * 65535.0f);

// CORRECT
A_u_short val16 = CONVERT8TO16(val8);
```

AE uses 32768 (not 65535) as the maximum 16-bit value. Code using 65535 will produce values that are approximately twice as bright as intended, and fully opaque pixels will have alpha values around 32768 instead of the expected maximum.

### Bug: Not Clamping Float-to-Integer Conversions

```cpp
// WRONG -- negative float values wrap to large unsigned values
A_u_char val8 = (A_u_char)(valf * 255.0f);

// CORRECT
PF_FpShort clamped = MIN(MAX(valf, 0.0f), 1.0f);
A_u_char val8 = (A_u_char)(clamped * 255.0f + 0.5f);
```

32-bit float worlds can contain values outside [0,1]. Always clamp before converting to integer types, unless you intentionally want wraparound behavior.

### Bug: Integer Overflow in 16-bit Multiplication

```cpp
// WRONG -- A_u_short * A_u_short can overflow 32-bit int
A_u_short result = (val16_a * val16_b) / PF_MAX_CHAN16;

// CORRECT -- cast to A_long or A_u_long first
A_u_short result = (A_u_short)(((A_long)val16_a * val16_b + PF_HALF_CHAN16) / PF_MAX_CHAN16);
```

`32768 * 32768 = 1,073,741,824` which fits in a 32-bit unsigned int but not a signed int. Always use `A_long` or wider types for intermediate 16-bit arithmetic.

---

## Fixed-Point Coordinate Macros

The SDK also provides fixed-point conversion macros in `AE_Macros.h` for coordinate math (not pixel values):

```cpp
#define FIX2INT(X)           ((A_long)(X) >> 16)
#define INT2FIX(X)           ((A_long)(X) << 16)
#define FIX2INT_ROUND(X)     (FIX2INT((X) + 32768))
#define FIX_2_FLOAT(X)       ((A_FpLong)(X) / 65536.0)
#define FLOAT2FIX(F)         ((PF_Fixed)((F) * 65536 + (((F) < 0) ? -0.5 : 0.5)))
```

These are for `PF_Fixed` coordinates (16.16 fixed point), not for pixel channel values. Do not confuse the fixed-point 65536 scale factor with 16-bit pixel ranges.
