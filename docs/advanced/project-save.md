# Project Save

> 3 Q&As · source: AE plugin dev community Discord

### Can I save sequence data during PF_Cmd_SEQUENCE_SETDOWN?

No. Sequence setdown is AE's signal to destruct and free the data, not to save it. The data saved with the project is the last handle used or flattened. To have data saved with the project, set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING during global setup to receive PF_Cmd_SEQUENCE_FLATTEN calls, where you serialize your data into a flat handle that AE will store.

*Tags: `flatten`, `persistence`, `project-save`, `sequence-data`, `sequence-setdown`*

---

### How can I detect when an AE project is saved, from a plugin?

There's no reliable pre-save notification. Two approaches: (1) Use command_hook to catch save events, but it's not 100% bulletproof (sometimes saves happen without triggering the hook). (2) Implement an idle_hook and check if the project dirty flag changes from dirty to clean - if so, the user has saved. For effect plugins specifically, use sequence data with PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING, which causes AE to call your effect with PF_Cmd_SEQUENCE_FLATTEN on save.

*Tags: `command-hook`, `dirty-flag`, `idle-hook`, `project-save`, `sequence-flatten`*

---

### How does PF_Cmd_SEQUENCE_RESETUP work and when is sequence data flattened?

PF_Cmd_SEQUENCE_RESETUP is called either when opening a project with saved (flat) sequence data, or after AE has just asked to flatten sequence data (for a save) and now needs to unflatten it. The in_data->sequence_data pointer will always point to flattened data at this point, and your task is to unflatten it. There is a flag in the input data that tells you if data is flat or not. The documentation phrase 're-create (usually unflatten) sequence data' means that for simple plugins that only use regular parameters and don't need flattening/unflattening, the unflatten step is unnecessary. The input data should always be flat and the output data should always be unflat for this function.

```cpp
// We got here because we're either opening a project w/saved (flat) sequence data,
// or we've just been asked to flatten our sequence data (for a save) and now we're blowing it back up.
```

*Tags: `flatten`, `project-save`, `sequence-data`, `sequence-resetup`, `unflatten`*

---
