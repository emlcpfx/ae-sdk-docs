# Max Sleep

> 1 Q&A · source: AE plugin dev community Discord

### How can I make the idle_hook get called more frequently for near-real-time communication?

Use AEGP_CauseIdleRoutinesToBeCalled() to trigger more frequent idle calls. Setting *max_sleepPL directly doesn't reliably work as it gets reset to its default value. Note that on macOS the idle hook caps at around 30 calls per second, while on Windows it can reach 60.

*Tags: `cause-idle`, `frame-rate`, `idle-hook`, `max-sleep`, `real-time`*

---
