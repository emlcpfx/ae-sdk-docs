# Path Suite

> 1 Q&A · source: AE plugin dev community Discord

### How does the AddArc function work in the Drawbot Path Suite, and why do two consecutive arcs produce unexpected shapes?

When drawing multiple arcs/circles in the same path, you need to call close() on the path after each circle. Without closing, the entire sequence is treated as one continuous path, and the renderer draws a connecting line from the end point of one arc back to the beginning of the next, creating unexpected shapes. You can have multiple closed shapes in one path, or alternatively draw each shape in separate paths with separate drawing calls.

*Tags: `addarc`, `custom-ui`, `drawbot`, `drawing`, `path-suite`*

---
