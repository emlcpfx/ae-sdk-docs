# Compositing Operations Documentation

This document describes how to use the After Effects SDK for compositing operations, with specific focus on avoiding black outlines/halos that commonly occur with alpha-aware operations.

## Table of Contents
1. [The Black Edge Problem](#the-black-edge-problem)
2. [Core Compositing Strategy](#core-compositing-strategy)
3. [SDK Suites Used](#sdk-suites-used)
4. [Premultiplied Alpha Handling](#premultiplied-alpha-handling)
5. [Layer Stacking with BEHIND Mode](#layer-stacking-with-behind-mode)
6. [Final Compositing with IN_FRONT Mode](#final-compositing-with-in_front-mode)
7. [The "Photoshop Mode" Approach](#the-photoshop-mode-approach)
8. [Alpha Expansion for Edge Extension](#alpha-expansion-for-edge-extension)
9. [Keep Original Alpha Mode](#keep-original-alpha-mode)
10. [Common Pitfalls and Solutions](#common-pitfalls-and-solutions)

---

## The Black Edge Problem

When compositing with semi-transparent pixels, black edges/halos appear due to:

1. **Unpremultiplied garbage**: RGB values exist in pixels where alpha = 0
2. **Improper erosion**: Scaling RGB down with alpha creates dark fringes
3. **Wrong compositing math**: Mixing premultiplied and unpremultiplied data

### The Root Cause

In premultiplied images:
- `RGB_premult = RGB_straight * Alpha`
- When alpha is reduced during erosion, if you scale RGB proportionally, you get darkening

**Wrong approach:**
```cpp
// This causes black edges!
float scale = new_alpha / old_alpha;
pixel.red *= scale;
pixel.green *= scale;
pixel.blue *= scale;
pixel.alpha = new_alpha;
```

**Correct approach (Photoshop mode):**
```cpp
// Keep RGB unchanged, only modify alpha
pixel.alpha = new_alpha;
// RGB stays the same - maintains color continuity
```

---

## Core Compositing Strategy

A multi-step approach for edge extension:

### Step 1: Erosion
- Shrink the alpha channel inward
- **Critical**: Keep RGB unchanged (`blur_in_premult_space = true`)
- This creates the "starting point" for edge extension

### Step 2: Iterative Blur and Stack
- Apply progressively larger blurs to the eroded image
- Stack layers using **BEHIND** mode (more blurred under less blurred)
- This builds up the extended edge colors

### Step 3: Final Composite
- Composite the eroded original **ON TOP** using **IN_FRONT** mode
- Optionally constrain to original alpha bounds (Keep mode)

---

## SDK Suites Used

### PF_WorldTransformSuite1
Primary suite for compositing operations.

```cpp
AEFX_SuiteScoper<PF_WorldTransformSuite1> wtransformSuite(
    in_data,
    kPFWorldTransformSuite,
    kPFWorldTransformSuiteVersion1,
    out_data);
```

**Key function: `transfer_rect`**
```cpp
wtransformSuite->transfer_rect(
    in_data->effect_ref,
    PF_Quality_HI,           // Quality setting
    PF_MF_Alpha_PREMUL,      // Alpha interpretation flag
    NULL,                    // Masks (optional)
    &src->extent_hint,       // Source rectangle
    src,                     // Source world
    &comp_mode,              // Compositing mode structure
    NULL,                    // Matting (optional)
    0, 0,                    // Destination offset
    dst);                    // Destination world
```

### PF_FillMatteSuite2
For premultiplication/unpremultiplication operations.

```cpp
AEFX_SuiteScoper<PF_FillMatteSuite2> fillMatteSuite(
    in_data,
    kPFFillMatteSuite,
    kPFFillMatteSuiteVersion2,
    out_data);

// Premultiply
fillMatteSuite->premultiply(effect_ref, true, worldP);

// Unpremultiply
fillMatteSuite->premultiply(effect_ref, false, worldP);
```

### PF_Iterate Suites
For per-pixel operations across different bit depths.

```cpp
// 8-bit
AEFX_SuiteScoper<PF_Iterate8Suite1> iterate8Suite(...);
iterate8Suite->iterate(..., FilterImage8, ...);

// 16-bit
AEFX_SuiteScoper<PF_iterate16Suite2> iterate16Suite(...);
iterate16Suite->iterate(..., FilterImage16, ...);

// 32-bit float
AEFX_SuiteScoper<PF_iterateFloatSuite1> iterateFloatSuite(...);
iterateFloatSuite->iterate(..., FilterImage32, ...);
```

---

## Premultiplied Alpha Handling

### The PF_MF_Alpha_PREMUL Flag

This flag tells the SDK how to interpret alpha values during compositing:

```cpp
wtransformSuite->transfer_rect(
    ...
    PF_MF_Alpha_PREMUL,  // Tell SDK: "this data is premultiplied"
    ...
);
```

### Why Stay Premultiplied

Keeping data premultiplied throughout processing is beneficial because:

1. **No dark fringe artifacts**: Blurring premultiplied data doesn't introduce dark halos
2. **SDK compatibility**: AE's compositing assumes premultiplied data
3. **Faster processing**: No unnecessary conversion overhead

### Clearing Garbage in Transparent Areas

Before blurring, we clear RGB in fully transparent pixels:

```cpp
// Clear RGB if alpha is very low to prevent garbage from spreading
if (pixel->alpha < 0.001f) {
    pixel->red = 0.0f;
    pixel->green = 0.0f;
    pixel->blue = 0.0f;
}
```

This prevents unpremultiplied garbage (RGB values where alpha = 0) from bleeding into the extended edge.

---

## Layer Stacking with BEHIND Mode

### The Concept

To extend edges, we need to stack blurred versions with the least-blurred on top:

```
Top:    Layer 1 (least blurred) - closest to original
        Layer 2 (more blurred)
        Layer 3 (even more blurred)
Bottom: Layer N (most blurred) - extends furthest
```

### PF_Xfer_BEHIND Mode

```cpp
PF_CompositeMode comp_mode;
AEFX_CLR_STRUCT(comp_mode);
comp_mode.xfer = PF_Xfer_BEHIND;  // Source goes BEHIND destination
comp_mode.rand_seed = 0;
comp_mode.opacity = 255;
comp_mode.rgb_only = FALSE;
comp_mode.opacitySu = 1.0f;
```

### Implementation

```cpp
// temp3 is the accumulator (starts empty)
// temp1 contains the current blur iteration

if (iter_idx == 0) {
    // First iteration: Initialize accumulator
    ERR(PF_COPY(temp1_worldP, temp3_worldP, NULL, NULL));
} else {
    // Subsequent iterations: Composite BEHIND
    // This puts more-blurred content UNDER less-blurred content
    ERR(wtransformSuite->transfer_rect(
        in_data->effect_ref,
        PF_Quality_HI,
        PF_MF_Alpha_PREMUL,
        NULL,
        &temp1_worldP->extent_hint,
        temp1_worldP,        // Source: current (more blurred)
        &comp_mode,          // BEHIND mode
        NULL,
        0, 0,
        temp3_worldP));      // Dest: accumulator (less blurred on top)
}
```

---

## Final Compositing with IN_FRONT Mode

### Putting Original Back on Top

After building the extended edge, we composite the eroded original back on top:

```cpp
PF_CompositeMode comp_mode;
AEFX_CLR_STRUCT(comp_mode);
comp_mode.xfer = PF_Xfer_IN_FRONT;  // Source goes IN FRONT of destination
comp_mode.rand_seed = 0;
comp_mode.opacity = 255;
comp_mode.rgb_only = FALSE;
comp_mode.opacitySu = 1.0f;

// output_worldP already contains the stacked blur layers
// temp3_worldP contains the eroded original

ERR(wtransformSuite->transfer_rect(
    in_data->effect_ref,
    PF_Quality_HI,
    PF_MF_Alpha_PREMUL,
    NULL,
    &temp3_worldP->extent_hint,
    temp3_worldP,        // Source: eroded original
    &comp_mode,          // IN_FRONT mode
    NULL,
    0, 0,
    output_worldP));     // Dest: stacked blur layers
```

---

## The "Photoshop Mode" Approach

### What Is It?

"Photoshop mode" refers to blurring in premultiplied space, which is how Photoshop handles blur operations. This contrasts with "Nuke mode" which unpremultiplies, blurs, and repremultiplies.

### Implementation in Erosion

The key is the `blur_in_premult_space` flag:

```cpp
// ApplyErosion_Template

if (new_alpha < PixelTraits<PixelType>::GetAlpha(inP) &&
    PixelTraits<PixelType>::GetAlpha(inP) > 0.0f) {

    if (blur_in_premult_space) {
        // Photoshop mode: Keep RGB unchanged, only erode alpha
        // This prevents black edges when staying in premult space
        SetRGB(outP, GetRed(inP), GetGreen(inP), GetBlue(inP));
        SetAlpha(outP, new_alpha);
    } else {
        // Normal mode: Scale RGB to maintain premultiplication
        // This can cause black fringes
        float scale = new_alpha / GetAlpha(inP);
        SetRGB(outP, GetRed(inP) * scale, GetGreen(inP) * scale, GetBlue(inP) * scale);
        SetAlpha(outP, new_alpha);
    }
}
```

### Why It Works

In premultiplied space:
- RGB values are already multiplied by alpha
- When we reduce alpha but keep RGB, the "straight" color appears to get brighter
- But after the blur, the SDK's compositing math handles this correctly
- Result: Smooth color extension without dark halos

---

## Alpha Expansion for Edge Extension

### Purpose

Alpha expansion pushes the semi-transparent edge further out by remapping alpha values:

```
Original: alpha 0.1 -> Expanded: alpha 0.5
Original: alpha 0.5 -> Expanded: alpha 1.0
```

### S-Curve Smoothstep

We use a smoothstep function to avoid harsh transitions:

```cpp
// Smooth S-curve: 3t^2 - 2t^3 (smoothstep)
float t2 = alpha * alpha;
float t3 = t2 * alpha;
alpha = 3.0f * t2 - 2.0f * t3;
```

### SIMD Optimization (32-bit)

For 32-bit float, we use AVX2 for performance:

```cpp
#ifdef __AVX2__
// Process 8 pixels at a time
__m256 t2 = _mm256_mul_ps(alphas, alphas);      // t^2
__m256 t3 = _mm256_mul_ps(t2, alphas);          // t^3
__m256 three = _mm256_set1_ps(3.0f);
__m256 two = _mm256_set1_ps(2.0f);
alphas = _mm256_sub_ps(_mm256_mul_ps(three, t2), _mm256_mul_ps(two, t3));
#endif
```

---

## Keep Original Alpha Mode

### Purpose

Sometimes you want extended colors but constrained to the original alpha bounds. This is useful when:
- The layer has a precise matte you want to preserve
- You only want to fix edge colors, not expand coverage

### Implementation

```cpp
// ConstrainToAlpha_Template

template<typename PixelType>
inline void ConstrainToAlpha_Template(
    const PixelType& inP,      // Extended result
    const PixelType& origP,    // Original input
    PixelType& outP,
    float alpha_gamma)
{
    // Apply optional gamma to original alpha
    float adjusted_alpha = GetAlpha(origP);
    if (alpha_gamma > 0.001f) {
        adjusted_alpha = powf(adjusted_alpha, 1.0f / alpha_gamma);
    }

    // If original was transparent, output is transparent
    if (GetAlpha(origP) < MIN_THRESHOLD) {
        SetRGBA(outP, 0.0f, 0.0f, 0.0f, 0.0f);
    } else {
        // Keep extended RGB but use original (gamma-adjusted) alpha
        SetRGB(outP, GetRed(inP), GetGreen(inP), GetBlue(inP));
        SetAlpha(outP, adjusted_alpha);
    }
}
```

---

## Common Pitfalls and Solutions

### Pitfall 1: Black Halos from Erosion

**Symptom**: Dark ring around the edge after erosion

**Cause**: RGB scaled down with alpha

**Solution**: Use `blur_in_premult_space = true`

### Pitfall 2: Color Bleeding from Garbage Data

**Symptom**: Strange colors appearing in extended edges

**Cause**: RGB values exist where alpha = 0 in input

**Solution**: Clear RGB in low-alpha pixels before blurring:
```cpp
if (pixel->alpha < threshold) {
    pixel->red = pixel->green = pixel->blue = 0;
}
```

### Pitfall 3: Wrong Compositing Order

**Symptom**: Most-blurred layer visible instead of least-blurred

**Cause**: Using wrong transfer mode or order

**Solution**:
- Use `PF_Xfer_BEHIND` for layer stacking
- Start with least-blurred, composite more-blurred behind

### Pitfall 4: Double Premultiplication

**Symptom**: Colors appear washed out or too dark

**Cause**: Premultiplying already-premultiplied data

**Solution**: Track premult state, use `PF_MF_Alpha_PREMUL` flag

### Pitfall 5: 16-bit Quantization Artifacts

**Symptom**: Banding in gradients

**Cause**: Intermediate calculations in 16-bit precision

**Solution**: Use float math internally, quantize only at output

---

## Quick Reference: Transfer Modes

| Mode | SDK Constant | Behavior |
|------|-------------|----------|
| Normal (Over) | `PF_Xfer_IN_FRONT` | Source over destination |
| Behind | `PF_Xfer_BEHIND` | Source under destination |
| Add | `PF_Xfer_ADD` | Additive blend |
| Multiply | `PF_Xfer_MULTIPLY` | Multiplicative blend |

## Quick Reference: Alpha Flags

| Flag | Value | Use When |
|------|-------|----------|
| `PF_MF_Alpha_PREMUL` | - | Data is premultiplied |

> **Note:** `PF_MF_Alpha_STRAIGHT` may not exist as a named constant in all SDK versions. The absence of `PF_MF_Alpha_PREMUL` typically implies straight alpha.

---

## Summary

The key to avoiding black edges in compositing:

1. **Stay premultiplied** throughout the pipeline
2. **Clear garbage** in transparent pixels before blurring
3. **Use "Photoshop mode"** erosion (don't scale RGB with alpha)
4. **Stack with BEHIND mode** (less blurred on top)
5. **Composite original with IN_FRONT** at the end
6. **Tell SDK about premult state** via `PF_MF_Alpha_PREMUL`
