# Death Hook

> 2 Q&As · source: AE plugin dev community Discord

### What suites should you avoid calling from Death Hook?

You shouldn't rely on any suite being available in the Death Hook. Attempting to use suites like UtilitySuite's ReportInfo from Death Hook can cause unhandled exceptions. By that point in the shutdown process, many suites may already be torn down.

*Tags: `aegp`, `death-hook`, `shutdown`, `suites`*

---

### Are there certain AEGP suites that should not be called from Death Hook, and does using Utility Suite to ReportInfo cause exceptions?

The user reported unhandled exceptions when attempting to call Utility Suite's ReportInfo function from Death Hook. This suggests that certain suites have restrictions on being called from Death Hook contexts, likely due to the cleanup phase not supporting all operations. The answer was not explicitly provided in the conversation, but the question indicates a known limitation.

*Tags: `aegp`, `death-hook`, `debugging`*

---
