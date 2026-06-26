# Command Hook

> 2 Q&As · source: AE plugin dev community Discord

### How can I detect when an effect is removed from a layer?

There's no direct API for detection. PF_Cmd_SEQUENCE_SETDOWN is called when AE destroys an instance, but only when the deletion is out of the undo stack (100 steps later) or on project close/purge. Approaches: (1) Scan the project during idle_hook, catalog effect instances, and detect when one disappears. Not immediate. (2) Monitor 'cut' and 'delete' via command hooks, but this is unreliable - redo operations don't trigger menu commands. Important considerations: 'cut' puts the effect in limbo (can be pasted back or undone). For the idle_hook approach, note that in_data is not available in IdleHook - you need to acquire suites differently, using the SPBasicSuite passed during AEGP initialization.

*Tags: `command-hook`, `effect-removal`, `idle-hook`, `sequence-setdown`, `undo-stack`*

---

### How can I detect when an AE project is saved, from a plugin?

There's no reliable pre-save notification. Two approaches: (1) Use command_hook to catch save events, but it's not 100% bulletproof (sometimes saves happen without triggering the hook). (2) Implement an idle_hook and check if the project dirty flag changes from dirty to clean - if so, the user has saved. For effect plugins specifically, use sequence data with PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING, which causes AE to call your effect with PF_Cmd_SEQUENCE_FLATTEN on save.

*Tags: `command-hook`, `dirty-flag`, `idle-hook`, `project-save`, `sequence-flatten`*

---
