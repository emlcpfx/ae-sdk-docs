# Color Accuracy

> 1 Q&A · source: AE plugin dev community Discord

### Why does DrawBot draw colors at ~90% brightness compared to what you specify?

This can be caused by using DrawImage with opacity less than 1.0, which makes the drawing darker while alpha still appears as 1.0 (since AE's UI is behind with solid alpha). It could also be related to AE's UI brightness preferences or the project's color space settings, as AE's drawing suites may compensate for those things.

*Tags: `brightness`, `color-accuracy`, `custom-ui`, `drawbot`*

---
