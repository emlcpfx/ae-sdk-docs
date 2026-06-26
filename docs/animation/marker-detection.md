# Marker Detection

> 1 Q&A · source: AE plugin dev community Discord

### How can I trigger an event or function when the playhead crosses a marker in After Effects?

There is no direct marker event in the After Effects SDK. Instead, you can use an idle hook to monitor the composition's current time and detect marker crossings by comparing the previous time with the current time. Alternatively, you can listen to time change events and perform the same check. You may also investigate AEGP_RegisterCommandHook to see if a command is triggered on marker events.

*Tags: `aegp`, `marker-detection`, `reference`, `sdk`*

---
