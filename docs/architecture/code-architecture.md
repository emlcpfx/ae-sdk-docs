# Code Architecture

> 1 Q&A · source: AE plugin dev community Discord

### What are the advantages of writing host and plugin format-independent code across multiple applications?

Writing format-independent code for multiple hosts (AE, Premiere Pro, OFX, Nuke, Frei0r, etc.) frees up significant resources by allowing you to write the processing code once and only occasionally update the wrapper projects, rather than maintaining separate implementations for each host.

*Tags: `code-architecture`, `cross-platform`, `deployment`, `premiere`*

---
