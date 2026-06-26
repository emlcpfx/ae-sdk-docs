# Flatten

> 4 Q&As · source: AE plugin dev community Discord

### How do you serialize/capture arb data from arbitrary third-party plugins?

Arb data is serialized via PF_Cmd_Arbitrary_Callback using flatten/unflatten functions, similar to sequence data. The flatten/unflatten versions are not accessible via AEGP functions. For plugins with pure stack-variable arb data (like Curves), you can reinterpret_cast the handle as char* and serialize directly. However, plugins with pointers in their arb structs (like Liquify) cannot be serialized this way because deserialized pointer addresses won't be valid. ExtendScript may offer a path through stream values, but it's at the limit of what's possible.

*Tags: `aegp`, `arb-data`, `flatten`, `serialization`, `unflatten`*

---

### Why is PF_Cmd_SEQUENCE_FLATTEN called when an effect is first applied, and why are Sequence Resetup/Setdown called multiple times on different threads?

This is correct behavior when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set. AE flattens sequence data so it can be copied (for render thread copies). Since AE 2015, rendering uses separate threads with separate project copies. Resetup may receive unflat data, so tag your data to indicate flat/unflat state. Use PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA to allow AE to get the flat copy without losing the original unflat one, reducing unnecessary resetup calls. During sequence setup/resetup, don't assume the plugin can find its own layer/comp - AE sometimes constructs the instance before associating it to a layer. For unique instance IDs, flag sequence data as 'requires checking' on resetup, then scan for duplicates when the instance can be located.

*Tags: `flatten`, `instance-id`, `mfr`, `resetup`, `sequence-data`, `threading`*

---

### Can I save sequence data during PF_Cmd_SEQUENCE_SETDOWN?

No. Sequence setdown is AE's signal to destruct and free the data, not to save it. The data saved with the project is the last handle used or flattened. To have data saved with the project, set PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING during global setup to receive PF_Cmd_SEQUENCE_FLATTEN calls, where you serialize your data into a flat handle that AE will store.

*Tags: `flatten`, `persistence`, `project-save`, `sequence-data`, `sequence-setdown`*

---

### How does PF_Cmd_SEQUENCE_RESETUP work and when is sequence data flattened?

PF_Cmd_SEQUENCE_RESETUP is called either when opening a project with saved (flat) sequence data, or after AE has just asked to flatten sequence data (for a save) and now needs to unflatten it. The in_data->sequence_data pointer will always point to flattened data at this point, and your task is to unflatten it. There is a flag in the input data that tells you if data is flat or not. The documentation phrase 're-create (usually unflatten) sequence data' means that for simple plugins that only use regular parameters and don't need flattening/unflattening, the unflatten step is unnecessary. The input data should always be flat and the output data should always be unflat for this function.

```cpp
// We got here because we're either opening a project w/saved (flat) sequence data,
// or we've just been asked to flatten our sequence data (for a save) and now we're blowing it back up.
```

*Tags: `flatten`, `project-save`, `sequence-data`, `sequence-resetup`, `unflatten`*

---
