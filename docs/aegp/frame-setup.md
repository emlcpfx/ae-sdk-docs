# Frame Setup

> 3 Q&As · source: AE plugin dev community Discord

### Why does memory usage spike to 6+ GB when I add PF_OutFlag_NON_PARAM_VARY to my plugin?

This is normal AE behavior - it's hoarding cached frames. PF_OutFlag_NON_PARAM_VARY tells AE the output varies independently of parameters, so AE caches each unique frame. Verify by purging AE's memory (Edit > Purge > All Memory) - if RAM consumption drops to expected levels, everything is working correctly. AE will release cached memory when it's needed for new frame renders. PF_OutFlag_WIDE_TIME_INPUT is only needed if you use a layer selector and sample it at different times. Implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup.

*Tags: `caching`, `frame-setdown`, `frame-setup`, `memory`, `non-param-vary`, `purge`*

---

### How can I resize the output buffer in an After Effects plugin and determine the correct size before rendering?

You cannot access the input image during Frame_Setup, but you have two options: (1) Save necessary data during the render call and trigger a re-render using PF_ForceRerender from AdvItemSuite, during which you can change the output size; or (2) Use the layer dimensions from in_data->width and in_data->height, which represent the unaffected layer dimensions. For more advanced control, consider migrating to SmartFX, which uses pre-render calls instead of Frame_Setup and supports 32-bit color depth.

*Tags: `aegp`, `frame-setup`, `output-rect`, `render-loop`*

---

### Can I change the output size during a re-render in After Effects plugins?

Yes, you can change the output size during a re-render. When you call for a re-render using PF_ForceRerender, you receive a Frame_Setup call first, during which you can modify the size even if the frame was previously rendered. This allows you to adjust output dimensions based on calculations from the previous render pass.

*Tags: `aegp`, `frame-setup`, `output-rect`, `render-loop`*

---
