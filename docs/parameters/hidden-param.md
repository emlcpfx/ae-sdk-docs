# Hidden Param

> 1 Q&A · source: AE plugin dev community Discord

### How can I force a re-render from a separate thread (e.g., when receiving WebSocket messages)?

It's only possible via idle_hook, which runs on the UI thread (30-50 times per second). Check your other thread for messages during idle events. Options to force re-render: (1) Change a hidden/invisible parameter value - this forces AE to re-render. (2) Call the command number to purge the cache. (3) Call AEGP_EffectCallGeneric to have the effect instance act on itself. You cannot set out_data->out_flags from a separate thread - it must happen on the main UI thread during an AE callback.

*Tags: `force-rerender`, `hidden-param`, `idle-hook`, `threading`, `ui-thread`, `websocket`*

---
