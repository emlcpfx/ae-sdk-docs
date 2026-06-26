# Ae Versions

> 1 Q&A · source: AE plugin dev community Discord

### Why can't I draw a custom UI in PF_Window_LAYER even though it works in PF_Window_COMP?

This was a bug in After Effects CS3 and CS4 where custom UI drawing in layer windows was not supported. The plug-in would receive PF_Event_DRAW calls and return a buffer, but After Effects would ignore the rendered output. This issue was fixed in CS5. As a workaround, you may need to upgrade to CS5 or later, or reference how other plugins like 'meshwarp' or 'vector paint' handle this limitation.

*Tags: `ae-versions`, `custom-ui`, `debugging`, `ui`*

---
