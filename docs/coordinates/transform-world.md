# Transform_World

> 6 Q&As · source: AE plugin dev community Discord

### What is the correct first argument to pass to transform_world and other suite callbacks in After Effects plugins?

According to community expert Shachar Carmi, the Adobe documentation is incorrect about the first argument. Instead of passing in_data as documented, you should pass NULL or in_data->effect_ref. Passing NULL works because many suite callbacks accept null instead of effect_ref, and the PF_INTERRUPT macro internally uses effect_ref to make interrupt checks. The incorrect documentation may be legacy from before CC2015 when a separate rendering thread was introduced.

*Tags: `debugging`, `params`, `reference`, `sdk`, `transform_world`*

---

### Why does transform_world give better results when entering parameter values numerically versus dragging sliders?

When working with transform_world, numeric input (typing a value and pressing enter) produces better results than slider dragging. This is theorized to be related to how After Effects halts the transformation process during interactive slider adjustments to provide a smoother user experience, though this behavior may be legacy code from before CC2015's separate rendering thread was introduced.

*Tags: `params`, `render-loop`, `transform_world`, `ui`*

---

### Can you use transform_world multiple times in a loop with the same output buffer as input for subsequent transformations?

No, you cannot use the same buffer as both input and output for transform_world, as it reads and overwrites simultaneously, resulting in corrupted output. Instead, create a temporary buffer and alternate between buffers: transform input to temp, temp to output, output back to temp, repeating as needed. Never overwrite the original input buffer that After Effects caches, as this causes bizarre bugs.

```cpp
// Correct approach for repeated transformations
// 1. create a new buffer "temp"
// 2. transform the input to temp
ERR(in_data->utils->transform_world(in_data->effect_ref, in_data->quality, PF_MF_Alpha_STRAIGHT, in_data->field, input, &composite_mode, NULL, &matrix1, 1L, TRUE, &output->extent_hint, temp));
// 3. transform temp to output
ERR(in_data->utils->transform_world(in_data->effect_ref, in_data->quality, PF_MF_Alpha_STRAIGHT, in_data->field, temp, &composite_mode, NULL, &matrix2, 1L, TRUE, &output->extent_hint, output));
// 4. repeat by transforming output back to temp for next iteration
```

*Tags: `aegp`, `debugging`, `memory`, `render-loop`, `transform_world`*

---

### Is it better to apply multiple transform_world operations separately or combine them into a single matrix multiplication?

Always combine transformations into a single matrix multiplication rather than applying multiple transform_world renders. Matrix multiplication of 3x3 matrices is extremely cheap (~20 operations), while each transform_world render performs millions of pixel operations. Additionally, multiple sequential transforms cause image degradation (blur/jaggedness) because pixels that don't land on exact whole pixels get blended, compounding with each render. A single combined transformation applied once produces superior quality and performance.

*Tags: `optimization`, `performance`, `render-loop`, `transform_world`*

---

### How can you adjust the level of motion blur when using transform_world in a plugin?

According to shachar carmi, transform_world spreads 16 instances across provided matrices to create motion blur. The level of motion blur can be adjusted in three ways: (1) by controlling the length of the blur (shutter angle), (2) by changing the number of samples in the blur, or (3) by adjusting the mix between the motion-blurred result and non-motion-blurred render. Providing more matrices gives a more refined look with better rotations and curved paths, though it doesn't increase the total number of samples. With 2 matrices you get 16 linear positions between them; with 3 matrices you get 8 samples from mat1 to mat2 and another 8 from mat2 to mat3.

*Tags: `motion-blur`, `params`, `premiere`, `smartfx`, `transform_world`*

---

### Can a 3x3 matrix be used to apply perspective transformations with transform_world()?

No. 3x3 matrices can only perform 2D affine transformations: position, scale, Z rotation, and skewing (anything the Transform effect does). 4x4 matrices contain additional parameters for X and Y rotations and Z translations. You cannot directly translate a 4x4 matrix into a 3x3 matrix; instead, you must decompose the 4x4 matrix and re-feed its components into the 3x3. Corner pin style transforms are not possible using transform_world().

*Tags: `3d`, `affine`, `matrix`, `perspective`, `transform_world`*

---
