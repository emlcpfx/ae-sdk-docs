# Ui Thread

> 2 Q&As · source: AE plugin dev community Discord

### Why is sequence data different between PF_Cmd_EVENT (UI thread) and SmartPreRender (render thread)?

Sequence data is separate on the render and UI threads by design. The render threads are free to render asynchronously without having their data change from another thread. Global data IS shared between render and UI threads, though it's not instance-specific. To share per-instance data: store data in global data keyed by instance identifiers (comp item ID + layer ID + effect index on layer), protected by a mutex. Comp item IDs don't change during a session. Layer IDs don't change on reorder but may change on copy/paste. Effect index changes when moved in the stack.

*Tags: `global-data`, `mutex`, `render-thread`, `sequence-data`, `threading`, `ui-thread`*

---

### How can I force a re-render from a separate thread (e.g., when receiving WebSocket messages)?

It's only possible via idle_hook, which runs on the UI thread (30-50 times per second). Check your other thread for messages during idle events. Options to force re-render: (1) Change a hidden/invisible parameter value - this forces AE to re-render. (2) Call the command number to purge the cache. (3) Call AEGP_EffectCallGeneric to have the effect instance act on itself. You cannot set out_data->out_flags from a separate thread - it must happen on the main UI thread during an AE callback.

*Tags: `force-rerender`, `hidden-param`, `idle-hook`, `threading`, `ui-thread`, `websocket`*

---
