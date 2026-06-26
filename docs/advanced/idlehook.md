# Idlehook

> 1 Q&A · source: AE plugin dev community Discord

### What are the limitations of using AEGP_ExecuteScript from IdleHook?

AEGP_ExecuteScript will fail with 'Cannot run a script while a modal dialog is waiting for a response' if any modal windows (like Composition Settings) are open. Additionally, other AEGP suites like StreamSuite may be inaccessible from IdleHook, limiting what can be modified from background threads.

*Tags: `aegp`, `idlehook`, `scripting`, `threading`*

---
