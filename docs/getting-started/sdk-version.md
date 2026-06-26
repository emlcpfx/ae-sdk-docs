# Sdk Version

> 1 Q&A · source: AE plugin dev community Discord

### Why might SmartRender never be called even though PreRender returns PF_Err_NONE?

Possible causes: (1) GuidMixInPtr in PreRender is telling AE 'no new render needed' if the mixed-in data hasn't changed. (2) Mismatched SDK versions - building with 2025.2 SDK source but 2023 SDK headers causes undefined behavior, especially in Release builds where optimizations may expose the mismatch. Always ensure your SDK headers and source files are from the same version.

*Tags: `guid-mixin`, `pre-render`, `release-build`, `sdk-version`, `smart-render`*

---
