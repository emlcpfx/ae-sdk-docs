# Convolution and Kernel Operations: PF_CONVOLVE and PF_GAUSSIAN_KERNEL

After Effects provides built-in convolution and kernel generation callbacks through the `PF_UtilCallbacks` structure and the `PF_WorldTransformSuite1`. These allow you to apply spatial filters (blur, sharpen, edge detect, emboss) without writing your own convolution loops.

This document covers the `PF_KernelFlags` system, the `PF_CONVOLVE` and `PF_GAUSSIAN_KERNEL` callbacks, their limitations, and guidance on when to implement your own convolution instead.

---

## PF_KernelFlags

The `PF_KernelFlags` type (defined as `A_u_long`) is a bitmask that controls both kernel generation (`gaussian_kernel`) and convolution behavior (`convolve`). Flags are combined with bitwise OR.

### Flag Definitions

All flags are defined in `AE_EffectCB.h` (lines 112-158). The **default value** for each pair is listed first (value 0 / bit not set).

| Flag | Value | Description |
|---|---|---|
| `PF_KernelFlag_2D` | `0` | **Default.** Kernel is a 2D matrix |
| `PF_KernelFlag_1D` | `1L << 0` | Kernel is a 1D vector (horizontal or vertical) |
| `PF_KernelFlag_UNNORMALIZED` | `0` | **Default.** Kernel values are used as-is |
| `PF_KernelFlag_NORMALIZED` | `1L << 1` | Normalize kernel so volume equals covered pixel area |
| `PF_KernelFlag_CLAMP` | `0` | **Default.** Clamp output values to valid range |
| `PF_KernelFlag_NO_CLAMP` | `1L << 2` | Do not clamp output values |
| `PF_KernelFlag_USE_LONG` | `0` | **Default.** Kernel array is `A_long` values (0-255 range) |
| `PF_KernelFlag_USE_CHAR` | `1L << 3` | Kernel array is `A_u_char` values (0-255 range) |
| `PF_KernelFlag_USE_FIXED` | `1L << 4` | Kernel array is `PF_Fixed` values (0.0-1.0 in 16.16 fixed point) |
| `PF_KernelFlag_USE_UNDEFINED` | `(1L << 4) \| (1L << 3)` | Undefined -- do not use |
| `PF_KernelFlag_HORIZONTAL` | `0` | **Default.** Apply 1D kernel horizontally |
| `PF_KernelFlag_VERTICAL` | `1L << 5` | Apply 1D kernel vertically |
| `PF_KernelFlag_TRANSPARENT_BORDERS` | `0` | **Default.** Treat out-of-bounds pixels as alpha zero (transparent black) |
| `PF_KernelFlag_REPLICATE_BORDERS` | `1L << 6` | Replicate edge pixels for out-of-bounds samples |
| `PF_KernelFlag_STRAIGHT_CONVOLVE` | `0` | **Default.** Standard convolution |
| `PF_KernelFlag_ALPHA_WEIGHT_CONVOLVE` | `1L << 7` | Alpha-weight pixel contributions during convolution |

### Unimplemented Flags (Critical Warning)

The SDK header contains explicit warnings that several flags are **not implemented**:

> **`PF_KernelFlag_USE_CHAR`**: The header states: "NOTE! For now, only USE_LONG is implemented!" This means `USE_CHAR` and `USE_FIXED` are non-functional. You **must** use `A_long` arrays for your kernels.

> **`PF_KernelFlag_REPLICATE_BORDERS`**: The header states: "NOTE! The replicate borders flag is unimplemented and this will be ignored." Out-of-bounds samples always use transparent borders.

> **`PF_KernelFlag_ALPHA_WEIGHT_CONVOLVE`**: The header states: "NOTE! The alpha weighted convolve is not implemented and this will be ignored." Convolution is always straight (non-alpha-weighted).

### Summary of What Actually Works

| Flag | Status |
|---|---|
| `PF_KernelFlag_2D` / `_1D` | Implemented |
| `PF_KernelFlag_NORMALIZED` / `_UNNORMALIZED` | Implemented |
| `PF_KernelFlag_CLAMP` / `_NO_CLAMP` | Implemented |
| `PF_KernelFlag_USE_LONG` | **Only implemented data type** |
| `PF_KernelFlag_USE_CHAR` | NOT implemented |
| `PF_KernelFlag_USE_FIXED` | NOT implemented |
| `PF_KernelFlag_HORIZONTAL` / `_VERTICAL` | Implemented (1D only) |
| `PF_KernelFlag_REPLICATE_BORDERS` | NOT implemented |
| `PF_KernelFlag_ALPHA_WEIGHT_CONVOLVE` | NOT implemented |

---

## PF_CONVOLVE

Convolves an image with per-channel kernels of arbitrary size.

### Macro

```c
#define PF_CONVOLVE(SRC, RCT_P, FLAGS, KRNL_SZ, AK, RK, GK, BK, DST)
```

This expands to:

```c
(*in_data->utils->convolve)(
    in_data->effect_ref, (SRC), (RCT_P), (FLAGS), (KRNL_SZ),
    (AK), (RK), (GK), (BK), (DST))
```

### Suite Function (PF_WorldTransformSuite1)

```c
PF_Err (*convolve)(
    PF_ProgPtr      effect_ref,
    PF_EffectWorld  *src,
    const PF_Rect   *area,          /* pass NULL for all pixels */
    PF_KernelFlags  flags,
    A_long          kernel_size,
    void            *a_kernel,      /* alpha channel kernel */
    void            *r_kernel,      /* red channel kernel */
    void            *g_kernel,      /* green channel kernel */
    void            *b_kernel,      /* blue channel kernel */
    PF_EffectWorld  *dst);
```

### Parameters

| Parameter | Description |
|---|---|
| `src` | Source world to read from |
| `area` | Rectangle to convolve within. Pass `NULL` for the entire image. Use `extent_hint` for efficiency. |
| `flags` | Bitwise OR of `PF_KernelFlags` values |
| `kernel_size` | For 2D kernels: the side length (kernel is `kernel_size x kernel_size`). For 1D kernels: the length of the vector. Must be odd. |
| `a_kernel` | Pointer to kernel array for the alpha channel. Pass `NULL` to leave alpha unchanged. |
| `r_kernel` | Pointer to kernel array for the red channel |
| `g_kernel` | Pointer to kernel array for the green channel |
| `b_kernel` | Pointer to kernel array for the blue channel |
| `dst` | Destination world to write to |

### Kernel Layout

Since only `PF_KernelFlag_USE_LONG` is implemented, all kernel arrays are `A_long*`.

**For a 2D kernel** of size N:
- The array has `N * N` elements
- Elements are in row-major order
- The center of the kernel is at index `(N/2) * N + (N/2)` (integer division)

**For a 1D kernel** of size N:
- The array has `N` elements
- The center of the kernel is at index `N/2`

### Per-Channel Kernels

You can apply different kernels to each channel, or pass the same kernel pointer for all channels. This enables effects like:
- Chromatic aberration (different blur radii per channel)
- Channel-specific sharpening
- Leaving alpha untouched while processing RGB

### Code Example: 3x3 Sharpen

```c
static PF_Err ApplySharpen(PF_InData *in_data, PF_OutData *out_data,
                            PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;

    // 3x3 sharpen kernel (values are A_long)
    // The kernel sums to 1 (identity) + sharpening contribution
    A_long sharpen_kernel[9] = {
         0, -1,  0,
        -1,  5, -1,
         0, -1,  0
    };

    PF_KernelFlags flags = PF_KernelFlag_2D |
                           PF_KernelFlag_USE_LONG |
                           PF_KernelFlag_CLAMP;

    err = PF_CONVOLVE(&params[0]->u.ld,     // source
                      &output->extent_hint,  // area
                      flags,
                      3,                     // 3x3 kernel
                      NULL,                  // alpha: leave unchanged
                      sharpen_kernel,        // R
                      sharpen_kernel,        // G
                      sharpen_kernel,        // B
                      output);               // destination
    return err;
}
```

### Code Example: 1D Horizontal Edge Detect

```c
static PF_Err HorizontalEdge(PF_InData *in_data, PF_OutData *out_data,
                              PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;

    // Sobel horizontal edge detection kernel
    A_long edge_kernel[3] = { -1, 0, 1 };

    PF_KernelFlags flags = PF_KernelFlag_1D |
                           PF_KernelFlag_HORIZONTAL |
                           PF_KernelFlag_USE_LONG |
                           PF_KernelFlag_NO_CLAMP;

    err = PF_CONVOLVE(&params[0]->u.ld,
                      &output->extent_hint,
                      flags,
                      3,                    // 3 elements
                      NULL,                 // skip alpha
                      edge_kernel,
                      edge_kernel,
                      edge_kernel,
                      output);
    return err;
}
```

---

## PF_GAUSSIAN_KERNEL

Generates a Gaussian-distribution kernel that you can then pass to `PF_CONVOLVE`. This saves you from computing Gaussian weights manually.

### Macro

```c
#define PF_GAUSSIAN_KERNEL(K_RAD, FLAGS, MULT, DIAM, KERNEL)
```

This expands to:

```c
(*in_data->utils->gaussian_kernel)(
    in_data->effect_ref, (K_RAD), (FLAGS), (MULT), (DIAM), (KERNEL))
```

### Suite Function (from PF_UtilCallbacks)

```c
PF_Err (*gaussian_kernel)(
    PF_ProgPtr      effect_ref,
    A_FpLong        kRadius,        /* desired gaussian radius */
    PF_KernelFlags  flags,          /* see kernel flags above */
    A_FpLong        multiplier,     /* multiplied into every kernel value */
    A_long          *diameter,      /* << OUT: actual kernel diameter */
    void            *kernel);       /* << OUT: pre-allocated kernel buffer */
```

### Parameters

| Parameter | Description |
|---|---|
| `kRadius` | The Gaussian radius (standard deviation proxy). Larger values = wider/blurrier kernel. |
| `flags` | Relevant flags: `_1D`/`_2D`, `_NORMALIZED`/`_UNNORMALIZED`, `_USE_LONG`/`_USE_CHAR`/`_USE_FIXED` |
| `multiplier` | Scalar multiplied into every generated value. Pass `1.0` for standard Gaussian. |
| `diameter` | **Output**: the actual integer diameter of the generated kernel. Always `ceil(kRadius) * 2 + 1`. |
| `kernel` | **Pre-allocated** buffer. You must allocate this before calling. |

### Buffer Sizing

The kernel diameter is always:

```
diameter = (int)ceil(kRadius) * 2 + 1
```

For a **1D kernel**, allocate `diameter` elements.
For a **2D kernel**, allocate `diameter * diameter` elements.

Since only `USE_LONG` is reliably implemented, allocate `A_long` arrays:

```c
A_long diameter = (A_long)ceil(radius) * 2 + 1;

// For 1D:
A_long *kernel1D = (A_long *)malloc(diameter * sizeof(A_long));

// For 2D:
A_long *kernel2D = (A_long *)malloc(diameter * diameter * sizeof(A_long));
```

### Code Example: Generate and Apply a Gaussian Blur

```c
static PF_Err GaussianBlur(PF_InData *in_data, PF_OutData *out_data,
                            PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;
    PF_FpLong radius = params[BLUR_RADIUS]->u.fd.value;

    if (radius < 0.5) {
        // No blur needed, just copy
        err = PF_COPY(&params[0]->u.ld, output, NULL, NULL);
        return err;
    }

    // Calculate diameter
    A_long expected_diam = (A_long)ceil(radius) * 2 + 1;

    // --- Separable 2-pass approach (1D horizontal + 1D vertical) ---
    // This is O(N) per pixel instead of O(N^2) for a 2D kernel.

    // Allocate kernel buffer
    A_long *kernel = (A_long *)calloc(expected_diam, sizeof(A_long));
    if (!kernel) return PF_Err_OUT_OF_MEMORY;

    A_long actual_diam = 0;

    // Generate 1D normalized Gaussian kernel
    PF_KernelFlags gen_flags = PF_KernelFlag_1D |
                               PF_KernelFlag_NORMALIZED |
                               PF_KernelFlag_USE_LONG;

    err = PF_GAUSSIAN_KERNEL(radius, gen_flags, 1.0,
                             &actual_diam, kernel);
    if (err) { free(kernel); return err; }

    // We need a temp world for the intermediate pass
    PF_EffectWorld temp_world;
    err = PF_NEW_WORLD(output->width, output->height,
                       PF_NewWorldFlag_CLEAR_PIXELS, &temp_world);
    if (err) { free(kernel); return err; }

    // Pass 1: Horizontal convolution (src -> temp)
    PF_KernelFlags conv_flags_h = PF_KernelFlag_1D |
                                  PF_KernelFlag_HORIZONTAL |
                                  PF_KernelFlag_USE_LONG |
                                  PF_KernelFlag_CLAMP;

    err = PF_CONVOLVE(&params[0]->u.ld,
                      &output->extent_hint,
                      conv_flags_h,
                      actual_diam,
                      kernel,       // A
                      kernel,       // R
                      kernel,       // G
                      kernel,       // B
                      &temp_world);
    if (err) goto cleanup;

    // Pass 2: Vertical convolution (temp -> output)
    PF_KernelFlags conv_flags_v = PF_KernelFlag_1D |
                                  PF_KernelFlag_VERTICAL |
                                  PF_KernelFlag_USE_LONG |
                                  PF_KernelFlag_CLAMP;

    err = PF_CONVOLVE(&temp_world,
                      &output->extent_hint,
                      conv_flags_v,
                      actual_diam,
                      kernel,
                      kernel,
                      kernel,
                      kernel,
                      output);

cleanup:
    PF_DISPOSE_WORLD(&temp_world);
    free(kernel);
    return err;
}
```

---

## Separable Convolution: The Recommended Approach

A 2D Gaussian convolution of radius R has complexity O(D^2) per pixel, where D is the kernel diameter. For large radii this becomes very expensive.

Because the Gaussian kernel is **separable**, you can decompose a 2D convolution into two 1D passes (horizontal then vertical) with complexity O(2D) per pixel. This is dramatically faster for large kernels.

```
2D convolution of 21x21 kernel: 441 multiplies per pixel
Separable 1D + 1D of 21 kernel: 42 multiplies per pixel (10.5x faster)
```

The code example above demonstrates this approach.

---

## Limitations of Built-In Convolution

The built-in `PF_CONVOLVE` has several limitations that may force you to implement your own:

### 1. Only USE_LONG Kernel Data

Only `A_long` kernel arrays are supported. If you need floating-point kernel weights for precision, you must implement your own convolution.

### 2. No Border Replication

`PF_KernelFlag_REPLICATE_BORDERS` is unimplemented. Out-of-bounds pixels are always treated as transparent black. For effects where edge darkening is unacceptable, you must handle borders yourself.

### 3. No Alpha-Weighted Convolution

`PF_KernelFlag_ALPHA_WEIGHT_CONVOLVE` is unimplemented. In premultiplied images, convolving without alpha weighting can cause semi-transparent edges to darken or exhibit color fringing. If your effect requires proper premultiplication-aware convolution, implement your own.

### 4. 8-Bit Only

The built-in convolve operates on the pixel data as-is. For 16-bit and 32-bit float worlds, the behavior is undocumented and unreliable. For deep color support, implement your own convolution using the iterate suites.

### 5. No Progress Reporting

Unlike iterate functions, `PF_CONVOLVE` does not provide a progress callback parameter. For very large images with large kernels, the user sees no progress feedback.

### 6. Fixed Kernel Size

The kernel size must be determined before calling. For effects that need adaptive kernel sizes (varying by position), you need a custom implementation.

---

## When To Implement Your Own Convolution

Use the built-in `PF_CONVOLVE` when:
- You need a simple, small-radius convolution (sharpen, emboss, simple edge detect)
- You are working in 8-bit mode
- Edge handling (transparent borders) is acceptable
- You do not need progress reporting

Implement your own when:
- You need 16-bit or 32-bit float precision
- You need replicated or wrapped borders
- You need alpha-weighted convolution for premultiplied content
- You need adaptive or spatially-varying kernels
- You need progress reporting for large kernels
- You need floating-point kernel weights
- Performance is critical and you want to use SIMD, GPU, or more advanced algorithms (FFT-based, recursive Gaussian, etc.)

### Custom Convolution Pattern

```c
typedef struct {
    PF_EffectWorld  *src;
    A_long          kernel_radius;
    PF_FpLong       *kernel_weights;  // float weights
    A_long          src_width;
    A_long          src_height;
} ConvolveInfo;

static PF_Err ConvolvePixel8(
    void        *refcon,
    A_long      x,
    A_long      y,
    PF_Pixel    *inP,
    PF_Pixel    *outP)
{
    ConvolveInfo *ci = (ConvolveInfo *)refcon;
    PF_FpLong sumR = 0, sumG = 0, sumB = 0, sumA = 0;
    A_long radius = ci->kernel_radius;

    for (A_long ky = -radius; ky <= radius; ky++) {
        for (A_long kx = -radius; kx <= radius; kx++) {
            A_long sx = x + kx;
            A_long sy = y + ky;

            PF_Pixel sample = {0, 0, 0, 0};

            // Replicate borders (what the built-in cannot do)
            sx = MAX(0, MIN(sx, ci->src_width - 1));
            sy = MAX(0, MIN(sy, ci->src_height - 1));

            PF_Pixel *row = (PF_Pixel *)((char *)ci->src->data +
                                          sy * ci->src->rowbytes);
            sample = row[sx];

            PF_FpLong w = ci->kernel_weights[(ky + radius) * (2 * radius + 1) +
                                              (kx + radius)];
            sumA += sample.alpha * w;
            sumR += sample.red   * w;
            sumG += sample.green * w;
            sumB += sample.blue  * w;
        }
    }

    outP->alpha = (A_u_char)MAX(0, MIN(255, (A_long)(sumA + 0.5)));
    outP->red   = (A_u_char)MAX(0, MIN(255, (A_long)(sumR + 0.5)));
    outP->green = (A_u_char)MAX(0, MIN(255, (A_long)(sumG + 0.5)));
    outP->blue  = (A_u_char)MAX(0, MIN(255, (A_long)(sumB + 0.5)));

    return PF_Err_NONE;
}
```

---

## PF_Kernel Struct

The SDK does not define a formal `PF_Kernel` struct. Kernels are simply raw arrays of `A_long` (or theoretically `A_u_char` or `PF_Fixed`, but those types are unimplemented). The kernel pointer is passed as `void*` and cast internally based on the `PF_KernelFlags`.

---

## Quick Reference: Flag Combinations

### Gaussian Blur (separable)

```c
// Generation flags
PF_KernelFlag_1D | PF_KernelFlag_NORMALIZED | PF_KernelFlag_USE_LONG

// Horizontal pass
PF_KernelFlag_1D | PF_KernelFlag_HORIZONTAL | PF_KernelFlag_USE_LONG | PF_KernelFlag_CLAMP

// Vertical pass
PF_KernelFlag_1D | PF_KernelFlag_VERTICAL | PF_KernelFlag_USE_LONG | PF_KernelFlag_CLAMP
```

### Sharpen (2D, 3x3)

```c
PF_KernelFlag_2D | PF_KernelFlag_USE_LONG | PF_KernelFlag_CLAMP
```

### Edge Detection (no clamping for negative values)

```c
PF_KernelFlag_2D | PF_KernelFlag_USE_LONG | PF_KernelFlag_NO_CLAMP
```

---

## Common Pitfalls

### 1. Forgetting to pre-allocate the kernel buffer

`PF_GAUSSIAN_KERNEL` writes into a buffer you provide. If you pass an unallocated or undersized pointer, you corrupt memory. Always compute `ceil(radius) * 2 + 1` and allocate accordingly.

### 2. Using USE_CHAR or USE_FIXED

These flags compile without error but are silently ignored at runtime. Your kernel data will be interpreted as `A_long` regardless. This can produce wildly incorrect results if you actually filled the buffer with `A_u_char` or `PF_Fixed` values.

### 3. Even-sized kernels

The kernel size should always be odd so that the kernel has a clear center pixel. Passing even sizes may produce offset or incorrect results.

### 4. Expecting border replication to work

Despite the flag existing, `PF_KernelFlag_REPLICATE_BORDERS` does nothing. If you need edge replication, pad your source buffer manually before convolving or implement your own convolution.

### 5. Applying 2D convolution for separable kernels

Using a full 2D kernel for a Gaussian blur is correct but unnecessarily slow. Always use the separable (two-pass 1D) approach when the kernel is separable.
