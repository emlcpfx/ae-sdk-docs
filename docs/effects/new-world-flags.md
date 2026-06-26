# World Creation and Query Flags

The After Effects SDK has two separate flag enums related to `PF_EffectWorld` objects: **`PF_NewWorldFlags`** for creating worlds, and **`PF_WorldFlags`** for querying the properties of existing worlds. These are **different types** used in **different contexts**, and confusing them is a common source of bugs.

---

## The Two Enums at a Glance

| Enum | Defined In | Used For | Type |
|---|---|---|---|
| `PF_NewWorldFlags` | `AE_EffectCB.h` | Creating new worlds (`PF_NEW_WORLD`, `PF_WorldSuite1::new_world`) | `A_long` |
| `PF_WorldFlags` | `AE_Effect.h` | Querying existing worlds (reading `world->world_flags`) | `A_long` |

These enums have **different values** for conceptually similar concepts. `PF_NewWorldFlag_DEEP_PIXELS` is `(1L << 1)` = `2`, while `PF_WorldFlag_DEEP` is `(1L << 0)` = `1`. Passing one where the other is expected will silently set the wrong bits.

---

## PF_NewWorldFlags (Creation Flags)

Used when creating temporary or scratch worlds via `PF_NEW_WORLD` (the macro) or the `PF_WorldSuite1::new_world` suite function.

### Definition (from AE_EffectCB.h)

```c
enum {
    PF_NewWorldFlag_NONE            = 0,
    PF_NewWorldFlag_CLEAR_PIXELS    = 1L << 0,    /* clear pixels to zero on creation */
    PF_NewWorldFlag_DEEP_PIXELS     = 1L << 1,    /* create deep (16-bit) world */
    PF_NewWorldFlag_RESERVED0       = 1L << 2,    /* internal use */
    PF_NewWorldFlag_RESERVED1       = 1L << 3
};
typedef A_long PF_NewWorldFlags;
```

### Flag Details

| Flag | Value | Description |
|---|---|---|
| `PF_NewWorldFlag_NONE` | `0` | No special behavior. Pixels contain uninitialized memory. |
| `PF_NewWorldFlag_CLEAR_PIXELS` | `1` | Zero-fill all pixel data on creation. Essential for compositing operations where unwritten pixels must be transparent black. |
| `PF_NewWorldFlag_DEEP_PIXELS` | `2` | Create a 16-bit-per-channel world (`PF_Pixel16`). Without this flag, the world is 8-bit (`PF_Pixel8`). |

### Usage: PF_NEW_WORLD Macro

```c
#define PF_NEW_WORLD(WIDTH, HEIGHT, FLAGS, WORLD) \
    (*in_data->utils->new_world)( \
        in_data->effect_ref, (WIDTH), (HEIGHT), (FLAGS), (WORLD))
```

### Usage: PF_WorldSuite1

```c
typedef struct PF_WorldSuite1 {
    PF_Err (*new_world)(
        PF_ProgPtr          effect_ref,
        A_long              width,
        A_long              height,
        PF_NewWorldFlags    flags,
        PF_EffectWorld      *world);

    PF_Err (*dispose_world)(
        PF_ProgPtr          effect_ref,
        PF_EffectWorld      *world);
} PF_WorldSuite1;
```

### Code Example: Creating Temp Worlds

```c
// 8-bit temp world, cleared to transparent black
PF_EffectWorld temp8;
err = PF_NEW_WORLD(width, height,
                   PF_NewWorldFlag_CLEAR_PIXELS,
                   &temp8);

// 16-bit temp world, cleared
PF_EffectWorld temp16;
err = PF_NEW_WORLD(width, height,
                   PF_NewWorldFlag_CLEAR_PIXELS | PF_NewWorldFlag_DEEP_PIXELS,
                   &temp16);

// 8-bit temp world, NOT cleared (faster if you will write every pixel)
PF_EffectWorld temp_fast;
err = PF_NEW_WORLD(width, height,
                   PF_NewWorldFlag_NONE,
                   &temp_fast);

// Always dispose when done
PF_DISPOSE_WORLD(&temp8);
PF_DISPOSE_WORLD(&temp16);
PF_DISPOSE_WORLD(&temp_fast);
```

---

## PF_WorldFlags (Query Flags)

Used to inspect the properties of an **existing** `PF_EffectWorld`, such as the input layer, output layer, or any world checked out via SmartFX.

### Definition (from AE_Effect.h)

```c
enum {
    PF_WorldFlag_DEEP           = 1L << 0,
    PF_WorldFlag_WRITEABLE      = 1L << 1,

    PF_WorldFlag_RESERVED0      = 1L << 24,
    PF_WorldFlag_RESERVED1      = 1L << 25,
    PF_WorldFlag_RESERVED2      = 1L << 26,
    PF_WorldFlag_RESERVED3      = 1L << 27,
    PF_WorldFlag_RESERVED4      = 1L << 28,
    PF_WorldFlag_RESERVED5      = 1L << 29,
    PF_WorldFlag_RESERVED6      = 1L << 30,
    PF_WorldFlag_RESERVED       = 1L << 31
};
typedef A_long PF_WorldFlags;
```

### Flag Details

| Flag | Value | Description |
|---|---|---|
| `PF_WorldFlag_DEEP` | `1` | World contains 16-bit-per-channel pixels (or higher). |
| `PF_WorldFlag_WRITEABLE` | `2` | World's pixel data can be written to. Source/input layers are typically NOT writeable. |

### Where WorldFlags Live

The `world_flags` field is part of the `PF_LayerDef` (aka `PF_EffectWorld`) struct:

```c
typedef struct PF_LayerDef {
    void            *reserved0;
    void            *reserved1;
    PF_WorldFlags   world_flags;    // <-- HERE
    PF_PixelPtr     data;
    A_long          rowbytes;
    A_long          width;
    A_long          height;
    PF_UnionableRect extent_hint;
    // ... more fields ...
} PF_LayerDef;

typedef PF_LayerDef PF_EffectWorld;
```

### PF_WORLD_IS_DEEP Macro

The SDK provides a convenience macro for the most common query:

```c
#define PF_WORLD_IS_DEEP(W)  ( ((W)->world_flags & PF_WorldFlag_DEEP) != 0 )
```

This checks whether a world has 16-bit (or higher) pixel data.

### Code Example: Checking World Properties

```c
static PF_Err Render(PF_InData *in_data, PF_OutData *out_data,
                     PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;
    PF_EffectWorld *input = &params[0]->u.ld;

    // Check if the project is in 16-bit mode
    if (PF_WORLD_IS_DEEP(output)) {
        // Use 16-bit pixel processing
        err = Process16bit(in_data, input, output);
    } else {
        // Use 8-bit pixel processing
        err = Process8bit(in_data, input, output);
    }

    // Check if the input world is writeable
    if (input->world_flags & PF_WorldFlag_WRITEABLE) {
        // Safe to modify input pixels in-place
        // (uncommon -- most source worlds are read-only)
    }

    return err;
}
```

---

## CRITICAL: Why These Enums Are Different

The bit positions do not align:

| Concept | PF_NewWorldFlags (creation) | PF_WorldFlags (query) |
|---|---|---|
| Deep / 16-bit | `PF_NewWorldFlag_DEEP_PIXELS` = `1L << 1` = **2** | `PF_WorldFlag_DEEP` = `1L << 0` = **1** |
| Clear pixels | `PF_NewWorldFlag_CLEAR_PIXELS` = `1L << 0` = **1** | (no equivalent) |
| Writeable | (no equivalent) | `PF_WorldFlag_WRITEABLE` = `1L << 1` = **2** |

### The Dangerous Mistake

```c
// WRONG: Using PF_WorldFlag_DEEP (value 1) as a creation flag
// This actually sets PF_NewWorldFlag_CLEAR_PIXELS, NOT deep pixels!
PF_EffectWorld temp;
err = PF_NEW_WORLD(w, h, PF_WorldFlag_DEEP, &temp);  // BUG!

// WRONG: Using PF_NewWorldFlag_DEEP_PIXELS (value 2) to check depth
// This actually checks PF_WorldFlag_WRITEABLE, NOT depth!
if (world->world_flags & PF_NewWorldFlag_DEEP_PIXELS) { ... }  // BUG!
```

Both of these compile without any warning because both types are `A_long`. The compiler cannot catch the type mismatch.

### The Correct Way

```c
// CORRECT: Use PF_NewWorldFlags for creation
err = PF_NEW_WORLD(w, h, PF_NewWorldFlag_DEEP_PIXELS, &temp);

// CORRECT: Use PF_WorldFlags (or the macro) for querying
if (PF_WORLD_IS_DEEP(world)) { ... }
// or equivalently:
if (world->world_flags & PF_WorldFlag_DEEP) { ... }
```

---

## PF_WorldSuite2: The Explicit Pixel Format Approach

`PF_WorldSuite2` (frozen in AE 7.0) takes a different approach to world creation. Instead of a flags enum, it accepts an explicit `PF_PixelFormat` value, eliminating the ambiguity entirely.

### Suite Definition

```c
#define kPFWorldSuite           "PF World Suite"
#define kPFWorldSuiteVersion2   2

typedef struct PF_WorldSuite2 {
    PF_Err (*PF_NewWorld)(
        PF_ProgPtr      effect_ref,
        A_long          widthL,
        A_long          heightL,
        PF_Boolean      clear_pixB,         /* TRUE to zero-fill */
        PF_PixelFormat  pixel_format,       /* explicit format */
        PF_EffectWorld  *worldP);

    PF_Err (*PF_DisposeWorld)(
        PF_ProgPtr      effect_ref,
        PF_EffectWorld  *worldP);

    PF_Err (*PF_GetPixelFormat)(
        const PF_EffectWorld    *worldP,
        PF_PixelFormat          *pixel_formatP);
} PF_WorldSuite2;
```

### PF_PixelFormat Values (from AE_EffectPixelFormat.h)

| Format | Description |
|---|---|
| `PF_PixelFormat_ARGB32` | Standard 8-bit ARGB (0-255 per channel) |
| `PF_PixelFormat_ARGB64` | 16-bit ARGB (0-32768 per channel) |
| `PF_PixelFormat_ARGB128` | 32-bit float ARGB (1.0 = white) |
| `PF_PixelFormat_GPU_BGRA128` | GPU float BGRA (for GPU rendering) |

### Advantages of PF_WorldSuite2

1. **Supports 32-bit float worlds** -- `PF_NEW_WORLD` with `PF_NewWorldFlags` can only create 8-bit or 16-bit worlds. There is no `PF_NewWorldFlag_FLOAT_PIXELS`. `PF_WorldSuite2` is the only way to create 32-bit float scratch worlds.

2. **Explicit format** -- No bit flag confusion. You specify exactly what you want.

3. **Format querying** -- `PF_GetPixelFormat` tells you the exact format of any world, which is more informative than the binary deep/not-deep check of `PF_WORLD_IS_DEEP`.

### Code Example: Creating a Float World with PF_WorldSuite2

```c
static PF_Err CreateFloatWorld(PF_InData *in_data, PF_OutData *out_data,
                                A_long width, A_long height,
                                PF_EffectWorld *worldP)
{
    PF_Err err = PF_Err_NONE;

    AEFX_SuiteScoper<PF_WorldSuite2> worldSuite(
        in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);

    err = worldSuite->PF_NewWorld(
        in_data->effect_ref,
        width,
        height,
        TRUE,                       // clear pixels
        PF_PixelFormat_ARGB128,     // 32-bit float
        worldP);

    return err;
}
```

### Code Example: Matching the Input World's Format

```c
static PF_Err CreateMatchingWorld(PF_InData *in_data, PF_OutData *out_data,
                                   PF_EffectWorld *referenceP,
                                   A_long width, A_long height,
                                   PF_EffectWorld *newWorldP)
{
    PF_Err err = PF_Err_NONE;

    AEFX_SuiteScoper<PF_WorldSuite2> worldSuite(
        in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);

    // Query the format of the reference world
    PF_PixelFormat format = PF_PixelFormat_ARGB32;
    err = worldSuite->PF_GetPixelFormat(referenceP, &format);
    if (err) return err;

    // Create a new world with the same format
    err = worldSuite->PF_NewWorld(
        in_data->effect_ref,
        width, height,
        TRUE,       // clear
        format,
        newWorldP);

    return err;
}
```

---

## Relationship: PF_NewWorldFlag_DEEP_PIXELS and PF_WorldSuite2

When you create a world with `PF_NewWorldFlag_DEEP_PIXELS` via the macro or `PF_WorldSuite1`:
- The world is created as `PF_PixelFormat_ARGB64` (16-bit)
- Its `world_flags` field will have `PF_WorldFlag_DEEP` set

When you create a world with `PF_WorldSuite2::PF_NewWorld` and `PF_PixelFormat_ARGB128`:
- The world is 32-bit float
- Its `world_flags` field will **also** have `PF_WorldFlag_DEEP` set
- `PF_WORLD_IS_DEEP()` returns TRUE for **both** 16-bit and 32-bit float worlds

This means `PF_WORLD_IS_DEEP` alone cannot distinguish 16-bit from 32-bit float. For that distinction, use `PF_WorldSuite2::PF_GetPixelFormat`:

```c
PF_PixelFormat format;
worldSuite->PF_GetPixelFormat(worldP, &format);

switch (format) {
    case PF_PixelFormat_ARGB32:
        // 8-bit processing
        break;
    case PF_PixelFormat_ARGB64:
        // 16-bit processing
        break;
    case PF_PixelFormat_ARGB128:
        // 32-bit float processing
        break;
}
```

---

## Complete Decision Guide

### Creating a Temp World

```
Need 32-bit float world?
    YES --> Must use PF_WorldSuite2 with PF_PixelFormat_ARGB128
    NO  --> Need 16-bit world?
                YES --> PF_NEW_WORLD with PF_NewWorldFlag_DEEP_PIXELS
                        or PF_WorldSuite2 with PF_PixelFormat_ARGB64
                NO  --> PF_NEW_WORLD with PF_NewWorldFlag_NONE (8-bit)

Need cleared pixels?
    YES --> Add PF_NewWorldFlag_CLEAR_PIXELS (macro)
            or pass TRUE for clear_pixB (suite)
    NO  --> Omit the flag / pass FALSE (uninitialized, slightly faster)
```

### Matching the Project Bit Depth

A common pattern is to create temp worlds that match the current project bit depth:

```c
PF_NewWorldFlags flags = PF_NewWorldFlag_CLEAR_PIXELS;

if (PF_WORLD_IS_DEEP(output)) {
    // Project is in 16-bit or 32-bit mode
    // For a simple approach using the macro:
    flags |= PF_NewWorldFlag_DEEP_PIXELS;

    // For precise format matching, use PF_WorldSuite2 instead
}

PF_EffectWorld temp;
err = PF_NEW_WORLD(output->width, output->height, flags, &temp);
```

> **Best practice**: Use `PF_WorldSuite2` with `PF_GetPixelFormat` to create temp worlds that exactly match the input format, especially in effects that support all three bit depths.

---

## Common Pitfalls

### 1. Mixing up PF_NewWorldFlags and PF_WorldFlags

The most dangerous bug. Both are `A_long` so the compiler will not warn you. Always use `PF_NewWorldFlag_*` with `PF_NEW_WORLD` and `PF_WorldFlag_*` with `world_flags`.

### 2. Forgetting to clear pixels

If you create a world without `PF_NewWorldFlag_CLEAR_PIXELS` and do not write every pixel, you will have random garbage in unwritten areas. This is especially visible when compositing, as uninitialized alpha values cause unpredictable transparency.

### 3. Assuming PF_WORLD_IS_DEEP means exactly 16-bit

`PF_WORLD_IS_DEEP` returns true for both 16-bit and 32-bit float worlds. If your 16-bit code path uses `PF_Pixel16` on a float world, you will read corrupted data. Always use `PF_GetPixelFormat` for disambiguation when supporting all three depths.

### 4. Forgetting to dispose temp worlds

Every `PF_NEW_WORLD` or `PF_WorldSuite2::PF_NewWorld` call **must** have a corresponding `PF_DISPOSE_WORLD` or `PF_WorldSuite2::PF_DisposeWorld`. Leaking worlds causes memory exhaustion, especially in long renders.

### 5. Creating float worlds with the macro

`PF_NEW_WORLD` cannot create 32-bit float worlds. There is no `PF_NewWorldFlag_FLOAT_PIXELS`. You must use `PF_WorldSuite2` for float scratch buffers.

### 6. Checking writeability before modifying

Source/input worlds are not necessarily writeable. The `PF_WorldFlag_WRITEABLE` flag tells you whether it is safe to write. The output world is always writeable. Checked-out layers from SmartFX may or may not be, depending on how they were obtained.

---

## Summary

| Task | API | Flags/Values |
|---|---|---|
| Create 8-bit world | `PF_NEW_WORLD` | `PF_NewWorldFlag_NONE` or `PF_NewWorldFlag_CLEAR_PIXELS` |
| Create 16-bit world | `PF_NEW_WORLD` | `PF_NewWorldFlag_DEEP_PIXELS` |
| Create 32-bit float world | `PF_WorldSuite2::PF_NewWorld` | `PF_PixelFormat_ARGB128` |
| Check if world is deep | `PF_WORLD_IS_DEEP(w)` | Tests `PF_WorldFlag_DEEP` |
| Check exact pixel format | `PF_WorldSuite2::PF_GetPixelFormat` | Returns `PF_PixelFormat` enum |
| Check if world is writeable | `w->world_flags & PF_WorldFlag_WRITEABLE` | Tests bit 1 |
| Dispose a world | `PF_DISPOSE_WORLD` | N/A |
