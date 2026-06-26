# Disposal

> 1 Q&A · source: AE plugin dev community Discord

### How can you test whether memory disposal is working correctly in an After Effects plugin?

One practical approach is stress-testing by rendering a long video (e.g., 5 hours) to identify memory leaks. If the plugin runs without crashing and memory usage remains stable throughout the render, it indicates that necessary resources have been properly disposed. This method relies on observation rather than formal memory profiling tools.

*Tags: `debugging`, `disposal`, `memory`, `testing`*

---
