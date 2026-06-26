# Render Format

> 1 Q&A · source: AE plugin dev community Discord

### Can you encode data into dead pixels and retrieve it using sampleImage?

Yes, you can encode and render data into dead pixels and use a sampleImage expression to retrieve those pixels and set text. This works with text layers and Null objects. However, you must encode each color separately and limit it to one byte per color to avoid byte order or bit depth issues. A critical caveat is that after AE's threading changes, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering yet.

*Tags: `expressions`, `pixel-encoding`, `render-format`, `sampleImage`, `threading`*

---
