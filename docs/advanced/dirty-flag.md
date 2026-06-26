# Dirty Flag

> 1 Q&A · source: AE plugin dev community Discord

### How can I detect when an AE project is saved, from a plugin?

There's no reliable pre-save notification. Two approaches: (1) Use command_hook to catch save events, but it's not 100% bulletproof (sometimes saves happen without triggering the hook). (2) Implement an idle_hook and check if the project dirty flag changes from dirty to clean - if so, the user has saved. For effect plugins specifically, use sequence data with PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING, which causes AE to call your effect with PF_Cmd_SEQUENCE_FLATTEN on save.

*Tags: `command-hook`, `dirty-flag`, `idle-hook`, `project-save`, `sequence-flatten`*

---
