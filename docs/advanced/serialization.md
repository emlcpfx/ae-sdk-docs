# Serialization

> 4 Q&As · source: AE plugin dev community Discord

### How do you serialize/capture arb data from arbitrary third-party plugins?

Arb data is serialized via PF_Cmd_Arbitrary_Callback using flatten/unflatten functions, similar to sequence data. The flatten/unflatten versions are not accessible via AEGP functions. For plugins with pure stack-variable arb data (like Curves), you can reinterpret_cast the handle as char* and serialize directly. However, plugins with pointers in their arb structs (like Liquify) cannot be serialized this way because deserialized pointer addresses won't be valid. ExtendScript may offer a path through stream values, but it's at the limit of what's possible.

*Tags: `aegp`, `arb-data`, `flatten`, `serialization`, `unflatten`*

---

### What is the correct workflow for storing custom data in an arbitrary (ARB) parameter for custom UI interactions?

The workflow should be: user interacts with UI -> data is stored in a LOCAL array -> local array is serialized and stored in an ARB -> AE sees new value and invalidates cached frames. Before displaying UI, read the value from the ARB, deserialize into your local array, and draw accordingly (this ensures undo/redo reflects correctly). Serializing means writing all data into one continuous block of AE-allocated memory. You don't need to convert to a string - use whichever method works for collecting data in and reconstructing out. This is referred to as 'flattening' and 'unflattening' in the AE SDK.

*Tags: `arb-param`, `custom-ui`, `ecw`, `flattening`, `serialization`, `undo-redo`*

---

### Is there a way to serialize all plugin data including arbitrary data?

No clear answer was provided in this conversation. The question was asked but not addressed by other participants.

*Tags: `arb-data`, `plugin-data`, `serialization`*

---

### How does After Effects serialize arbitrary data from plugins to disk?

Arbitrary data is serialized using PF_Cmd_Arbitrary_Callback with the flatten function, similar to sequence data. However, the flatten/unflatten versions are not accessible via AEGP functions. For plugins with stack-variable handles (like "curves"), the handle can be reinterpreted cast as char* and serialized. Plugins with pointers in their structures (like "liquify") cannot be easily serialized this way because deserialized pointer addresses are invalid on reload, causing crashes.

*Tags: `aegp`, `arb-data`, `params`, `serialization`*

---
