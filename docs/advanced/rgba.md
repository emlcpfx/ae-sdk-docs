# Rgba

> 1 Q&A · source: AE plugin dev community Discord

### What are the byte order and format considerations when encoding data into pixels for sampleImage retrieval?

sampleImage always returns RGBA format regardless of the internal render format or CPU vs GPU byte order differences (ARGB vs RGBA vs BGRA in CUDA). To avoid byte order issues, encode each color channel separately and limit it to one byte per color. After AE's threading changes, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering the frame, so timing is critical.

*Tags: `byte-order`, `pixel-encoding`, `rgba`, `sampleImage`, `threading`*

---
