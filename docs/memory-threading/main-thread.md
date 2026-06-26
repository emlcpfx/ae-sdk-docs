# Main Thread

> 1 Q&A · source: AE plugin dev community Discord

### Is it possible to import footage in the background using AEGP functions?

No, it is not possible to import footage in the background. AEGP functions must execute on the main thread only, so background import operations are not supported by the After Effects SDK.

*Tags: `aegp`, `import`, `main-thread`, `threading`*

---
