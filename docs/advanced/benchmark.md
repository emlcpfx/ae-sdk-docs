# Benchmark

> 1 Q&A · source: AE plugin dev community Discord

### How much faster is GPU rendering compared to CPU path for plugin operations?

According to benchmarks, there is a significant difference: GPU processing can complete operations in milliseconds while the CPU path takes several seconds. However, performance depends on the actual operations being offloaded—if critical work like UV unwrapping still falls back to CPU, the GPU advantage is negated.

*Tags: `benchmark`, `cpu`, `gpu`, `optimization`, `performance`*

---
