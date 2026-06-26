# Effect Removal

> 1 Q&A · source: AE plugin dev community Discord

### How can I detect when an effect is removed from a layer?

There's no direct API for detection. PF_Cmd_SEQUENCE_SETDOWN is called when AE destroys an instance, but only when the deletion is out of the undo stack (100 steps later) or on project close/purge. Approaches: (1) Scan the project during idle_hook, catalog effect instances, and detect when one disappears. Not immediate. (2) Monitor 'cut' and 'delete' via command hooks, but this is unreliable - redo operations don't trigger menu commands. Important considerations: 'cut' puts the effect in limbo (can be pasted back or undone). For the idle_hook approach, note that in_data is not available in IdleHook - you need to acquire suites differently, using the SPBasicSuite passed during AEGP initialization.

*Tags: `command-hook`, `effect-removal`, `idle-hook`, `sequence-setdown`, `undo-stack`*

---
