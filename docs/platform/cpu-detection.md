# Cpu Detection

> 1 Q&A · source: AE plugin dev community Discord

### Were in_data->what_cpu and what_fpu fields meaningfully used in plugin development?

These fields were likely used to steer code toward or away from certain instruction streams based on CPU/FPU-specific performance characteristics, to detect software-emulated FPUs, and to handle the 68882 coprocessor differently during multi-tasking scenarios.

*Tags: `aegp`, `cpu-detection`, `legacy`, `performance`*

---
