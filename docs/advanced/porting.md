# Porting

> 1 Q&A · source: AE plugin dev community Discord

### How should I port a plug-in from CS4 to CS6 when PF_Context's cgrafptr and PF_GET_CGRAF_DATA macro are removed?

The drawing mechanism changed between CS4 and CS5. Direct drawing into a context is no longer supported. Instead, you must use the DrawBot suite to draw. The quickest conversion approach is to wrap the DrawBot commands under your old drawing command names. This change also makes the code cross-platform compatible.

*Tags: `cross-platform`, `mfr`, `porting`, `ui`, `windows`*

---
