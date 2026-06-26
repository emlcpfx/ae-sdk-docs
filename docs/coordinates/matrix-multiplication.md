# Matrix Multiplication

> 1 Q&A · source: AE plugin dev community Discord

### How can I chain multiple transform_world calls, using the output of one as input for the next?

Never use the same buffer as both input and output of transform_world, as it reads and overwrites simultaneously, causing corrupted output. Never overwrite the input buffer (AE caches it). For repeated transforms: (1) Create a temp buffer, (2) transform input to temp, (3) transform temp to output, (4) transform output to temp, (5) repeat as needed, (6) copy to output if last result is in temp. However, matrix multiplication is extremely cheap (20-something operations for 3x3), while each transform_world render costs millions of operations per image. Always prefer multiplying matrices and doing a single render.

*Tags: `buffer-management`, `matrix-multiplication`, `performance`, `rendering`, `transform-world`*

---
