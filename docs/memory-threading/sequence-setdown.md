# Sequence Setdown

> 2 Q&As · source: AE plugin dev community Discord

### How can I detect when an effect is removed from a layer?

There's no direct API for detection. PF_Cmd_SEQUENCE_SETDOWN is called when AE destroys an instance, but only when the deletion is out of the undo stack (100 steps later) or on project close/purge. Approaches: (1) Scan the project during idle_hook, catalog effect instances, and detect when one disappears. Not immediate. (2) Monitor 'cut' and 'delete' via command hooks, but this is unreliable - redo operations don't trigger menu commands. Important considerations: 'cut' puts the effect in limbo (can be pasted back or undone). For the idle_hook approach, note that in_data is not available in IdleHook - you need to acquire suites differently, using the SPBasicSuite passed during AEGP initialization.

*Tags: `command-hook`, `effect-removal`, `idle-hook`, `sequence-setdown`, `undo-stack`*

---

### Can I save sequence data during PF_Cmd_SEQUENCE_SETDOWN?

No. Sequence setdown is AE's signal to destruct and free the data, not to save it. The data saved with the project is the last handle used or flattened. To have data saved with the project, set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING during global setup to receive PF_Cmd_SEQUENCE_FLATTEN calls, where you serialize your data into a flat handle that AE will store.

*Tags: `flatten`, `persistence`, `project-save`, `sequence-data`, `sequence-setdown`*

---
