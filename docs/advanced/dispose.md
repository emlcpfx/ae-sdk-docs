# Dispose

> 1 Q&A · source: AE plugin dev community Discord

### Is 5000+ arb dispose calls on AE shutdown normal?

Yes, this is expected behavior. AE's allocators generally don't dispose until available memory is saturated, thread destruction, or app exit. You can test this by setting AE's available memory artificially low (like 2GB) - you'll see many more destruction calls during normal operation. On exit, AE cleans up everything at once.

*Tags: `arb-data`, `dispose`, `memory-management`, `shutdown`*

---
