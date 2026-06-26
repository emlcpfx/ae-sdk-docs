# Open Path

> 1 Q&A · source: AE plugin dev community Discord

### Why does the render not refresh automatically when drawing/deleting an open path/mask with PF_Path suites?

You need to set the flag PF_OutFlag2_DEPENDS_ON_UNREFERENCED_MASKS. This tells After Effects to invalidate your output when a mask is modified that doesn't appear to be referenced by your effect. Closed masks automatically refresh because they alter the input layer pixels before they reach your plugin (AE must call the plugin again since input changed). Open masks don't alter anything normally, so AE thinks there's no reason to re-render.

*Tags: `masks`, `open-path`, `outflags`, `pf-path`, `render-invalidation`*

---
