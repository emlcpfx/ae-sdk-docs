# 16Bpc

> 1 Q&A · source: AE plugin dev community Discord

### What is the optimal way to make an AE effect plugin compatible with all bit depths (8, 16, and 32 bpc)?

The simplest approach is to write wrapper/converter functions to convert to 32bpc and back, then write your processing code once in 32bpc. These converter functions can also use the multi-threaded iteration suites. The alternative is branching with switch statements for each bit depth, but that results in three separate pieces of code that only differ in type names.

*Tags: `16bpc`, `32bpc`, `8bpc`, `bit-depth`, `conversion`, `pixel-format`*

---
