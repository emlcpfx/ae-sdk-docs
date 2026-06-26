# Path Vertices

> 1 Q&A · source: AE plugin dev community Discord

### Can the AE SDK provide post-processed shape path vertices (after Offset Path, Zig Zag, Wiggle modifiers)?

AE doesn't provide post-processed paths. You can only read the raw parameter streams and then try to emulate the processed path yourself. There is no way to get the final computed path after shape modifiers like Offset, Zig Zag, or Wiggle have been applied.

*Tags: `aegp`, `limitation`, `path-vertices`, `shape-layer`, `shape-modifiers`*

---
