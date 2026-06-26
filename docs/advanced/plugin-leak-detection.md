# Plugin Leak Detection

> 1 Q&A · source: AE plugin dev community Discord

### Why doesn't macOS Instruments detect memory leaks from undisposed PF_EffectWorlds even though After Effects runs out of memory?

PF_EffectWorlds allocated in After Effects plugins may not be detected by Instruments leak detection because After Effects itself may be managing the memory lifecycle through its own memory management systems, or the allocations may be attributed to After Effects' internal memory pools rather than the plugin's direct allocations. Instruments can detect direct memory leaks (e.g., data allocated in pre-render that isn't disposed), but large framework-managed allocations like PF_EffectWorlds may be tracked differently and not flagged as leaks even when they cause memory exhaustion.

*Tags: `debugging`, `macos`, `memory`, `plugin-leak-detection`*

---
