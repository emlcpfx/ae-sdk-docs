# Undo Redo

> 1 Q&A · source: AE plugin dev community Discord

### What is the correct workflow for storing custom data in an arbitrary (ARB) parameter for custom UI interactions?

The workflow should be: user interacts with UI -> data is stored in a LOCAL array -> local array is serialized and stored in an ARB -> AE sees new value and invalidates cached frames. Before displaying UI, read the value from the ARB, deserialize into your local array, and draw accordingly (this ensures undo/redo reflects correctly). Serializing means writing all data into one continuous block of AE-allocated memory. You don't need to convert to a string - use whichever method works for collecting data in and reconstructing out. This is referred to as 'flattening' and 'unflattening' in the AE SDK.

*Tags: `arb-param`, `custom-ui`, `ecw`, `flattening`, `serialization`, `undo-redo`*

---
