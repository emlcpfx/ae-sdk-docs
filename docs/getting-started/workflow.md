# Workflow

> 1 Q&A · source: AE plugin dev community Discord

### What UI approach do some AE plugins use to improve interactivity with parameters?

Some plugins like Gifgun and Datamosh use an external application window (often built with C++ and ImGui) that allows users to adjust parameters interactively in a fullscreen UI, then pre-render and send results back to After Effects. This avoids the need for re-rendering on every parameter change within the comp, which would be required if using a custom UI directly in After Effects like Optical Flares does.

*Tags: `open-source`, `sequence-data`, `ui`, `workflow`*

---
