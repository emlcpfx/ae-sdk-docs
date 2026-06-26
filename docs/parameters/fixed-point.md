# Fixed-Point Math: PF_Fixed, FIX_2_FLOAT, FLOAT2FIX

> Comprehensive guide to fixed-point types and conversion macros in the After Effects SDK.

## Overview

The AE SDK uses **16.16 fixed-point** integers (`PF_Fixed`) for many parameter types, geometry values, and legacy APIs. Understanding fixed-point conversion is essential because angle parameters, point parameters, and fixed sliders all store values in this format.

**Key macros** (defined in `AE_Macros.h`):

| Macro | Direction | What it does |
|---|---|---|
| `FIX_2_FLOAT(X)` | fixed → double | Divides by 65536.0 |
| `FLOAT2FIX(F)` | double → fixed | Multiplies by 65536 with rounding |
| `INT2FIX(X)` | int → fixed | Left-shifts by 16 bits |
| `FIX2INT(X)` | fixed → int | Right-shifts by 16 (truncates) |
| `FIX2INT_ROUND(X)` | fixed → int | Right-shifts with proper rounding |
| `RATIO2FLOAT(R)` | A_Ratio → double | Divides `num` by `den` |

## What is PF_Fixed?

`PF_Fixed` is a **signed 32-bit integer** using 16.16 fixed-point format: 16 bits for the integer part, 16 bits for the fractional part.

```cpp
// From A.h:
typedef A_long    A_Fixed;     // A_long is int32_t

// From AE_Effect.h:
typedef A_long    PF_Fixed;    // Same type, different name
```

- **Scale factor**: 65536 (2^16)
- `A_Fixed_ONE` = `0x00010000` = 65536 = the value **1.0**
- `A_Fixed_HALF` = `0x00008000` = 32768 = the value **0.5**
- **Range**: approximately -32768.0 to +32767.99998

`PF_Fixed` and `A_Fixed` are the same underlying type — interchangeable without casting.

## Macro Definitions

From `AE_Macros.h`:

```cpp
#define FIX2INT(X)           ((A_long)(X) >> 16)
#define INT2FIX(X)           ((A_long)(X) << 16)
#define FIX2INT_ROUND(X)     (FIX2INT((X) + 32768))
#define FIX_2_FLOAT(X)       ((A_FpLong)(X) / 65536.0)
#define FLOAT2FIX(F)         ((PF_Fixed)((F) * 65536 + (((F) < 0) ? -0.5 : 0.5)))
#define RATIO2FLOAT(R)       (A_FpLong)((A_FpLong)(R).num / ((A_FpLong)(R).den))
```

**Watch the naming asymmetry**: it's `FIX_2_FLOAT` (with underscores) but `FLOAT2FIX` (without). There is no `FIX2FLOAT` or `FLOAT_2_FIX` — using the wrong name gives an "undefined macro" compiler error.

## Where Fixed-Point Values Appear

### Fixed Sliders (`PF_Param_FIX_SLIDER`)

```cpp
// Registration — pass human-readable values:
PF_ADD_FIXED("Opacity", 0, 100, 0, 100, 50, 1, 0, 0, OPACITY_ID);
// The macro converts 50 → 50 * 65536 = 3276800 internally

// At render time — convert back:
double opacity = FIX_2_FLOAT(params[OPACITY_ID]->u.fd.value);  // returns 50.0
```

### Angle Parameters (`PF_Param_ANGLE`)

Angles store **degrees** as 16.16 fixed-point. Not limited to 0–360 — can represent multiple revolutions.

```cpp
PF_ADD_ANGLE("Rotation", 45, ROTATION_ID);  // stores 45 * 65536

// At render time:
double degrees = FIX_2_FLOAT(params[ROTATION_ID]->u.ad.value);  // 45.0
double radians = degrees * PF_RAD_PER_DEGREE;
```

### Point Parameters (`PF_Param_POINT`)

Point values use fixed-point for x/y coordinates as percentages of layer dimensions.

```cpp
// Reading point values:
double center_x = FIX_2_FLOAT(params[POINT_ID]->u.td.x_value);
double center_y = FIX_2_FLOAT(params[POINT_ID]->u.td.y_value);
```

### Geometry Types

- `PF_FixedPoint` — `PF_Fixed x, y`
- `PF_FixedRect` — `PF_Fixed left, top, right, bottom`
- `PF_Matrix` — 3×3 matrix of `PF_Fixed`

Rectangle conversion helpers:

```cpp
PF_RECT_2_FIXEDRECT(rect, fixedRect);    // INT2FIX on each field
PF_FIXEDRECT_2_RECT(fixedRect, rect);    // FIX2INT_ROUND on each field
```

### Other SDK Fields

- `in_data->shutter_angle` — PF_Fixed, range 0 to 1
- `in_data->shutter_phase` — PF_Fixed, percentage offset
- `PF_RGB_Pixel`, `PF_YIQ_Pixel`, `PF_HLS_Pixel` — arrays of 3 `PF_Fixed` values

## Code Examples

### Reading fixed slider values for computation

```cpp
// From SDK's CCU.cpp:
PF_FpLong rad_xF   = FIX_2_FLOAT(params[CCU_X_RADIUS]->u.fd.value);
PF_FpLong rad_yF   = FIX_2_FLOAT(params[CCU_Y_RADIUS]->u.fd.value);
PF_FpLong center_xF = FIX_2_FLOAT(params[CCU_PT]->u.td.x_value);
PF_FpLong center_yF = FIX_2_FLOAT(params[CCU_PT]->u.td.y_value);
```

### Converting opacity to pixel channel values

```cpp
// From SDK's Transformer.cpp:
PF_FpLong opacityF = FIX_2_FLOAT(params[XFORM_LAYERBLEND]->u.fd.value);
mode.opacity   = static_cast<A_u_char>((opacityF * PF_MAX_CHAN8) + 0.5);
mode.opacitySu = static_cast<A_u_short>((opacityF * PF_MAX_CHAN16) + 0.5);
```

### Building a PF_FixedRect from float computations

```cpp
frameFiRP->top    = FLOAT2FIX(yF - y_radF);
frameFiRP->bottom = FLOAT2FIX(yF + y_radF);
frameFiRP->left   = FLOAT2FIX(xF - x_radF * par_invF);
frameFiRP->right  = FLOAT2FIX(xF + x_radF * par_invF);
```

### Setting frame rate with FLOAT2FIX (AEGP)

```cpp
ERR(suites.IOInSuite4()->AEGP_SetInSpecNativeFPS(specH,
    FLOAT2FIX(static_cast<float>(headerP->fpsT.value) / headerP->fpsT.scale)));
```

## Common Pitfalls

### 1. Overflow when multiplying fixed-point values

PF_Fixed is only 32 bits. Multiplying two fixed values naively overflows:

```cpp
// WRONG — overflows for values > ~181
PF_Fixed a = INT2FIX(200);
PF_Fixed b = INT2FIX(300);
PF_Fixed result = (a * b) >> 16;  // 32-bit overflow!

// CORRECT — promote to 64-bit
PF_Fixed result = (PF_Fixed)(((int64_t)a * (int64_t)b) >> 16);

// BETTER — just use float for math
double aF = FIX_2_FLOAT(a);
double bF = FIX_2_FLOAT(b);
PF_Fixed result = FLOAT2FIX(aF * bF);
```

### 2. Confusing PF_Fixed with PF_FpLong

`PF_FpLong` is `double`. `PF_Fixed` is `int32_t`. Completely different types.

| Parameter type | Union member | Value type | Needs conversion? |
|---|---|---|---|
| `PF_Param_FLOAT_SLIDER` | `u.fs_d.value` | `PF_FpLong` (double) | No |
| `PF_Param_FIX_SLIDER` | `u.fd.value` | `PF_Fixed` | Yes — use `FIX_2_FLOAT` |
| `PF_Param_ANGLE` | `u.ad.value` | `PF_Fixed` | Yes — use `FIX_2_FLOAT` |
| `PF_Param_POINT` | `u.td.x_value` | `PF_Fixed` | Yes — use `FIX_2_FLOAT` |
| `PF_Param_POINT_3D` | `u.point3d_d.x_value` | `PF_FpLong` | No |

### 3. FIX2INT truncates, doesn't round

```cpp
FIX2INT(FLOAT2FIX(2.9));       // returns 2, not 3
FIX2INT_ROUND(FLOAT2FIX(2.9)); // returns 3
```

Use `FIX2INT_ROUND` when you want proper rounding.

### 4. PF_ADD_FIXED already converts for you

The `PF_ADD_FIXED` macro multiplies your arguments by 65536 internally. Pass human-readable values to the macro, but remember the `.value` you read back is fixed-point:

```cpp
// These are human-readable — the macro handles conversion:
PF_ADD_FIXED("Amount", 0, 100, 0, 100, 50, ...);

// But this is fixed-point — you must convert:
double amount = FIX_2_FLOAT(params[AMOUNT_ID]->u.fd.value);
```

### 5. The macro name asymmetry

```cpp
FIX_2_FLOAT(x);  // ✓ Correct — has underscores around "2"
FLOAT2FIX(x);    // ✓ Correct — no underscores

FIX2FLOAT(x);    // ✗ Does not exist!
FLOAT_2_FIX(x);  // ✗ Does not exist!
```

## Modern Best Practice

For new plugins, **prefer `PF_ADD_FLOAT_SLIDERX` over `PF_ADD_FIXED`** — float sliders use `PF_FpLong` (double) directly, avoiding fixed-point conversion entirely.

However, you cannot avoid fixed-point completely because:

- `PF_ADD_ANGLE` still uses `PF_Fixed`
- `PF_ADD_POINT` still uses `PF_Fixed`
- Legacy code and SDK examples use fixed sliders extensively
- Some `PF_InData` fields (`shutter_angle`, `shutter_phase`) are `PF_Fixed`
- Geometry types (`PF_FixedRect`, `PF_Matrix`) use `PF_Fixed`

## SDK Header References

- **`AE_Macros.h`** — All conversion macro definitions
- **`A.h`** — `A_Fixed`, `A_FpLong`, `A_Ratio` type definitions
- **`AE_Effect.h`** — `PF_Fixed`, `PF_FixedSliderDef`, `PF_AngleDef`, `PF_PointDef`, `PF_FixedPoint`, `PF_FixedRect`, `PF_Matrix`
- **`Param_Utils.h`** — `PF_ADD_FIXED`, `PF_ADD_ANGLE`, `PF_ADD_POINT` convenience macros
