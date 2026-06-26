# Motion Blur

> 3 Q&As · source: AE plugin dev community Discord

### Where can I find discussions about transform_world and motion blur issues in After Effects?

Rich shared a question posted on the Adobe After Effects community forum discussing transform_world and motion blur. The forum thread can be found at https://community.adobe.com/t5/after-effects-discussions/transform-world-amp-motion-blur/td-p/12918545

*Tags: `debugging`, `motion-blur`, `reference`, `transform`*

---

### How can you adjust the level of motion blur when using transform_world in a plugin?

According to shachar carmi, transform_world spreads 16 instances across provided matrices to create motion blur. The level of motion blur can be adjusted in three ways: (1) by controlling the length of the blur (shutter angle), (2) by changing the number of samples in the blur, or (3) by adjusting the mix between the motion-blurred result and non-motion-blurred render. Providing more matrices gives a more refined look with better rotations and curved paths, though it doesn't increase the total number of samples. With 2 matrices you get 16 linear positions between them; with 3 matrices you get 8 samples from mat1 to mat2 and another 8 from mat2 to mat3.

*Tags: `motion-blur`, `params`, `premiere`, `smartfx`, `transform_world`*

---

### Why is motion blur incorrect when using PF_TransformWorld with fast rotation?

When using PF_TransformWorld with only two matrices (previous and current), motion blur artifacts can occur during fast rotation, causing particles to appear size-limited. The solution is to use multiple intermediate matrices for in-between values rather than just the start and end positions. Additionally, ensure that the PF_Rect *dest_rect is expanded to encompass the full area of all rotated samples to capture the complete motion blur region.

```cpp
PF_Err transform_world (
  PF_InData *in_data,
  PF_Quality quality,
  PF_ModeFlags m_flags,
  PF_Field field,
  const PF_EffectWorld *src_world,
  const PF_CompositeMode *comp_mode,
  const PF_MaskWorld *mask_world0,
  const PF_FloatMatrix *matrices,
  A_long num_matrices,
  Boolean src2dst_matrix,
  const PF_Rect *dest_rect,
  PF_EffectWorld *dst_world);
```

*Tags: `mfr`, `motion-blur`, `output-rect`, `rendering`, `transform`*

---
