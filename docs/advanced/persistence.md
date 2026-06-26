# Persistence

> 1 Q&A · source: AE plugin dev community Discord

### Can I save sequence data during PF_Cmd_SEQUENCE_SETDOWN?

No. Sequence setdown is AE's signal to destruct and free the data, not to save it. The data saved with the project is the last handle used or flattened. To have data saved with the project, set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING during global setup to receive PF_Cmd_SEQUENCE_FLATTEN calls, where you serialize your data into a flat handle that AE will store.

*Tags: `flatten`, `persistence`, `project-save`, `sequence-data`, `sequence-setdown`*

---
