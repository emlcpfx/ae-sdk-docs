# Execute Script

> 1 Q&A · source: AE plugin dev community Discord

### Is there an API to get the layer comment field from C++ code?

There is no C API for getting layer comments. Use AEGP_ExecuteScript() to run ExtendScript that reads the layer comment and retrieve the result back to the C side.

*Tags: `aegp`, `execute-script`, `extendscript`, `layer-comment`*

---
