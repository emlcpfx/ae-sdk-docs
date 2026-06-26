# Unflatten

> 2 Q&As · source: AE plugin dev community Discord

### How do you serialize/capture arb data from arbitrary third-party plugins?

Arb data is serialized via PF_Cmd_Arbitrary_Callback using flatten/unflatten functions, similar to sequence data. The flatten/unflatten versions are not accessible via AEGP functions. For plugins with pure stack-variable arb data (like Curves), you can reinterpret_cast the handle as char* and serialize directly. However, plugins with pointers in their arb structs (like Liquify) cannot be serialized this way because deserialized pointer addresses won't be valid. ExtendScript may offer a path through stream values, but it's at the limit of what's possible.

*Tags: `aegp`, `arb-data`, `flatten`, `serialization`, `unflatten`*

---

### How does PF_Cmd_SEQUENCE_RESETUP work and when is sequence data flattened?

PF_Cmd_SEQUENCE_RESETUP is called either when opening a project with saved (flat) sequence data, or after AE has just asked to flatten sequence data (for a save) and now needs to unflatten it. The in_data->sequence_data pointer will always point to flattened data at this point, and your task is to unflatten it. There is a flag in the input data that tells you if data is flat or not. The documentation phrase 're-create (usually unflatten) sequence data' means that for simple plugins that only use regular parameters and don't need flattening/unflattening, the unflatten step is unnecessary. The input data should always be flat and the output data should always be unflat for this function.

```cpp
// We got here because we're either opening a project w/saved (flat) sequence data,
// or we've just been asked to flatten our sequence data (for a save) and now we're blowing it back up.
```

*Tags: `flatten`, `project-save`, `sequence-data`, `sequence-resetup`, `unflatten`*

---
