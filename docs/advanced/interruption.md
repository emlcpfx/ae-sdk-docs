# Interruption

> 1 Q&A · source: AE plugin dev community Discord

### What happens when a user interrupts rendering by scrubbing the playhead in After Effects?

When the user interrupts rendering by moving the playhead, After Effects does not wait for the render function to complete, and frame setdown is not called in-between. The plugin should call PF_ABORT to check if After Effects wants to quit the current frame's render. If the plugin decides to honor the abort request, it should return PF_Interrupt_CANCEL as the error code from the main entry point.

*Tags: `aegp`, `interruption`, `mfr`, `render-loop`*

---
