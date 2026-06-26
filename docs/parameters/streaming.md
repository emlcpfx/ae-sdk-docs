# Streaming

> 2 Q&As · source: AE plugin dev community Discord

### How can a C++ effect plugin modify transform parameters of other layers in a project?

Use the StreamSuite to affect any parameter (which are internally streams) in the project. The 'project dumper' sample project demonstrates how to access streams. However, since CC2015, you cannot change the project during a render call. To modify project parameters, use UI events (anything other than render and pre-render calls) when you can modify the project as needed.

*Tags: `aegp`, `params`, `render-loop`, `sdk`, `streaming`*

---

### Is there an official sample project demonstrating how to access and manipulate streams in the After Effects SDK?

Yes, Adobe provides the 'project dumper' sample project as part of the After Effects SDK. This sample demonstrates how to access streams and is useful for understanding how to affect parameters in a project using the StreamSuite.

*Tags: `aegp`, `open-source`, `reference`, `sdk`, `streaming`*

---
