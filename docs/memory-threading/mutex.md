# Mutex

> 1 Q&A · source: AE plugin dev community Discord

### Why is sequence data different between PF_Cmd_EVENT (UI thread) and SmartPreRender (render thread)?

Sequence data is separate on the render and UI threads by design. The render threads are free to render asynchronously without having their data change from another thread. Global data IS shared between render and UI threads, though it's not instance-specific. To share per-instance data: store data in global data keyed by instance identifiers (comp item ID + layer ID + effect index on layer), protected by a mutex. Comp item IDs don't change during a session. Layer IDs don't change on reorder but may change on copy/paste. Effect index changes when moved in the stack.

*Tags: `global-data`, `mutex`, `render-thread`, `sequence-data`, `threading`, `ui-thread`*

---
