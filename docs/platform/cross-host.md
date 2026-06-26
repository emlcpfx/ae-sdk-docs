# Cross Host

> 1 Q&A · source: AE plugin dev community Discord

### Can the same .aex plugin work in both After Effects and Premiere Pro?

Yes, but it requires correct coding. Premiere does many things differently than AE and not all features are supported in both hosts. The AE SDK documentation has a dedicated section for Premiere Pro compatibility (https://ae-plugins.docsforadobe.dev/). Notably, Premiere does not support SmartRender, which is a significant difference. There are also long-standing bugs in Premiere's implementation of AE plugin APIs (present since at least 2006) that are regularly reported to but largely ignored by Adobe.

*Tags: `aex`, `compatibility`, `cross-host`, `premiere-pro`, `smartrender`*

---
