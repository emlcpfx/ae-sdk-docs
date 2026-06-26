# Purge

> 1 Q&A · source: AE plugin dev community Discord

### Why does memory usage spike to 6+ GB when I add PF_OutFlag_NON_PARAM_VARY to my plugin?

This is normal AE behavior - it's hoarding cached frames. PF_OutFlag_NON_PARAM_VARY tells AE the output varies independently of parameters, so AE caches each unique frame. Verify by purging AE's memory (Edit > Purge > All Memory) - if RAM consumption drops to expected levels, everything is working correctly. AE will release cached memory when it's needed for new frame renders. PF_OutFlag_WIDE_TIME_INPUT is only needed if you use a layer selector and sample it at different times. Implement PF_Cmd_FRAME_SETDOWN to free any memory allocated during frame setup.

*Tags: `caching`, `frame-setdown`, `frame-setup`, `memory`, `non-param-vary`, `purge`*

---
